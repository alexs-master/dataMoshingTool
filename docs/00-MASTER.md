# 🧭 DATAMOSH TOOL — Documento Mestre (LEIA ISTO PRIMEIRO)

> **Propósito deste arquivo:** fonte de verdade única do projeto. Se o contexto da
> sessão for perdido/compactado, **ler este arquivo do início ao fim re-orienta tudo.**
> Mantê-lo atualizado. Detalhes profundos estão em `RESEARCH.md` e `PLAN.md` (mesma pasta).

---

## ⏱️ STATUS ATUAL  ·  PRÓXIMA AÇÃO

- **Fase:** BUILD em andamento. O arquivo continuado na z.ai acrescentou novos efeitos bitstream e
  pixel-fx, mantendo a pilha unificada e o escopo por posição. A regressão que substituiu o decoder
  validado por fallback software + diagnóstico genérico de GPU foi removida em
  `codex/repair-zai-decoder`; o caminho rápido `prefer-hardware` foi restaurado sem apagar os efeitos.
  A composição acima do bitstream também voltou a reagir a blend/opacity durante o playback.
  A fronteira aplicada é rastreada pelo ID da camada bitstream (não por índice obsoleto), e pixel-fx
  agora usam blend/opacity reais tanto no preview quanto no export.
  Os efeitos slice-level foram reimplementados com parser avcC/SPS/PPS/RBSP/Exp-Golomb; não fazem
  mais XOR cego nem atribuem streams inválidos a “GPU fraca”.
- **Prazo de entrega:** SÁBADO 2026-06-20.
- **Janela de build:** SEXTA 2026-06-19 (estendida para a madrugada de sábado).
- **➡️ PRÓXIMA AÇÃO ao retomar:** validar com mídia real no navegador do usuário os efeitos
  blend/opacity ao vivo sobre o bitstream, os efeitos bitstream clássicos e, separadamente, os
  novos efeitos destrutivos de slice/macroblock; depois
  validar o export MP4 fora do navegador. Falhas devem expor o erro real do decoder, sem probe ou
  fallback que as atribua genericamente à GPU.
- **Log de progresso:** ver `DEVLOG.md` (atualizar a cada etapa).
- **Repositório:** https://github.com/alexs-master/dataMoshingTool — correção isolada na branch
  `codex/repair-zai-decoder`, aguardando validação com mídia real.

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

### Sistema de camadas (Etapa 1 — composição por frame)
Cada camada tem `id`, `type`, `kind` (src/pixelfx/bitstream), `enabled`, `blend`, `opacity`,
`clip`, `name`, e params específicos. A pilha (`project.stack`) é percorrida **bottom-up** na
renderização: o último índice (base do painel) é processado primeiro; o índice 0 (topo do painel)
é o último efeito aplicado. **Cada camada afeta o que está ABAIXO dela no painel** — convenção
Photoshop-like, fonte na base, efeitos em cima.

- **src** (vídeo/imagem): desenham no canvas com blend/opacity/clip
- **pixelfx** (pixel-sort/rgbshift): filtram o composto ATUAL (resultado de tudo que está abaixo
  no painel); opacity vira "força" do efeito
- **bitstream** (melt/bloom/corrupt): PULADAS aqui, atuam pós-encode (Etapa 2)

### Aplicação dos bitstream-fx (Etapa 2 — pós-encode, bottom-up)
Após o encode, as camadas bitstream são aplicadas em ordem bottom-up sobre os chunks:
melt → bloom → corrupt (ou a ordem que estiverem na pilha). Encoder põe keyframes na UNIÃO dos
cortes de todas as melt layers, garantindo que cada uma tenha o que remover.

### Dois modos de reprodução
- **Sem bitstream (tempo real):** zero encode, render direto no canvas com double-buffering (evita
  flicker). Debounce de 60ms — sliders reagem instantaneamente.
- **Com bitstream:** encode (cacheado por assinatura que inclui src+pixelfx+regiões+dimensões+fps+
  cortes), manipulação de chunks (~5ms), decode progressivo (começa a tocar após o 1º frame, o
  resto decodifica em background).

### Parâmetros-chave do encoder (evita B-frames → mosh limpo)
`codec:"avc1.42001f"` (baseline) · `latencyMode:"realtime"` · `keyFrame:true` só no frame 0 (+ cortes).

### Decoder — config crítica validada empiricamente
`hardwareAcceleration:"prefer-hardware"` é **obrigatório** para bloom e corrupt sobreviverem —
decoder de software aborta por validação estrita de `frame_num` no slice header do H.264.

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
**✅ Mostrou — Bloco 1 concluído em 2026-06-19.**

---

## ✅ BLOCO 1 — RESULTADO (validado ao vivo, 2026-06-19)

1. ✅ GOP totalmente controlado: `keyFrame:true` nos frames exatos pedidos, confirmado.
2. ✅ `description` (avcC) do encoder capturado via `meta.decoderConfig.description` — necessário p/ o decoder.
3. ✅ **Melt funciona puro**, sem config especial. Prova pixel-level: cor da cena anterior vazando
   pós-corte (distância de cor 232/441 vs. decode limpo).
4. ⚠️→✅ **Bloom e Corrupt exigem `hardwareAcceleration:'prefer-hardware'` no `VideoDecoder`** —
   achado não previsto no plano original. Sem isso, o decoder (software/no-preference) **aborta**
   por validação estrita do `frame_num` do H.264; o de hardware faz concealment e revela o glitch
   real (confirmado visualmente: objeto derrapa/desfaz, não congela).
5. 🆕 **Corrupt tem teto de densidade**: stride seguro ~15–20 bytes; mais denso aborta o decode
   mesmo em hardware → vira parâmetro de UI com piso mínimo.
6. **MPEG-4/ffmpeg.wasm (plano B) não foi necessário** — H.264 nativo do browser basta para as 3 técnicas.
7. ⏳ Ainda não testado: Mediabunny `EncodedVideoPacketSource` com packets duplicados/re-timestampados
   (testar no Bloco 4/export).

Detalhes completos do experimento: `DEVLOG.md` (entrada 2026-06-19) e `PLAN.md` §2/§5.

---

## ✅ DEFINIÇÃO DE "ENTREGUE" (sábado)

- [x] Vídeo enviado → datamosh real (melt + bloom + corrupt) → preview ao vivo → download MP4.
- [x] Imagem enviada → pixel-sort/corrupção real → export PNG (e MP4 se quiser via composição).
- [x] **Sistema de camadas completo:** fontes (vídeo/imagem) + efeitos pixel-level (pixel-sort/RGB
      shift) + efeitos bitstream (melt/bloom/corrupt) na MESMA pilha, com blend modes (13), opacity,
      máscara de recorte (clip), on/off e drag-reorder. Cada camada afeta o que está ABAIXO dela
      no painel (pipeline bottom-up).
- [x] Controles: tipo de efeito, intensidade/Nx, pontos de corte, **seed** reprodutível.
- [x] **Dois modos de reprodução:** tempo real (sem bitstream — zero encode, ~60ms debounce) e
      bitstream (encode cacheado + decode progressivo — começa a tocar assim que o 1º frame está
      pronto, decodifica o resto em background).
- [ ] Deploy Vercel (estático, single-file). ← pendente.
- [ ] Validar export MP4 real (baixar + abrir em player externo). ← pendente.
- Bônus: empilhar 2 fontes c/ blend ✓, reorder/drop ✓ (via sistema de camadas), GIF/PNG-seq
  (pendente), presets compartilháveis (pendente).

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
