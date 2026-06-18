# Datamosh Tool — Research & Materials

> Documento de planejamento. Coleta o que é reaproveitável do **Procedural Loop Lab**
> e a pesquisa sobre datamoshing **real** (sem simulação/overlay).
> Status: planejamento. Nada implementado ainda.

---

## 1. O que é datamoshing (de verdade)

Datamoshing = corromper deliberadamente a **compressão temporal** de um vídeo.
Vídeo comprimido tem dois tipos de frame relevantes:

- **I-frame (keyframe / IDR)** — imagem completa, independente. "Reinicia" a cena.
- **P-frame (delta)** — guarda só **vetores de movimento + diferença** em relação ao frame anterior.

O efeito nasce quando o decoder aplica vetores de movimento de um P-frame sobre a
imagem **errada** (porque o I-frame que deveria reiniciar a cena foi removido ou
porque P-frames foram repetidos). Não é filtro por cima — é o próprio decodificador
errando de propósito.

### Dois efeitos canônicos

| Efeito | Como se faz | Resultado visual |
|---|---|---|
| **I-frame removal ("melt")** | Remover o I-frame num corte de cena | Vetores da cena B aplicados nos pixels da cena A → cenas "derretem" uma na outra |
| **P-frame duplication ("bloom")** | Duplicar um P-frame de movimento forte 20–50× | O mesmo movimento se reaplica → pixels esticam, fazem rastro, pulsam |

Diferença prática: I-frame removal atua **na transição entre clipes**;
P-frame duplication atua **dentro de um único clipe** (cria ritmo/rastro).

---

## 2. Abordagem clássica (desktop) — referência de parâmetros

A pipeline tradicional usa **MPEG-4 ASP (Xvid) em container AVI** + Avidemux:

```bash
# Encode "moshável": GOP enorme (poucos keyframes), sem B-frames
ffmpeg -i input.mp4 -c:v mpeg4 -vtag xvid -qscale:v 4 -g 9999 -bf 0 -an output.avi
```

- `-g 9999` → praticamente um único keyframe (mais P-frames = mais mosh possível)
- `-bf 0` → sem B-frames (estrutura simples, previsível)
- `-qscale:v 2..6` → qualidade (menor = melhor)
- container **AVI** (mais fácil de editar byte-a-byte que MP4)

Inspecionar tipos de frame:
```bash
ffprobe -show_frames -select_streams v input.avi | grep pict_type
```

Avidemux: Video Output = **Copy** (sem recompressão), navega até o frame, deleta
(I-frame removal) ou duplica (P-frame duplication), salva.

> **Por que isso importa pra nós:** define os parâmetros-alvo de encode
> (GOP longo, sem B-frames, single-reference) que precisamos reproduzir no browser
> para que o stream fique moshável de forma controlada.

---

## 3. Abordagem browser-native (o que vamos usar): WebCodecs + Mediabunny

Não dá pra rodar Avidemux no browser, mas dá pra fazer **o mesmo** com WebCodecs +
Mediabunny — manipulando os **chunks codificados** diretamente. É datamosh real:
mexemos no bitstream, não na imagem.

### Fluxo
```
[entrada] → DEMUX → [pacotes codificados: key/delta] → MANIPULAR → REMUX → [arquivo .mp4]
                                                          ↘ DECODE → canvas (preview ao vivo)
```

- **Demux** (vídeo enviado pelo usuário): Mediabunny `Input` + `EncodedPacketSink`
  → itera `EncodedPacket` (`.data` Uint8Array, `.type` `'key'|'delta'`, `.timestamp`).
- **Manipular o stream** (o coração do datamosh):
  - *I-frame removal*: descartar pacotes `key` (exceto o 1º) → deltas seguintes
    aplicam em referência errada.
  - *P-frame duplication*: repetir pacotes `delta` N vezes.
  - *bonus*: reordenar/embaralhar dentro de janela, dropar % de frames, corromper bytes de delta.
- **Remux**: `EncodedVideoPacketSource('avc')` + `output.addVideoTrack` →
  `source.add(EncodedPacket.fromEncodedChunk(chunk))` → `Mp4OutputFormat` → Blob.
  ⚠️ Pacotes devem ser adicionados **em ordem de decode**; timestamps precisam ser regerados.
- **Preview ao vivo**: `VideoDecoder` decodifica o stream JÁ manipulado e desenha no canvas.
  O glitch aparece porque as referências estão quebradas de propósito.

### Detalhe crítico para conteúdo PROCEDURAL/gerado
Para moshar conteúdo que a ferramenta gera (não um upload), nós mesmos **encodamos**
com GOP controlado: **1 keyframe no início, todo o resto delta** (`keyFrame:false`).
Isso é 100% controlável via `CanvasSource`/`VideoEncoder` — o Loop Lab já faz exatamente
esse controle de `keyFrame` por frame no export MP4.

### Para vídeo enviado pelo usuário
Uploads têm keyframes onde nós não controlamos. Melhor: **decodar → re-encodar H.264
com GOP longo (1 key, resto delta) → então aplicar o mosh no stream → remux**.
Garante que fica moshável de forma previsível.

### Riscos/abertos a validar
- WebCodecs `VideoEncoder` (H.264) expõe poucos botões; precisa confirmar controle de
  B-frames e reference frames. Baseline profile (`avc1.42001f`) tende a não emitir B-frames.
- Alguns muxers/players rejeitam stream que começa com delta ou com pacote marcado
  errado. Truque conhecido: manter 1 key inicial e marcar o resto como delta.
- H.264 tem deblocking/concealment que pode "limpar" parte do glitch vs MPEG-4 ASP.
  Mitigar com profile baseline + single reference. (Opcional futuro: caminho MPEG-4/AVI
  via ffmpeg.wasm para mosh mais "sujo" estilo clássico.)

---

## 4. O que é REAPROVEITÁVEL do Procedural Loop Lab

Arquivo: `../procedural-loop-lab/index.html` (single-file, ~12.9k linhas, estética
terminal verde-no-preto). Peças diretamente reutilizáveis:

| Peça | Onde (linha aprox.) | Reuso para datamosh |
|---|---|---|
| **Export MP4 WebCodecs + Mediabunny** | `exportMP4Mediabunny` ~12618 | Base do remux; já controla `keyFrame` por frame |
| **Muxer WebM EBML/Matroska próprio** | ~12441 | Alternativa de container/muxing |
| **Export WebM (WebCodecs offline + MediaRecorder fallback)** | ~12517 | Padrão de fallback entre browsers |
| **Encoder GIF próprio (paleta 3-3-2 + LZW)** | ~12324 | Export GIF do resultado moshado |
| **PNG sequence + ZIP builder** | `buildZip` / mp4 pack ~12698 | Export de frames + pack ffmpeg |
| **`renderFrame(lp, frame, canvas)` determinístico** | ~9799 | Fonte de frames procedurais para encodar/moshar |
| **Sistema de camadas (`project.stack`, blend, opacity, clip)** | ~9819 | UI/estrutura de pilha de efeitos |
| **Entrada de VÍDEO** (`VP`, `ensureVideoAsset`, `syncVideoAssetToTime`, seek exato) | ~1136–1330 | Upload + amostragem de vídeo do usuário |
| **Entrada de IMAGEM** (`IP`, `ensureImageAsset`) | ~1113–1135 | Upload de foto |
| **EFFECTS pixel-level via `getImageData`** | `EFFECTS` ~2419 | Pixel-sort real opera nos mesmos bytes |
| **RNG com seed (mulberry32/hashInt)** | ~553 | Determinismo/reprodutibilidade (seed) |
| **Sistema de parâmetros tipados + motion** | `P/CP/SP/...` ~582 | Painel de controles com keyframes/motion |
| **Export session UI (progresso/ETA/cancel)** | `withExportSession` | Encode/mosh são pesados; reusar UX |
| **Toda a UI terminal (CSS, topbar, painéis, dropdowns)** | `<style>` 7–544 | Identidade visual consistente entre as ferramentas |
| **Presets/share via JSON + seed** | `project` ~572 | Salvar/compartilhar receitas de mosh |

**Resumo:** ~70% da casca (UI, export, determinismo, I/O de mídia, muxing) já existe.
O que é novo é o **núcleo de manipulação de bitstream** (seção 3) e os controles
específicos de datamosh.

---

## 5. Esboço de arquitetura da nova ferramenta

```
Entrada (foto OU vídeo, ambos suportados)
  ├─ foto  → vira "clipe" de N frames (ou par de imagens A/B p/ transição mosh)
  └─ vídeo → decode → re-encode H.264 GOP-longo
        │
        ▼
  PILHA DE OPERAÇÕES DE MOSH (reusa modelo project.stack)
    • I-frame removal (alvo: cortes / todos / primeiro)
    • P-frame duplication (frame alvo + Nx repetições)
    • frame reorder / shuffle em janela
    • frame drop por %
    • corromper bytes de delta (intensidade)
    • pixel-sort real (sobre frames decodificados, via getImageData)
        │
        ▼
  PREVIEW ao vivo (VideoDecoder → canvas)
        │
        ▼
  EXPORT (reusa pipeline): MP4 / WebM / GIF / PNG-seq
```

Stack: mesma do Loop Lab — **single-file HTML + WebCodecs + Mediabunny (ESM via CDN)**,
deploy estático no Vercel. Sem servidor (o Artsen/H.264-Datamosh-Web-Tool usa
Python/Flask/ffmpeg no backend — nós evitamos isso ficando 100% no browser).

---

## 6. Fontes

- Datamoshing — visão geral: http://datamoshing.com/
- Como datamoshar (FFmpeg/Avidemux): http://datamoshing.com/2016/06/26/how-to-datamosh-videos/
- Glitchology — workflow + comandos: https://glitchology.com/datamoshing/
- O que é datamoshing: https://spotlightfx.com/blog/what-is-datamoshing
- WebCodecs Fundamentals — muxing/demuxing: https://webcodecsfundamentals.org/basics/muxing/
- WebCodecs Handbook (freeCodeCamp): https://www.freecodecamp.org/news/the-webcodecs-handbook-native-video-processing-in-the-browser/
- Mediabunny — quick start / media sources / reading: https://mediabunny.dev/guide/quick-start
- MDN WebCodecs API: https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API
- Ref. de tool web (backend Python, NÃO o nosso caminho): https://github.com/Artsen/H.264-Datamosh-Web-Tool
```
