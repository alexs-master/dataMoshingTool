# 🗺️ MAPA DO MVP — Datamosh Tool

> O que entra no MVP (entrega sábado), quais mecanismos portamos do Loop Lab (reescritos limpo,
> NÃO fork), e o modelo de dados. Princípio: **build feito para datamoshing, zero código morto;
> composição = mecanismos portados; bitstream = a parte nova.**
> Ver também `00-MASTER.md` (visão geral), `PLAN.md` (cronograma) e `BIBLE-/REFS-` (fontes).

---

## 0. A arquitetura em DUAS ETAPAS (a chave de tudo)

A confusão a evitar: nem todo "efeito" é igual. Há dois lugares onde a coisa acontece:

```
┌── ETAPA 1: COMPOSIÇÃO (por frame) ── mecanismos PORTADOS (reescritos limpo, NÃO fork) ─────────────┐
│  PILHA DE CAMADAS                                                                                   │
│   • fontes: vídeo / imagem do usuário   → produzem pixels   (SEM geradores procedurais)            │
│   • blend mode + opacity + enabled por camada                                                      │
│   • efeitos de GLITCH PIXEL-LEVEL (código NOVO): pixel-sort, RGB shift, channel displace (ImageData)│
│   → loop de frames determinístico  → 1 canvas composto por frame                                   │
└────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                   │  produtor de frames (sink onFrame, mecanismo portado)
                                   ▼
┌──────────────────── ETAPA 2: BITSTREAM (pós-encode) ── A PARTE NOVA ───────────────────────────────┐
│  ENCODER WebCodecs (GOP controlado: 1 key + cortes)  → chunks[] {type:key|delta, data}             │
│  MOSH no stream:  (A) drop key=melt · dup delta=bloom · drop% · reorder                            │
│                   (B) corromper bytes de chunks delta  (= hex-edit do mdat da bíblia)              │
│     ├─→ MUX (Mediabunny → .mp4)   [EXPORT]                                                          │
│     └─→ DECODE (VideoDecoder → canvas)  [PREVIEW do resultado moshado]                              │
└────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

⚠️ **Não é fork do Loop Lab.** "Mecanismo portado" = reimplementado limpo no arquivo novo, usando o
Loop Lab só como referência. Geradores e efeitos artísticos do Loop Lab **não entram**.
- **Etapa 1** = composição das FONTES do usuário (vídeo/imagem) + glitch pixel-level (código novo).
- **Etapa 2** = datamosh "de verdade" (decoder errando). É o núcleo novo (PLAN §2).
- Glitch pixel-level (pixel-sort/RGB shift) é da Etapa 1; datamosh de codec é da Etapa 2. Não misturar.

---

## 1. ESCOPO DO MVP

### ✅ Dentro (entrega sábado)
- **Entradas:** upload de **vídeo** e de **imagem** (a mídia do usuário — **sem geradores procedurais**).
- **Composição:** ✅ pilha de camadas com **blend modes + opacity + on/off + reordenar + clip mask**
  (implementado em 2026-06-19/20, Bloco 3 completo).
- **Loop:** controles de **duração (s) + fps** → define frames do loop E o tamanho do stream (portado).
- **Glitch pixel-level (Etapa 1):** ✅ **pixel-sort (ASDF/Kim Asendorf)** + **RGB channel shift** como
  camadas (não globais).
- **Datamosh de bitstream (Etapa 2):** ✅ **I-frame removal (melt)** + **P-frame duplication (bloom)**
  + **corrupção de bytes de delta** — todos como camadas.
- **Controles do mosh:** intensidade / Nx repetições / pontos de corte (múltiplos) / % drop / **seed**
  reprodutível — params por camada.
- **Preview ao vivo** do resultado (decode do stream manipulado).
- **Dois modos de reprodução** (adicionado em 2026-06-20): tempo real (sem bitstream, zero encode) e
  decode progressivo (com bitstream, começa a tocar após 1º frame).
- **Export:** **MP4** (núcleo) — funciona nos dois modos. GIF/PNG-seq: PNG ✅, GIF pendente.
- **Deploy Vercel**, single-file. ← pendente.

### ❌ Fora do MVP (depois, se houver tempo)
- Feedback/echo como família completa (3ª família Menkman) — opcional.
- Caminho MPEG-4 ASP/AVI via ffmpeg.wasm (plano C).
- Motion/keyframes por-parâmetro do Loop Lab (mantemos params simples).
- Glitch de JPG por hex, WordPad, audio-bending (nicho).
- Presets compartilháveis avançados (um punhado de presets fixos basta).

---

## 2. MODELO DE DADOS (estende o do Loop Lab)

Implementado em 2026-06-19/20 — estado atual do `index.html`:

```js
project = {
  width, height,            // PORTAR — dimensões de saída
  duration, fps,            // PORTAR — "comprimento do loop" → timelineFrameCount = round(duration*fps)
  seed,                     // PORTAR — determinismo (mulberry32/hashInt)
  stack: []                 // PORTAR+EXPANDIDO — pilha de camadas (índice 0 = topo do painel = último efeito)
};

// Item de camada — todos os tipos compartilham base comum:
layer = {
  id, type, kind,           // type: video|image|pixelsort|rgbshift|melt|bloom|corrupt
                            // kind:  src|pixelfx|bitstream (derivado do type)
  enabled: true,            // eye toggle
  blend: "source-over",     // 13 modos (Normal/Add/Screen/Multiply/Overlay/Difference/Exclusion/Hard-Light/Soft-Light/Dodge/Burn/Darken/Lighten)
  opacity: 1,               // 0..1 (pixel-fx: vira "força" do efeito)
  clip: false,              // máscara de recorte (destination-in no composto abaixo)
  name: "Vídeo",            // editável inline
  _open: true,              // card minimizado/expandido (UI state)
  // params específicos por type:
  //   video:      { file, src, regionOffset, stretch:false, scale:1, posX:0, posY:0 }
  //               (src = { input, track, duration, width, height, getFrameCanvas })
  //   image:      { file, bitmap, stretch:false, scale:1, posX:0, posY:0 }
  //   melt:       { cutFrames: "auto"|"30,60" }
  //   bloom:      { targetFrame, count }
  //   corrupt:    { count, intensity }               (intensity 0..100, stride seguro 15..30)
  //   pixelsort:  { dir:"h"|"v", criterion:"luma"|"hue"|"sat", min, max }
  //   rgbshift:   { angle, amount }
};

// Transform (aplicado a src layers na composição):
//   stretch=true  → drawImage(src, 0,0,w,h) — estica para preencher o canvas (ignora scale/pos)
//   stretch=false → desenha com dims NATURAIS, escala por `scale`, posiciona por posX/posY (centro=0,0)
//                   fórmula: dx = (canvasW - srcW*scale)/2 + posX; dy = (canvasH - srcH*scale)/2 + posY
//   Default: stretch=false, scale=1, posX=0, posY=0 — preserva a forma original centrada
```

**Convenção da pilha (bottom-up):** `renderFrame` e `applyBitstreamLayers` iteram do
último índice (base do painel) ao índice 0 (topo do painel). Cada camada afeta o que está
**ABAIXO** dela no painel (Photoshop-like): fontes na base desenham primeiro, efeitos acima
transformam o resultado acumulado.

Tipos de fonte (kind:"src"): `video`, `image`. **Sem `procedural` — não há geradores.**
Tipos de efeito pixel-level (kind:"pixelfx"): `pixelsort`, `rgbshift`.
Tipos de efeito bitstream (kind:"bitstream"): `melt`, `bloom`, `corrupt`.

---

## 3. MAPA DE PORTE — peça por peça (as que você citou)

> "PORTAR" = reimplementar limpo no arquivo novo, usando o Loop Lab só como referência (linhas).
> **NÃO** é copiar o arquivo. Geradores e efeitos artísticos do Loop Lab **não entram** (ver fim da tabela).

| Peça do MVP | Status | Ref. no Loop Lab | Notas |
|---|---|---|---|
| **Comprimento dos loops** (duração+fps→frames) | ✅ PORTAR | `project.duration/fps`, `timelineFrameCount`, `loopProgressForFrame`, `timelineFrameAtElapsed` (9873+) | Mesmo valor define a timeline E o nº de frames do stream a moshar |
| **Pipeline de render** | ✅ PORTAR+REESCRITO | `renderFrame` (9799) + `renderExportFrames({onFrame})` (12258) + `withExportSession` (12217) | `onFrame` é o ponto de entrada da Etapa 2 (empurra canvas no encoder). Reescrito bottom-up p/ sistema de camadas |
| **Upload de vídeo** | ✅ PORTAR | `ensureVideoAsset`/`seekVideoAsset`/`waitVideoReady`/`videoTrimBounds`/`targetVideoTime` (1136–1330) | Trocado por Mediabunny demux (mais confiável que `<video>`) |
| **Upload de imagem** | ✅ PORTAR | `ensureImageAsset`/`IP` (1113) | Foto vira fonte de camada (ImageBitmap) |
| **Sistema de camadas** | ✅ PORTAR+EXPANDIDO | `project.stack`, `makeGen`/`makeEffect` (9681), cards + drag-reorder + enable + clip (11000+) | Implementado em 2026-06-19/20. Drag-reorder via HTML5 DnD (só handle dispara), eye toggle, blend dropdown, opacity slider, clip mask, params por tipo |
| **Blend modes** | ✅ PORTAR+EXPANDIDO | `BLENDS`/`BLEND_LABELS` (9678) aplicados em `renderFrame` via `globalCompositeOperation` | 13 modos: Normal/Add/Screen/Multiply/Overlay/Difference/Exclusion/Hard-Light/Soft-Light/Dodge/Burn/Darken/Lighten |
| **Opacity / on-off / clip por camada** | ✅ PORTAR | `it.opacity`/`it.enabled`/`it.clip` no `renderFrame` | Clip usa `destination-in` estilo Photoshop |
| **Determinismo (seed)** | ✅ PORTAR | `mulberry32`/`hashInt`/`withItemSeed` (553) | Mosh reprodutível |
| **Export MP4** | ✅ PORTAR (MP4 adaptado) | `exportMP4Mediabunny` (12618) | Trocar `CanvasSource` por `EncodedVideoPacketSource` alimentado pelos chunks moshados. Funciona nos dois modos (tempo real e bitstream) |
| **Export PNG** | ✅ PORTAR | PNG-seq/ZIP (12162) | Composição estática → PNG direto do canvas |
| **GIF** | ❌ Pendente | GIF (12324) | Não implementado |
| **UI shell (topbar, painéis, export progress)** | ✅ PORTAR | `<style>` 7–544 + markup | Identidade visual consistente; layout 3 painéis (add/composição, stage, pilha de camadas) |
| **Pixel-sort (Etapa 1)** | ✅ NOVO | só o padrão `getImageData` como referência (2419+) | Algoritmo ASDF (luma/hue/sat + threshold) — código novo, implementado como camada |
| **RGB channel shift (Etapa 1)** | ✅ NOVO | idem `getImageData` | offset por canal, implementado como camada |
| **Núcleo de bitstream (Etapa 2)** | ✅ NOVO | — (PLAN §2) | encoder→capturar chunks→manipular→mux/decode. Cada efeito (melt/bloom/corrupt) é uma camada |
| **Dois modos de reprodução** | ✅ NOVO | — | Tempo real (sem bitstream) + decode progressivo (com bitstream). Adicionado 2026-06-20 |
| ~~GENERATORS (aurora, neon, partículas, shapes, texto…)~~ | ❌ FORA | — | Não é uma tool de geração; código morto |
| ~~EFFECTS artísticos (ascii, halftone, riso…)~~ | ❌ FORA | — | Só ops de pixel que SÃO glitch entram, e como código novo |
| ~~Motion/keyframes por-param · fontes/SVG/texto · presets visuais~~ | ❌ FORA | — | Irrelevante a datamosh |

**Resumo:** o que você citou (loops, render, uploads, camadas, blends) é **portado** (reescrito limpo,
não herdado). Geradores e efeitos artísticos ficam de fora (zero código morto). O genuinamente novo
é a Etapa 2 (datamosh de codec) + as ops de pixel que são glitch (pixel-sort, RGB shift).

---

## 4. UI DO MVP (layout limpo, próprio da tool)

```
┌ TOPBAR: logo · [Duração] [FPS ▾] · [W×H ▾] · seed · ⟳ random · ▶/⏸ · [MP4][GIF][PNG] ┐
├──────────────┬──────────────────────────────────────┬───────────────────────────────────┤
│ ADICIONAR    │            PREVIEW (canvas)           │  PILHA DE CAMADAS                 │
│ (catálogo)   │  mostra o resultado MOSHADO ao vivo   │  ┌ camada: [👁][nome] blend▾ opac│
│ • Fontes:    │  (decode do stream manipulado)        │  │  ⋮ params da camada            │
│   Vídeo↑     │                                       │  └ …reordenável (drag)           │
│   Imagem↑    │  [barra de progresso/ETA no export]   │                                   │
│ • Glitch px: │                                       │  PAINEL MOSH (Etapa 2):           │
│   Pixel-sort │                                       │   técnica: ☑melt ☑bloom ☑corrupt │
│   RGB shift  │                                       │   cortes / Nx / drop% / intensid. │
│   Ch. displ. │                                       │   seed                            │
└──────────────┴──────────────────────────────────────┴───────────────────────────────────┘
```

Opcional (alinha c/ Menkman): agrupar o catálogo em **3 famílias** — *Compressão* (datamosh codec),
*Feedback* (futuro), *Corrupção* (byte/pixel glitch).

---

## 5. FLUXO DO USUÁRIO (caminho feliz do MVP)

1. Sobe um **vídeo** (ou imagem, ou empilha os dois com blend) → vira camada(s).
2. Ajusta **duração/fps** (comprimento do loop) e ordem/blend/opacity das camadas.
3. (Opcional Etapa 1) adiciona **pixel-sort / RGB shift** como camada de efeito.
4. No **Painel Mosh** escolhe técnica (melt/bloom/corrupt), intensidade e **seed**.
5. **Preview ao vivo** mostra o resultado já moshado (decode do stream manipulado).
6. **Exporta MP4** (ou GIF/PNG-seq). Deploy já no Vercel.

---

## 6. ORDEM DE CONSTRUÇÃO → ver `PLAN.md`

Bloco 0 (arquivo limpo + utils base) → **Bloco 1 (núcleo Etapa 2, de-risco)** → Bloco 2 (plugar produtores) →
Bloco 3 (UI camadas+mosh) → Bloco 4 (export) → Bloco 5 (pixel-sort+imagem) → Bloco 6 (polish+deploy).
A Etapa 1 (camadas/blends/render/uploads) são mecanismos portados; o esforço real concentra na Etapa 2.
