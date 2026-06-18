# Datamosh Tool — Plano de Implementação (afiado, time-boxed)

> Entrega: **sábado**. Build efetiva: **sexta** (1 dia). Você offline até sexta.
> Princípio-mestre: **BUILD LIMPO, feito para datamoshing.** NÃO é fork do Loop Lab, NÃO é
> "clonar e enxugar". Construímos um `index.html` NOVO e **portamos seletivamente** só os
> mecanismos relevantes a datamosh (timing de loop, plumbing de render/export, I/O de mídia,
> compositing de camadas + blend modes, seed). **Geradores e a biblioteca de efeitos artísticos
> do Loop Lab NÃO entram — zero código morto.** As únicas ops de pixel são as que SÃO glitch
> (pixel-sort, RGB shift, corrupção) = código novo. O núcleo de bitstream é novo.

---

## 0. A grande sacada de arquitetura

Tudo vira um **"produtor de frames"** alimentando UM pipeline:

```
                         ┌─ vídeo enviado  (decode-by-seek → canvas)  ┐
PRODUTOR DE FRAMES  ─────┼─ imagem enviada (segura N frames / par A→B) ┼─→ ENCODER (WebCodecs, GOP controlado)
                         └─ (compositing de camadas: blend/opacity)    ┘        │
                                                                                ▼
                                                      chunks[] {type:key|delta, data} ──→ MANIPULAR (o datamosh)
                                                                                ├─→ MUX  (Mediabunny → .mp4)   [EXPORT]
                                                                                └─→ DECODE (VideoDecoder→canvas) [PREVIEW]
```

As FONTES são a mídia do usuário (vídeo/imagem) — **não há geradores procedurais**. O loop de
frames determinístico (o mesmo mecanismo do `renderExportFrames`, reimplementado limpo, com sink
plugável `onFrame`) empurra cada canvas composto no `VideoEncoder` que nós controlamos.

---

## 1. O que PORTAR do Loop Lab (reimplementar limpo no arquivo novo)

Usar `../procedural-loop-lab/index.html` como **referência de implementação** (consultar e
adaptar funções pontuais), NÃO como base a editar. Portar SÓ estes mecanismos:

| Mecanismo | Ref. (linha) | Por quê é relevante a datamosh |
|---|---|---|
| Timing de loop (`duration`/`fps` → `timelineFrameCount`/`loopProgressForFrame`) | 9873+ | Define nº de frames do stream a moshar |
| Loop determinístico de frames (a essência do `renderExportFrames({onFrame})`) | 12258 | Produtor de frames p/ o encoder |
| Sessão de export (progresso/ETA/cancel — essência do `withExportSession`) | 12217 | Encode+mosh é pesado; boa UX |
| Compositing de camadas (essência do `renderFrame`: blend/opacity/enabled/clip) | 9799 | Empilhar fontes (ex: par A→B p/ melt) |
| Lista de blend modes (`BLENDS`/`BLEND_LABELS`) | 9678 | Compositing de fontes |
| I/O de vídeo (`ensureVideoAsset`/`seekVideoAsset`/`waitVideoReady`/`targetVideoTime`) | 1136–1330 | Decode-by-seek amostra frames do upload |
| I/O de imagem (`ensureImageAsset`) | 1113 | Upload de foto como fonte |
| Mux MP4 Mediabunny (scaffolding Output/BufferTarget/Mp4OutputFormat) | 12618 | Base do MUX (trocar fonte por packets) |
| GIF encoder (LZW) + PNG-seq + `downloadBlob`/`buildZip` | 12324/12162/12002 | Export do resultado |
| Utils: RNG seed (`mulberry32`/`hashInt`), `clamp`/`lerp`/`wrapMod`, `setStatus`/`toast` | 553+ | Determinismo + helpers |
| CSS/shell terminal (opcional — só se quisermos esse visual) | 7–544 | Identidade; reescrever enxuto |

**NÃO portar (ficam fora — código morto):** todos os `GENERATORS` (aurora, neon, partículas,
shapes, texto…), toda a biblioteca de `EFFECTS` artísticos (ascii, halftone, riso, etc.),
sistema de motion/keyframes por-parâmetro, fontes/SVG/textos, presets visuais.

**Ops de pixel que ENTRAM (código NOVO, datamosh-específico, não porte):** pixel-sort (ASDF),
RGB channel shift, channel displacement, corrupção de bytes.

---

## 2. O núcleo NOVO (a única parte de risco) — `mosh.js` lógico

### 2a. Encoder com GOP controlado (produz chunks moshaveis)
```
const chunks = [];                          // {type, timestamp, data:Uint8Array, dup, drop}
const enc = new VideoEncoder({
  output:(chunk)=>{ const b=new Uint8Array(chunk.byteLength); chunk.copyTo(b);
                    chunks.push({type:chunk.type, timestamp:chunk.timestamp, data:b}); },
  error:e=>console.error(e)
});
enc.configure({ codec:"avc1.42001f", width:W, height:H, bitrate,
                framerate:fps, latencyMode:"realtime" });   // realtime → sem B-frames/reorder
// por frame:
enc.encode(new VideoFrame(canvas, {timestamp:i*1e6/fps}),
           { keyFrame: (i===0 || cutPoints.has(i)) });        // 1 key no início (+ cortes)
// ...depois: await enc.flush();
```
`latencyMode:"realtime"` + baseline profile = sem B-frames, single-reference → mosh limpo e previsível.

### 2b. Manipular `chunks[]` (O DATAMOSH de verdade)
- **I-frame removal ("melt")**: remover chunks `type==='key'` nos cortes (manter o key do frame 0).
- **P-frame duplication ("bloom")**: para chunk delta alvo, `splice` inserindo N cópias.
- **frame drop / reorder em janela / corromper bytes de delta** (intensidade) → variações.
- Reaplicar timestamps em ordem de decode: `ts = i * 1e6 / fps`.

### 2c. MUX (reusa scaffolding do `exportMP4Mediabunny`)
```
const { Output, Mp4OutputFormat, BufferTarget, EncodedVideoPacketSource, EncodedPacket } = mb;
const src = new EncodedVideoPacketSource("avc");
output.addVideoTrack(src); await output.start();
for(const c of chunks) await src.add(new EncodedPacket(c.data, c.type, c.timestamp/1e6, 1/fps));
await output.finalize(); // → Blob mp4
```
⚠️ Adicionar **em ordem de decode**; primeiro packet deve ser `key`.

### 2d. PREVIEW (decode do stream JÁ manipulado)
```
const dec = new VideoDecoder({ output:f=>{ ctx.drawImage(f,0,0,W,H); f.close(); }, error:()=>{} });
dec.configure({ codec:"avc1.42001f", codedWidth:W, codedHeight:H });
for(const c of chunks) dec.decode(new EncodedVideoChunk({type:c.type, timestamp:c.timestamp, data:c.data}));
```
O decoder "errando" nas referências quebradas **é** o efeito. (Precisa do `description`/avcC do encoder;
guardar o `metadata.decoderConfig.description` do primeiro `output`.)

---

## 3. Entradas (só mídia do usuário — sem geradores)

- **Vídeo enviado** (caminho principal): `seekVideoAsset` (decode-by-seek) → `drawImage` cada frame
  no canvas → encoder. Re-encoda com NOSSO GOP → garantidamente moshável.
- **Imagem**: foto única → pixel-sort/displacement real (ImageData) + corrupção de bytes;
  par A→B → clipe curto p/ melt (transição "derretendo" entre duas fotos).
- **Padrão de teste (dev-only, NÃO é feature):** um único gerador trivial (ex: barras/ruído em
  movimento) só para validar o núcleo no Bloco 1 sem depender de upload. Não vai pro produto.

---

## 4. Cronograma de SEXTA (de-risca o núcleo primeiro)

| Bloco | Tempo | Tarefa |
|---|---|---|
| **0** | 30min | Criar `dataMoshingTool/index.html` LIMPO (shell mínimo). Portar do Loop Lab só os utils base (timing, seed, helpers, `downloadBlob`). Sem generators/effects |
| **1 ⚠️** | 2–3h | **PRIMEIRO**: protótipo isolado do núcleo (2a→2d), usando o padrão de teste dev-only. Encode 1 key → derrubar/duplicar/corromper → **decode e ver o smear**. Valida o risco antes de tudo |
| **2** | 1–2h | Plugar produtores reais: vídeo enviado (decode-by-seek) + imagem no encoder |
| **3** | 1–2h | UI: pilha de camadas (compositing+blend, portado) + Painel Mosh: melt / bloom ×N / corrupt / drop% / seed |
| **4** | 1h | Export: stream manipulado → MP4 (reusa mux) + GIF/PNG-seq do output decodificado |
| **5** | 1h | Pixel-sort real (ImageData) + entrada de imagem |
| **6** | 1h | Polish, presets JSON+seed, deploy Vercel |

**Sábado**: buffer, testes em navegador-alvo, entrega.

---

## 5. Riscos a validar no BLOCO 1 (antes de investir o resto)

1. `VideoEncoder` baseline emite GOP de 1 key sem B-frames? (esperado com `latencyMode:"realtime"`).
2. Mediabunny `EncodedVideoPacketSource` aceita packets duplicados/re-timestampados e stream começando em key?
3. `VideoDecoder` precisa do `decoderConfig.description` (avcC) do encoder — capturar no 1º `output.metadata`.
4. Se H.264 "limpar" demais o glitch (deblocking/concealment): plano B = caminho MPEG-4 ASP/AVI via
   `ffmpeg.wasm` (mosh clássico mais sujo). Só se sobrar tempo — H.264 deve bastar.

**Regra de ouro:** se o Bloco 1 funcionar (decode mostrar smear real), o resto é montagem
em volta do núcleo, com mecanismos portados pontualmente.

---

## 6. Definição de "entregue" (sábado)

- [ ] Upload de vídeo → datamosh real (I-frame removal + P-dup) → preview ao vivo → download MP4.
- [ ] Upload de imagem → pixel-sort/corrupção real → export.
- [ ] Controles: tipo de efeito, intensidade/Nx, pontos de corte, seed reprodutível.
- [ ] Deploy no Vercel (estático, single-file).
- Bônus se sobrar: empilhar 2 fontes c/ blend, reorder/drop, GIF/PNG-seq, presets compartilháveis.
```
