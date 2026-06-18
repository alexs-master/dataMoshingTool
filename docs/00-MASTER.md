# 🧭 DATAMOSH TOOL — Documento Mestre (LEIA ISTO PRIMEIRO)

> **Propósito deste arquivo:** fonte de verdade única do projeto. Se o contexto da
> sessão for perdido/compactado, **ler este arquivo do início ao fim re-orienta tudo.**
> Mantê-lo atualizado. Detalhes profundos estão em `RESEARCH.md` e `PLAN.md` (mesma pasta).

---

## ⏱️ STATUS ATUAL  ·  PRÓXIMA AÇÃO

- **Fase:** PLANEJAMENTO concluído. Nada de código ainda.
- **Prazo de entrega:** SÁBADO 2026-06-20.
- **Janela de build:** SEXTA 2026-06-19 (usuário offline quinta à noite → sexta).
- **➡️ PRÓXIMA AÇÃO ao retomar:** Bloco 0 (criar `index.html` LIMPO e portar só os utils base —
  NÃO clonar o Loop Lab) → Bloco 1 (protótipo isolado do núcleo de mosh e **ver o smear real**).
- **Log de progresso:** ver `DEVLOG.md` (atualizar a cada etapa).

---

## 🎯 O QUE É

Ferramenta web de **datamoshing REAL** para **fotos E vídeos**.

**Requisito inquebrável do usuário:** tudo procedural e datamosh **real** — manipular o
**bitstream codificado** de verdade. NADA de simulação, filtro CSS, ou overlay por cima do vídeo.
O usuário não conhece os internos de datamoshing → autenticidade/correção é responsabilidade minha.
**Respaldo teórico** (Rosa Menkman, enviado pela cliente): glitch real = o artefato do encode/decode;
glitch-como-filtro vira só "moda" e perde o momentum crítico. Ver `REFS-menkman-glitch-momentum.md`.

**Companheiro do projeto existente** `procedural-loop-lab` (ferramenta de vídeo procedural do
usuário, fonte de ~70% do reuso). Há também um app de FOTOS do usuário (localização ainda não fornecida).

---

## 🧠 DATAMOSHING EM 30 SEGUNDOS

Vídeo comprimido tem **I-frames** (imagem completa, independente) e **P-frames** (só vetores de
movimento + diferença). O glitch nasce quando o decoder aplica vetores no frame ERRADO:

- **I-frame removal ("melt")** → remover keyframe num corte → cenas derretem uma na outra.
- **P-frame duplication ("bloom")** → repetir um P-frame 20–50× → pixels esticam/pulsam/rastro.

É o decodificador errando de propósito — não é efeito por cima da imagem.

---

## 🏗️ ARQUITETURA (a grande sacada)

Tudo vira um **PRODUTOR DE FRAMES** alimentando UM pipeline:

```
  vídeo enviado (decode-by-seek→canvas)  ┐
  imagem enviada (N frames / par A→B)    ┼─→ ENCODER (WebCodecs, GOP controlado)
  (compositing de camadas: blend/opacity) ┘        │
                                                  ▼
                            chunks[] {type:key|delta, data:Uint8Array}
                                                  │
                                          MANIPULAR  ← O DATAMOSH (DUAS técnicas, ver bíblia)
                       (A) drop key=melt · dup delta=bloom · drop% · reorder
                       (B) CORROMPER bytes de chunks delta (= hex-edit do mdat; garante glitch em H.264)
                                                  ├─→ MUX (Mediabunny EncodedVideoPacketSource → .mp4)  [EXPORT]
                                                  └─→ DECODE (VideoDecoder → canvas)                    [PREVIEW]
```

As FONTES são a mídia do usuário (vídeo/imagem) — **sem geradores procedurais**. Um loop de
frames determinístico (mesmo mecanismo do `renderExportFrames`, reimplementado limpo, sink `onFrame`)
empurra cada canvas composto no `VideoEncoder` que controlamos.

**Stack:** single-file HTML + WebCodecs + Mediabunny (ESM via CDN), 100% client-side, deploy
estático no Vercel. SEM servidor. **NÃO é fork do Loop Lab** (ver abaixo).

### Parâmetros-chave do encoder (evita B-frames → mosh limpo)
`codec:"avc1.42001f"` (baseline) · `latencyMode:"realtime"` · `keyFrame:true` só no frame 0 (+ cortes).

---

## ♻️ MECANISMOS A PORTAR (NÃO é fork do Loop Lab)

⚠️ **PRINCÍPIO:** build LIMPO feito para datamoshing. NÃO clonar `index.html` e enxugar.
Construir um arquivo NOVO e **portar seletivamente** (reimplementar limpo) só os mecanismos
abaixo, usando `../procedural-loop-lab/index.html` como **referência de implementação**.

| Mecanismo | Ref. (linha) | Por quê é relevante a datamosh |
|---|---|---|
| Timing de loop (`duration`/`fps`→`timelineFrameCount`/`loopProgressForFrame`) | 9873+ | Nº de frames do stream a moshar |
| Loop determinístico de frames (essência do `renderExportFrames({onFrame})`) | 12258 | Produtor de frames p/ o encoder |
| Sessão de export (progresso/ETA/cancel — essência de `withExportSession`) | 12217 | Encode+mosh é pesado |
| Compositing de camadas (essência de `renderFrame`: blend/opacity/enabled/clip) | 9799 | Empilhar fontes (par A→B p/ melt) |
| Blend modes (`BLENDS`/`BLEND_LABELS`) | 9678 | Compositing de fontes |
| I/O de vídeo (`ensureVideoAsset`/`seekVideoAsset`/`waitVideoReady`/`targetVideoTime`) | 1136–1330 | Decode-by-seek do upload |
| I/O de imagem (`ensureImageAsset`) | 1113 | Upload de foto |
| Mux MP4 Mediabunny + GIF (LZW) + PNG-seq + `downloadBlob`/`buildZip` | 12618/12324/12162 | Export do resultado |
| Utils: `mulberry32`/`hashInt`/`clamp`/`lerp`/`wrapMod`/`setStatus`/`toast` | 553+ | Determinismo + helpers |
| CSS/shell (opcional, se quisermos o visual; reescrever enxuto) | 7–544 | Identidade |

**NÃO PORTAR (código morto — ficam fora):** todos os `GENERATORS` (aurora, neon, partículas,
shapes, texto…), biblioteca de `EFFECTS` artísticos (ascii, halftone, riso…), motion/keyframes
por-parâmetro, fontes/SVG/textos, presets visuais.

**Código NOVO (datamosh-específico):** núcleo de bitstream (encoder→chunks→manipular→mux→decode)
+ ops de pixel que SÃO glitch (pixel-sort ASDF, RGB shift, corrupção). **Esta é a parte de risco.**
Pseudocódigo completo em `PLAN.md` §2.

---

## 📅 CRONOGRAMA DE SEXTA (de-risca o núcleo PRIMEIRO)

| Bloco | Tempo | Tarefa |
|---|---|---|
| 0 | 0:30 | Criar `index.html` LIMPO (shell mínimo); portar só utils base (timing/seed/helpers/`downloadBlob`) |
| **1 ⚠️** | 2–3h | **Protótipo isolado do núcleo** (padrão de teste dev-only): encode 1 key → drop/dup/corromper → **decodar e ver o smear** |
| 2 | 1–2h | Plugar produtores reais: vídeo (decode-by-seek) + imagem no encoder |
| 3 | 1–2h | UI: camadas (compositing+blend portado) + Painel Mosh (melt / bloom ×N / corrupt / drop% / seed) |
| 4 | 1h | Export: stream manipulado → MP4 (mux portado) + GIF/PNG-seq |
| 5 | 1h | Pixel-sort real (ImageData) + RGB shift + entrada de imagem |
| 6 | 1h | Polish, presets JSON+seed, deploy Vercel |

**Regra de ouro:** se o Bloco 1 mostrar smear real, o resto é montagem em volta do núcleo.

---

## ⚠️ RISCOS A VALIDAR NO BLOCO 1

1. `VideoEncoder` baseline + `latencyMode:"realtime"` emite GOP de 1 key SEM B-frames?
2. Mediabunny `EncodedVideoPacketSource` aceita packets duplicados/re-timestampados e stream começando em key?
3. `VideoDecoder` precisa do `decoderConfig.description` (avcC) → capturar no 1º `output.metadata`.
4. Se H.264 limpar demais o glitch (deblocking) na REMOÇÃO de I-frame → usar a técnica de
   **corrupção de bytes de delta** (técnica B), que a bíblia faz em H.264 e garante o visual sujo.
   MPEG-4 ASP/AVI via `ffmpeg.wasm` vira plano C (provavelmente desnecessário).

---

## ✅ DEFINIÇÃO DE "ENTREGUE" (sábado)

- [ ] Vídeo enviado → datamosh real (melt + bloom) → preview ao vivo → download MP4.
- [ ] Imagem enviada → pixel-sort/corrupção real → export.
- [ ] Controles: tipo de efeito, intensidade/Nx, pontos de corte, seed reprodutível.
- [ ] Deploy Vercel (estático, single-file).
- Bônus: empilhar 2 fontes c/ blend, reorder/drop, GIF/PNG-seq, presets compartilháveis.

---

## 🗂️ ARQUIVOS DESTE PROJETO

- `docs/00-MASTER.md` ← (este) fonte de verdade / leia primeiro
- `docs/RESUMO-PARA-CLIENTE.md` (+ `.pdf`) — resumo TÉCNICO p/ a cliente aprovar escopo/parâmetros (com seção de retorno). PDF gerado via Edge headless a partir de HTML estilizado
- `docs/MVP-MAP.md` — mapa do MVP: arquitetura 2 etapas, escopo, modelo de dados, reuso peça-por-peça, UI
- `docs/BIBLE-datamoshing-com.md` — a "bíblia" técnica da cliente (datamoshing.com) + mapa p/ a build
- `docs/REFS-menkman-glitch-momentum.md` — texto teórico (Rosa Menkman) que a cliente enviou: o NORTE conceitual (glitch real ≠ filtro)
- `docs/refs_menkman_glitch-momentum.pdf` — PDF original enviado pela cliente
- `docs/PLAN.md` — plano time-boxed com pseudocódigo do núcleo
- `docs/RESEARCH.md` — pesquisa de datamoshing + análise de reuso + fontes
- `docs/DEVLOG.md` — log de progresso (atualizar durante a build)
- `docs/PROCESSO.md` — documentação NARRATIVA completa do processo (decisões + porquês; ver política de documentação no final dele)
- (a criar na build) `index.html` — o app

## 🔗 REFERÊNCIAS RÁPIDAS

- Loop Lab (referência de implementação, NÃO base a editar): `../procedural-loop-lab/index.html` · https://procedural-loop-lab.vercel.app/
- Mediabunny: https://mediabunny.dev/guide/quick-start
- WebCodecs muxing: https://webcodecsfundamentals.org/basics/muxing/
- Datamosh técnica: https://glitchology.com/datamoshing/
- ℹ️ datamoshing.com está NO AR para a cliente (meu sandbox recebia placeholder IIS). Cópia offline minerada em `BIBLE-datamoshing-com.md`. Backup via `https://web.archive.org/web/2021id_/http://datamoshing.com/<artigo>`
- Repositório do projeto (GitHub, público): https://github.com/alexs-master/dataMoshingTool
- ASDF Pixel Sort (ref. de pixel sort da bíblia): GitHub `kimasendorf/ASDFPixelSort`
