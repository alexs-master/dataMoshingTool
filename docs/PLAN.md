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

## 2. O núcleo — `mosh.js` lógico ✅ VALIDADO (Bloco 1 rodado de verdade em 2026-06-19)

> Testado ao vivo no Chrome (não é só pseudocódigo). Resultados e parâmetros corretos abaixo.
> Ver `DEVLOG.md` 2026-06-19 para os números do experimento.

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

### 2b. Manipular `chunks[]` (O DATAMOSH de verdade) — parâmetros validados empiricamente

- **Melt (I-frame removal)** ✅ funciona direto, sem config especial: remover chunks `type==='key'`
  nos cortes (manter o key do frame 0). Confirmado: decode não erra, pixel pós-corte fica com a
  cor da cena anterior.
- **Bloom (P-frame duplication)** ✅ funciona, **MAS requer decoder `hardwareAcceleration:
  'prefer-hardware'`** (ver 2d). Sem isso, o decoder (software ou "no-preference", que cai em
  software) **aborta** — Chrome valida `frame_num` no slice header do H.264 e duplicatas idênticas
  violam a continuidade exigida pelo spec. O decoder de hardware faz *concealment* e tolera.
  Implementação: para chunk `delta` alvo, `splice` inserindo N cópias (ex.: 20–25), depois
  **retimestampar sequencialmente** (`ts = i * 1e6/fps` na nova ordem).
- **Corrupt (corrupção de bytes)** ✅ funciona, **MAS tem teto de densidade**: XOR esparso (a cada
  ~20 bytes, pulando os 6 primeiros bytes do NAL = header+início do slice header) sobrevive 100% e
  propaga o glitch visualmente até o próximo keyframe. Stride mais denso (a cada 7 ou 2 bytes) faz
  o decoder abortar **mesmo em hardware**. → parâmetro de intensidade na UI deve ter um piso de
  stride seguro (~15–20); abaixo disso, avisar que pode quebrar o decode.
- **frame drop / reorder em janela**: ainda não testado experimentalmente; tratar como o melt
  (provavelmente seguro) até validar.
- Reaplicar timestamps em ordem de decode sempre que a lista de chunks for reordenada/expandida.

### 2c. MUX (reusa scaffolding do `exportMP4Mediabunny`)
```
const { Output, Mp4OutputFormat, BufferTarget, EncodedVideoPacketSource, EncodedPacket } = mb;
const src = new EncodedVideoPacketSource("avc");
output.addVideoTrack(src); await output.start();
for(const c of chunks) await src.add(new EncodedPacket(c.data, c.type, c.timestamp/1e6, 1/fps));
await output.finalize(); // → Blob mp4
```
⚠️ Adicionar **em ordem de decode**; primeiro packet deve ser `key`. (Ainda não testado ao vivo —
testar no Bloco 4/export.)

### 2d. PREVIEW (decode do stream JÁ manipulado) — config CORRIGIDA pelo teste
```
const dec = new VideoDecoder({ output:f=>{ ctx.drawImage(f,0,0,W,H); f.close(); }, error:e=>{/* tratar, não é fatal */} });
dec.configure({
  codec:"avc1.42001f", codedWidth:W, codedHeight:H,
  description: decoderDescription,           // capturado do 1º output.metadata.decoderConfig.description do encoder
  hardwareAcceleration:"prefer-hardware"      // ⚠️ OBRIGATÓRIO p/ bloom e corrupt sobreviverem
});
for(const c of chunks) dec.decode(new EncodedVideoChunk({type:c.type, timestamp:c.timestamp, duration:c.duration, data:c.data}));
```
O decoder "errando" nas referências quebradas **é** o efeito — confirmado pixel-level no Bloco 1.
Tratar `error` no decoder como **esperado** com corrupção/bloom agressivos, não como bug: se
disparar, a UI deve mostrar o último frame bom e sugerir reduzir a intensidade (não travar a tool).

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

| Bloco | Tempo | Tarefa | Status |
|---|---|---|---|
| **0** | 30min | Criar `dataMoshingTool/index.html` LIMPO (shell mínimo). Portar do Loop Lab só os utils base (timing, seed, helpers, `downloadBlob`). Sem generators/effects | ✅ feito (2026-06-19) |
| **1 ⚠️** | ~~2–3h~~ ✅ feito | ~~PRIMEIRO: protótipo isolado do núcleo~~ **CONCLUÍDO 2026-06-19** — rodado de verdade no Chrome (não index.html). Melt/bloom/corrupt confirmados reais, com os parâmetros corretos (ver §2b/2d). Ver `DEVLOG.md`. |
| **2** | 1–2h | Plugar produtores reais: vídeo enviado (decode-by-seek) + imagem no encoder | ✅ feito (2026-06-19) — vídeo via Mediabunny demux (não `<video>`) |
| **3** | 1–2h | UI: pilha de camadas (compositing+blend, portado) + Painel Mosh: melt / bloom ×N / corrupt / drop% / seed | ✅ feito (2026-06-19/20) — rewrite completo com 7 tipos de camada, drag-reorder, blend/opacity/clip, params por tipo, 2 modos de reprodução |
| **4** | 1h | Export: stream manipulado → MP4 (reusa mux) + GIF/PNG-seq do output decodificado | ⏳ parcial — MP4 ✅ (funciona nos 2 modos), PNG ✅, GIF pendente; export MP4 real ainda não validado fora do ambiente de teste |
| **5** | 1h | Pixel-sort real (ImageData) + entrada de imagem | ✅ feito (2026-06-19) — pixel-sort ASDF + RGB shift como camadas |
| **6** | 1h | Polish, presets JSON+seed, deploy Vercel | ⏳ pendente |

**Sábado**: buffer, testes em navegador-alvo, entrega.

---

## 5. Riscos do BLOCO 1 — RESULTADO (validado 2026-06-19)

1. ✅ **CONFIRMADO**: `VideoEncoder` baseline + `latencyMode:"realtime"` respeita `keyFrame:true`
   nos frames exatos pedidos (GOP totalmente controlado por nós).
2. ⏳ **Ainda não testado** (será no Bloco 4/export): Mediabunny `EncodedVideoPacketSource` com
   packets duplicados/re-timestampados. Testar cedo no Bloco 4.
3. ✅ **CONFIRMADO**: `VideoDecoder` precisa do `description` (avcC) do encoder — capturado de
   `meta.decoderConfig.description` no 1º callback de `output` do encoder. Funciona.
4. ✅ **RESOLVIDO, e não como esperado**: H.264 não limpa o melt (funciona puro). Mas bloom/corrupt
   exigiam uma config não prevista no plano original: **`hardwareAcceleration:'prefer-hardware'`**
   no `VideoDecoder` (decoder de software/CPU aborta por validação estrita de `frame_num`; o de
   hardware faz concealment e revela o glitch real). **Plano B (MPEG-4/ffmpeg.wasm) não foi necessário.**
5. 🆕 **Novo risco descoberto, já mitigado**: corrupção de bytes tem teto de densidade (stride
   seguro ~15–20; mais denso aborta o decode mesmo em hardware). Vira parâmetro de UI com piso.

**Regra de ouro:** se o Bloco 1 funcionar (decode mostrar smear real), o resto é montagem
em volta do núcleo, com mecanismos portados pontualmente.

---

## 6. Definição de "entregue" (sábado) — status

- [x] Upload de vídeo → datamosh real (I-frame removal + P-dup + corrupt) → preview ao vivo → download MP4.
- [x] Upload de imagem → pixel-sort/corrupção real → export PNG (e MP4 se quiser).
- [x] Controles: tipo de efeito, intensidade/Nx, pontos de corte (múltiplos), seed reprodutível.
- [x] **Sistema de camadas completo** (Bloco 3 ✅): 7 tipos de camada, blend/opacity/clip, drag-reorder.
- [x] **Dois modos de reprodução** (adicionado 2026-06-20): tempo real (sem bitstream) + decode progressivo (com bitstream).
- [ ] Deploy no Vercel (estático, single-file). ← pendente.
- [ ] Validar export MP4 real (baixar + abrir em player externo). ← pendente.
- Bônus: empilhar 2 fontes c/ blend ✅, reorder/drop ✅ (via sistema de camadas),
  GIF (pendente), presets compartilháveis (pendente).
