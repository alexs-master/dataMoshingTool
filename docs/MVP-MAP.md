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
- **Composição:** pilha de camadas com **blend modes + opacity + on/off + reordenar** (portado).
- **Loop:** controles de **duração (s) + fps** → define frames do loop E o tamanho do stream (portado).
- **Glitch pixel-level (Etapa 1):** **pixel-sort (ASDF/Kim Asendorf)** + **RGB channel shift**.
- **Datamosh de bitstream (Etapa 2):** **I-frame removal (melt)** + **P-frame duplication (bloom)**
  + **corrupção de bytes de delta** (garante glitch em H.264).
- **Controles do mosh:** intensidade / Nx repetições / pontos de corte / % drop / **seed** reprodutível.
- **Preview ao vivo** do resultado (decode do stream manipulado).
- **Export:** **MP4** (núcleo) + GIF/PNG-seq (portado) do resultado.
- **Deploy Vercel**, single-file.

### ❌ Fora do MVP (depois, se houver tempo)
- Feedback/echo como família completa (3ª família Menkman) — opcional.
- Caminho MPEG-4 ASP/AVI via ffmpeg.wasm (plano C).
- Motion/keyframes por-parâmetro do Loop Lab (mantemos params simples).
- Glitch de JPG por hex, WordPad, audio-bending (nicho).
- Presets compartilháveis avançados (um punhado de presets fixos basta).

---

## 2. MODELO DE DADOS (estende o do Loop Lab)

```js
project = {
  width, height,            // PORTAR — dimensões de saída
  duration, fps,            // PORTAR — "comprimento do loop" → timelineFrameCount = round(duration*fps)
  seed,                     // PORTAR — determinismo (mulberry32/hashInt)
  stack: [],                // PORTAR — pilha de camadas (fontes + efeitos pixel-level)
  selectedLayerId,          // PORTAR — seleção na UI
  mosh: {                   // ★ NOVO — config da Etapa 2 (bitstream)
    technique,              //   "iremoval" | "pdup" | "corrupt" (combináveis)
    cutPoints: [],          //   frames onde forçar/remover keyframe (melt)
    dupTarget, dupCount,    //   frame-alvo e Nx (bloom)
    dropPct, corruptAmt,    //   % de frames dropados / intensidade de corrupção
    codec: "avc1.42001f", bitrate
  }
};

// Item de camada — modelo enxuto (inspirado no makeGen/makeEffect do Loop Lab, reescrito):
layer = { id, kind:"src"|"fx", type, enabled, blend:"source-over", opacity:1, params };
```

Tipos de fonte (kind:"src"): `video`, `image`. **Sem `procedural` — não há geradores.**
Tipos de efeito pixel-level (kind:"fx"): `pixelsort`, `rgbshift` (mais no futuro).

---

## 3. MAPA DE PORTE — peça por peça (as que você citou)

> "PORTAR" = reimplementar limpo no arquivo novo, usando o Loop Lab só como referência (linhas).
> **NÃO** é copiar o arquivo. Geradores e efeitos artísticos do Loop Lab **não entram** (ver fim da tabela).

| Peça do MVP | Status | Ref. no Loop Lab | Notas |
|---|---|---|---|
| **Comprimento dos loops** (duração+fps→frames) | ♻️ PORTAR | `project.duration/fps`, `timelineFrameCount`, `loopProgressForFrame`, `timelineFrameAtElapsed` (9873+) | Mesmo valor define a timeline E o nº de frames do stream a moshar |
| **Pipeline de render** | ♻️ PORTAR | `renderFrame` (9799) + `renderExportFrames({onFrame})` (12258) + `withExportSession` (12217) | `onFrame` é o ponto de entrada da Etapa 2 (empurra canvas no encoder) |
| **Upload de vídeo** | ♻️ PORTAR | `ensureVideoAsset`/`seekVideoAsset`/`waitVideoReady`/`videoTrimBounds`/`targetVideoTime` (1136–1330) | Decode-by-seek amostra frames p/ canvas; re-encodamos com nosso GOP |
| **Upload de imagem** | ♻️ PORTAR | `ensureImageAsset`/`IP` (1113) | Foto vira fonte de camada |
| **Sistema de camadas** | ♻️ PORTAR | `project.stack`, `makeGen`/`makeEffect` (9681), cards + drag-reorder + enable + clip (11000+) | Modelo enxuto; catálogo só tem fontes (vídeo/imagem) + glitch |
| **Blend modes** | ♻️ PORTAR | `BLENDS`/`BLEND_LABELS` (9678) aplicados em `renderFrame` via `globalCompositeOperation` | 10 modos: Normal/Add/Screen/Multiply/Overlay/Difference/Exclusion/Hard-Light/Soft-Light/Dodge |
| **Opacity / on-off / clip por camada** | ♻️ PORTAR | `it.opacity`/`it.enabled`/`it.clip` no `renderFrame` | Semântica idêntica |
| **Determinismo (seed)** | ♻️ PORTAR | `mulberry32`/`hashInt`/`withItemSeed` (553) | Mosh reprodutível |
| **Export MP4/GIF/PNG-seq** | ♻️ PORTAR (MP4 adaptado) | `exportMP4Mediabunny` (12618), GIF (12324), PNG-seq/ZIP | MP4: trocar `CanvasSource` por `EncodedVideoPacketSource` alimentado pelos chunks moshados |
| **UI shell (topbar, painéis, export progress)** | ♻️ PORTAR | `<style>` 7–544 + markup | Identidade visual consistente |
| **Pixel-sort (Etapa 1)** | 🆕 NOVO | só o padrão `getImageData` como referência (2419+) | Algoritmo ASDF (luma/hue/sat + threshold) — código novo, NÃO porte de efeito |
| **RGB channel shift (Etapa 1)** | 🆕 NOVO (trivial) | idem `getImageData` | offset por canal |
| **Núcleo de bitstream (Etapa 2)** | 🆕 NOVO | — (PLAN §2) | encoder→capturar chunks→manipular→mux/decode |
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
