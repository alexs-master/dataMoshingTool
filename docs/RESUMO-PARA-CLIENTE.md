# Ferramenta de Datamoshing — Resumo Técnico de Escopo

*Apresentação das características, do pipeline e dos parâmetros previstos para a ferramenta.*

---

## A ideia em uma frase

Uma ferramenta web que faz **datamoshing real** em **fotos e vídeos**, manipulando o **bitstream
codificado** (não um filtro por cima). Roda 100% no navegador, client-side, sem servidor e sem
instalação.

## Filosofia (alinhada com suas referências)

Segue o **datamoshing.com** (técnica) e a **Rosa Menkman / *The Glitch Moment(um)*** (conceito):
o glitch é o **artefato genuíno do encode/decode** — o decoder aplicando vetores de movimento
sobre a referência errada. Sem simulação, sem overlay. A imprevisibilidade do datamosh é
preservada, mas com **seed** e parâmetros para você guiar o resultado.

O framework da Menkman (interrupção do sinal em **encoding/decoding**, **feedback** e **glitch/
corrupção**) organiza as famílias de efeito da ferramenta.

---

## Pipeline técnico (como funciona por dentro)

Stack: **WebCodecs + Mediabunny (ESM via CDN)**, tudo no browser.

```
ENTRADA (vídeo/imagem)
   │
   ├─ vídeo: decode dos frames  →  RE-ENCODE com GOP controlado
   │                                (1 keyframe + cortes; resto P-frames)
   │
   ▼
ENCODER (H.264 baseline, avc1.42001f, latencyMode "realtime" → sem B-frames)
   │   → chunks[] {type: key|delta, data:Uint8Array}
   ▼
MANIPULAÇÃO DO STREAM  ← o datamosh de verdade
   (A) remover chunks 'key'  = melt (I-frame removal)
   (B) duplicar chunks 'delta' = bloom (P-frame duplication)
   (C) corromper bytes de chunks 'delta' = data corruption (equivalente ao hex-edit do mdat)
   (+ drop de frames, reordenação em janela)
   │
   ├─→ MUX (Mediabunny EncodedVideoPacketSource → MP4)        [EXPORT]
   └─→ DECODE (VideoDecoder → canvas)                          [PREVIEW ao vivo]
```

**Por que re-encodar o vídeo enviado:** uploads vêm com keyframes espalhados que não controlamos.
Re-encodando com **GOP longo (1 keyframe + cortes definidos por você)**, o stream fica
moshável de forma previsível — o equivalente browser do `-g 99999999` / "Max I-frame Interval"
do Avidemux na bíblia.

**Sobre H.264 vs MPEG-4 ASP:** a bíblia usa MPEG-4 ASP (Xvid) no método Avidemux porque decoders
H.264 às vezes "limpam" o glitch (deblocking). Mas a própria bíblia faz datamosh em **H.264 via
corrupção do átomo `mdat`** — então usamos H.264 (nativo no browser via WebCodecs) e, se a
remoção de keyframe ficar limpa demais, a **técnica (C) de corrupção de bytes** garante o visual
sujo. Caminho MPEG-4/AVI via ffmpeg.wasm fica como reserva, provavelmente desnecessário.

---

## O que a ferramenta faz

### 1. Entradas
- **Vídeo** (upload) → re-encode + datamosh no próprio stream.
- **Foto** (uma ou duas) → glitch real na imagem (pixel-level) e/ou transição "derretida" A→B
  (duas fotos viram um clipe curto e aplicamos melt).

### 2. Datamosh de vídeo (manipulação de bitstream — Etapa de codec)
| Técnica | Mecanismo | Visual |
|---|---|---|
| **Melt** | remoção de I-frame (chunk `key`) nos cortes | cenas derretem; movimento de B vaza sobre A |
| **Bloom** | duplicação de P-frame (chunk `delta`) ×N | movimento acumula, pixels esticam/pulsam/rastro |
| **Corrupção** | alteração de bytes em chunks `delta` | macroblocos quebrados, sujeira de compressão |

> Combináveis. Parâmetros de intensidade, alvo e repetição abaixo.

### 3. Glitch de imagem (pixel-level — opera no `ImageData`)
- **Pixel sort** — algoritmo **ASDF (Kim Asendorf)**: ordena trechos de pixels por
  luminosidade / matiz / saturação, delimitados por *threshold*.
- **RGB channel shift** — deslocamento por canal (franjas cromáticas).
- **Channel displace** — deslocamento espacial por mapa.

### 4. Camadas e composição
- Pilha de **camadas** (empilhar vídeo + foto, ou duas fotos A→B).
- **Blend modes** por camada via `globalCompositeOperation`: Normal, Add (lighter), Screen,
  Multiply, Overlay, Difference, Exclusion, Hard Light, Soft Light, Dodge.
- Opacidade, reordenar (drag), ligar/desligar por camada.

### 5. Preview e exportação
- **Preview ao vivo** = decode do stream **já manipulado** (o que você vê é o resultado real).
- Export: **MP4** (H.264, via mux do Mediabunny), **GIF** (encoder próprio) e **sequência PNG**.

---

## Parâmetros que você poderá controlar

| Grupo | Controles |
|---|---|
| **Loop / tempo** | Duração (s) · FPS → definem o nº de frames do stream a moshar |
| **Saída** | Resolução / proporção (1:1, 9:16, 16:9, 4:3, custom) · bitrate (qualidade) |
| **Codec / GOP** | Posição dos **cortes** (onde forçar/remover keyframe) · keyframe inicial fixo |
| **Melt** | quais cortes remover (primeiro / todos / específicos) |
| **Bloom** | frame-alvo · nº de repetições (Nx, ~20–50) |
| **Corrupção** | intensidade · alcance (quais deltas, % de bytes) |
| **Frames** | % de drop · reordenação em janela |
| **Pixel sort** | direção (H/V) · critério (luma/hue/sat) · threshold (início/fim) |
| **Cor / canais** | offset RGB · ângulo · intensidade do displace |
| **Reprodutibilidade** | **Seed** (RNG determinístico) — mesma seed = mesmo resultado; permite salvar/recuperar um visual |

Receita salvável/compartilhável como **JSON** (parâmetros + seed).

---

## Onde roda
- 100% client-side no navegador (WebCodecs). Sem upload para servidor — a mídia não sai da máquina.
- Publicável como página estática (mesmo modelo do "Procedural Loop Lab"), deploy no Vercel.
- Requer navegador com WebCodecs (Chrome/Edge/Chromium atuais; Firefox/Safari recentes).

---

## Resumo do que está incluído

Vídeo + foto · melt / bloom / corrupção · pixel sort (ASDF) · RGB shift · channel displace ·
camadas com blend · seed · preview ao vivo · export MP4/GIF/PNG · receita JSON.

---

*Resumo de escopo para alinhamento — sujeito a ajustes.*
