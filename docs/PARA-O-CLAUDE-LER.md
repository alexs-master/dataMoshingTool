# PARA O CLAUDE LER — RESUMO DA SESSÃO NO OUTRO ASSISTENTE (2026-06-19/20)

> **ATENÇÃO CLAUDE:** este arquivo é um resumo de tudo que foi feito em uma sessão de iteração
> com outro assistente (Gemini/GLM). O usuário pediu para eu criar este resumo antes de migrar
> para você. **Leia este arquivo primeiro** antes de tocar no projeto — contém o estado atual,
> bugs conhecidos não resolvidos, decisões travadas, e armadilhas que já caímos.
>
> O usuário está frustrado porque vários bugs de preview/playback surgiram nas últimas iterações
> e não foram resolvidos corretamente. O principal problema em aberto é: **decode falha quando
> há efeitos bitstream ativos**. Veja seção "BUGS EM ABERTO" abaixo.

---

## CONTEXTO DO PROJETO

**Datamosh Tool** — ferramenta web single-file (`index.html`, ~1571 linhas) para datamoshing
**real** de vídeo e foto, manipulando o bitstream H.264 de verdade (não é filtro CSS/overlay).
100% client-side, usando **WebCodecs + Mediabunny** (ESM via CDN), deploy estático Vercel.

Filosofia: respaldada pela teoria da Rosa Menkman (em `docs/REFS-menkman-glitch-momentum.md`)
— glitch autêntico = o artefato do encode/decode, não decoração.

---

## ESTADO ATUAL DO CÓDIGO

### Arquitetura
- **Sistema de camadas completo (7 tipos)** numa pilha unificada:
  - `src`: `video` (Mediabunny demux), `image` (ImageBitmap)
  - `pixelfx`: `pixelsort` (ASDF), `rgbshift`
  - `bitstream`: `melt` (I-frame removal), `bloom` (P-frame duplication), `corrupt` (XOR esparso)
- **Pipeline bottom-up** (Photoshop-like): `stack[0]` = topo do painel = último efeito aplicado.
  Cada camada afeta o que está **ABAIXO** dela no painel. `renderFrame` e `applyBitstreamLayers`
  iteram do último índice ao primeiro.
- **Nova camada sempre no topo** (`project.stack.unshift(layer)`, NÃO `push`).
- **APLICAR e REPRODUZIR são responsabilidades SEPARADAS:**
  - `applyAll()` — SÓ processa (encode + mosh + decode) → armazena frames em `currentDecodedFrames`
    → desenha 1º frame. **NÃO toca.**
  - `startPlayback()` / `startDecodedPlayback()` / `startRealtimePlayback()` — controlam reprodução.
  - `playPauseBtn.onclick` alterna entre `stopPlayback()` e `startPlayback()`.
- **AUTO-APPLY COMPLETAMENTE DESATIVADO.** Mudar qualquer parâmetro NÃO dispara `applyAll`.
  Só atualiza o indicador vermelho/verde. Usuário deve clicar **Aplicar** manualmente.
- **Dois modos de reprodução:**
  - Mode 1 (sem bitstream): composição em tempo real (zero encode, double-buffering)
  - Mode 2 (com bitstream): decode COMPLETO antes de tocar (espera todos os frames)
- **Sistema de playback:** `currentPlaybackPlayer` com flag `cancelled` (NÃO usar tokens —
  race conditions freqüentes; foi o que quebrou tudo antes).
- **Indicador vermelho/verde** em camadas bitstream:
  - 🟢 Verde = `isBitstreamEncodeFresh()` true = encode atualizado
  - 🔴 Vermelho pulsante = encode pendente
- **Transform por camada (src layers):** `stretch`, `scale`, `posX`, `posY`.
  - `stretch=true` → estica para preencher canvas (ignora scale/pos)
  - `stretch=false` (default) → preserva proporção, escala por `scale`, posiciona por `posX`/`posY`
    (centro = 0,0). Fórmula: `dx = (canvasW - srcW*scale)/2 + posX`
- **Cards minimizáveis:** botão `▾/▸` em todos os cards (não só vídeo/imagem).
- **Drag-reorder:** só o handle `⋮⋮` é `draggable=true` (não o card inteiro — senão arrastar
  slider vira drag).

### Técnicas bitstream (validadas empiricamente)
- **Melt** (I-frame removal): funciona puro. `applyMelt(chunks, cutFrameIndices)` remove chunks
  `type==='key'` nos cortes (mantém key do frame 0). Múltiplos cortes suportados (`parseCutFrames`
  aceita "30,60" ou "auto").
- **Bloom** (P-frame duplication): `applyBloom(chunks, targetIndex, count, fps)` duplica o chunk
  delta mais próximo de `targetIndex`, `count` vezes, e retimestampa. **EXIGE**
  `hardwareAcceleration:'prefer-hardware'` no decoder — software aborta por validação estrita de
  `frame_num` no slice header do H.264. **Importante:** o chunk duplicado é clonado
  (`data: dupChunk.data.slice()`) para evitar bug latente de referência compartilhada.
- **Corrupt** (XOR esparso): `applyCorrupt(chunks, count, intensity01, rng)` faz XOR esparso em
  N chunks delta. Stride seguro ~15-20 bytes (intensity 0..1 → stride 30..15). Mais denso aborta
  decode. `intensity=0` retorna cedo (não aplica XOR). Pula os 6 primeiros bytes do NAL
  (header+início do slice header).

### Encoder config (validado)
```js
codec: "avc1.42001f" (H.264 baseline)
latencyMode: "realtime"  // evita B-frames, mosh limpo
avc: { format: "avc" }
bitrate: Math.min(20_000_000, Math.max(1_500_000, Math.round(W*H*fps*0.12)))
keyFrame: true só no frame 0 (+ cortes de melt, na UNIÃO de todas as melt layers)
```

### Decoder config (validado)
```js
codec: "avc1.42001f"
description: decoderDescription  // capturado do 1º output.metadata.decoderConfig.description do encoder
hardwareAcceleration: "prefer-hardware"  // OBRIGATÓRIO para bloom/corrupt sobreviverem
```

---

## ⚠️ BUGS EM ABERTO (NÃO RESOLVIDOS)

### Bug principal: DECODE FALHA COM EFEITOS BITSTREAM

**Sintoma:** ao clicar Aplicar com qualquer efeito bitstream ativo (melt, bloom, ou corrupt),
o decode falha. Às vezes mostra "Decode não produziu frames", às vezes "reduza intensidade".

**O que já tentei (e falhou):**

1. **Controle de fluxo com `dec.decodeQueueSize > 8`** — CAUSOU DEADLOCK. O decoder entrava
   num estado onde `decodeQueueSize` não diminuía e ficávamos presos no while loop até o
   safety timeout de 30s. **Removido.**

2. **Yield periódico simples** (`if (i % 16 === 15) await new Promise(r => setTimeout(r, 0))`) —
   não resolveu o problema de fundo.

3. **Sistema de tokens** (`previewLoopToken`) — frágil, causava race conditions onde loops
   concorrentes se matavam. **Substituído** por `currentPlaybackPlayer` com flag `cancelled`.

4. **Decode progressivo** (começar a tocar após 1º frame, decodificar resto em background) —
   causava "buracos" no playback. **Removido** — agora espera TODOS os frames.

5. **Tentativas de separar apply de playback** (já feito) — `applyAll` só processa e armazena
   em `currentDecodedFrames`; `startPlayback` só toca. A separação está correta arquiteturalmente
   mas o decode em si ainda falha.

**Hipóteses não testadas:**
- Talvez o problema seja a forma como `applyBloom` retimestampa os chunks duplicados. A fórmula
  atual é `timestamp: Math.round(i * 1e6 / fps)` após o splice. Pode estar quebrando a continuidade
  esperada pelo decoder.
- Talvez o problema seja que `applyBloom` insere N referências ao mesmo chunk clonado — mas o
  clone é só shallow (`data.slice()`), talvez precise clonar mais campos.
- Talvez `hardwareAcceleration: 'prefer-hardware'` não esteja disponível no ambiente do usuário
  (GPU fraca, drivers pobres). Precisa de feature-detect com `VideoDecoder.isConfigSupported()`.
- Talvez o decoder de hardware tenha um limite de chunks duplicados que tolera, e bloom com count
  alto (20+) estoura esse limite.
- Talvez o problema seja o `flush()` — em alguns casos o decoder precisa de `await` entre decode
  e flush, ou precisa de um chunk "vazio" final.

**O que o usuário disse que funcionava antes:** "antes era bem mais rápido mesmo com 3 efeitos
bitstream ativados". Ou seja, em algum momento da sessão o decode funcionou com múltiplos
efeitos. Provavelmente uma das refatorações quebrou algo sutil. **Diff contra o commit anterior
ao sistema de camadas pode revelar a regressão.**

### Possível bug secundário: performance do encode
Quando adicionei suporte a transform, criei canvas temporário redundante em `renderFrame`.
Já foi removido, mas vale conferir se não há outras alocações desnecessárias no hot path.

---

## DECISÕES TRAVADAS (NÃO REVERTER SEM FALAR COM O USUÁRIO)

1. **Datamosh real via WebCodecs + Mediabunny** — não simulação, não filtro CSS.
2. **Build limpo, NÃO fork do Loop Lab** — portar seletivamente mecanismos, zero código morto.
3. **Sem geradores procedurais** — só mídia do usuário.
4. **Single-file HTML** — `index.html` auto-contido, deploy estático Vercel.
5. **Painel de camadas inicia VAZIO** — usuário constrói do zero (não template vídeo+melt).
6. **Convenção bottom-up** — cada camada afeta o que está ABAIXO no painel (Photoshop-like).
7. **Nova camada sempre no topo** (`unshift`, não `push`).
8. **AUTO-APPLY DESATIVADO** — usuário sempre clica Aplicar manualmente.
9. **APLICAR ≠ REPRODUZIR** — responsabilidades separadas.
10. **"Ajustar à próxima fonte"** é checkbox persistente (não botão one-shot).
11. **Label do clip** é só "Máscara de recorte" (sem texto explicativo extra).
12. **Decoder SEMPRE com `hardwareAcceleration: 'prefer-hardware'`** — validado empiricamente
    que software aborta em bloom/corrupt.

---

## ARMADILHAS JÁ CAÍDAS (NÃO REPITA)

1. **Não use `card.draggable = true` no card inteiro** — arrastar slider vira drag do card.
   Só o handle `.lc-handle` é draggable.

2. **Não use `canvas.width = w; canvas.height = h;` sem checar se mudou** — atribuir width/height
   (mesmo ao mesmo valor) LIMPA o canvas para preto. Causa flash. Use:
   ```js
   if (canvas.width !== w || canvas.height !== h){ canvas.width = w; canvas.height = h; }
   ```

3. **Não desenhe direto no canvas visível durante `await`** — clearRect + await getFrameCanvas
   causa flicker. Use **double-buffering**: renderize num `OffscreenCanvas` (back buffer) e só
   copie para o canvas visível quando o frame estiver completo.

4. **Não se esqueça do `clearRect` ANTES do `drawImage(back, 0, 0)`** — senão frames anteriores
   aparecem por baixo nas áreas transparentes do back buffer (rastro/trail quando scale<1 ou
   stretch=false deixa bordas transparentes).

5. **Não use `project.stack.push(layer)` para adicionar no topo** — push adiciona no final do
   array = base do painel. Use `unshift` para índice 0 = topo do painel.

6. **Não misture apply com playback** — `applyAll` só processa; `startPlayback`/`stopPlayback`
   controlam reprodução. Sefor juntar, volta a race condition.

7. **Não use tokens (`previewLoopToken`)** para controle de loop — frágil. Use referência direta
   ao player (`currentPlaybackPlayer`) com flag `cancelled`.

8. **Não use `while (dec.decodeQueueSize > N) await ...`** — causa deadlock. O decoder pode
   entrar em estado onde a queue não esvazia e fica preso até timeout.

9. **Não chame `scheduleAutoApply()` em handlers de params** — o usuário foi CATEGÓRICO: nada
   de auto-apply. Use `refreshEncDots()` para só atualizar o indicador, ou `redrawSourcePreview()`
   para sliders de transform (que só redesenham o 1º frame da fonte, sem processar).

10. **Handlers de transform (stretch/scale/posX/posY)** devem chamar `redrawSourcePreview()` —
    que só redesenha o 1º frame da fonte (sem processar) se não há `currentDecodedFrames` e não
    está tocando. Assim o usuário vê o efeito imediatamente sem disparar applyAll.

11. **`drawSourceWithTransform`** deve ser usada em TODOS os pontos onde desenha uma fonte
    (handleFile vídeo, handleFile imagem, renderFrame, drawSourceFirstFrame). Não use
    `drawImage(src, 0,0,w,h)` direto — isso sempre estica ignorando stretch/scale/pos.

12. **Re-renderizar o painel inteiro (`renderLayerStack()`) destrói inputs** — sliders perdem
    foco, valores em edição somem. Para atualizar só o indicador, use `refreshEncDots()` que
    só mexe nos `.lc-enc-dot`.

13. **`applyBloom` deve clonar o `data`** do chunk duplicado: `{ ...dupChunk, data: dupChunk.data.slice() }`.
    Sem isso, se uma manipulação futura mutar `c.data` in-place, todas as duplicatas viram juntas.

14. **`applyCorrupt` com `intensity=0`** deve retornar cedo (não aplicar XOR). Antes aplicava
    com stride=30 e xorVal=0x40 mesmo em intensity=0.

15. **`applyMelt` aceita múltiplos cortes** — `parseCutFrames` retorna array. `allCutPoints`
    coleta a UNIÃO dos cortes de todas as melt layers. Encoder põe keyframes nessa união.

16. **Não crie OffscreenCanvas temporário redundante** em `renderFrame` — `drawSourceWithTransform`
    pode receber direto o `frameCanvas` (vídeo) ou `layer.bitmap` (imagem). Canvas temporário
    intermediário dobra alocação e mata performance.

17. **Sempre `chame `Complete`** ao final de cada iteração de código — senão o gateway de IM
    fecha o painel de preview no chat do usuário.

---

## ESTRUTURA DOS ARQUIVOS

```
dataMoshingTool/
├── index.html                          ← o app (1571 linhas)
└── docs/
    ├── 00-MASTER.md                    ← fonte de verdade / snapshot de estado
    ├── PROCESSO.md                     ← história narrativa completa (Fases 0-15 + decisões D1-D17)
    ├── DEVLOG.md                       ← log curto por etapa (entradas mais recentes no topo)
    ├── PLAN.md                         ← plano time-boxed + pseudocódigo do núcleo
    ├── MVP-MAP.md                      ← arquitetura 2 etapas, escopo, modelo de dados, UI
    ├── RESEARCH.md                     ← pesquisa de datamoshing + reuso + fontes
    ├── BIBLE-datamoshing-com.md        ← bíblia técnica (datamoshing.com) minerada
    ├── REFS-menkman-glitch-momentum.md ← teoria (Rosa Menkman) — NORTE conceitual
    ├── refs_menkman_glitch-momentum.pdf← PDF original (não commitar, .gitignore)
    ├── RESUMO-PARA-CLIENTE.md          ← resumo técnico de escopo para a cliente
    ├── RESUMO-PARA-CLIENTE.pdf         ← PDF gerado via Edge headless
    └── PARA-O-CLAUDE-LER.md            ← ESTE ARQUIVO
```

---

## FLUXO DO USUÁRIO ESPERADO (quando tudo funciona)

1. Adiciona fonte (vídeo/imagem) → 1º frame aparece no canvas (estático, respeitando transform)
2. Ajusta parâmetros (qualquer um) → indicador atualiza, sliders de transform redesenham preview
3. Clica **Aplicar** → processa (encode+mosh+decode OU só composição) → 1º frame do resultado
   aparece no canvas (estático). NÃO toca.
4. Clica **▶** → reproduz em loop
5. Clica **⏸** → pausa (último frame fica no canvas)
6. Ajusta parâmetros de novo → nada muda (preview pausado mostra último frame)
7. Clica **Aplicar** de novo → re-processa → novo 1º frame aparece (estático)
8. Clica **▶** → reproduz o novo resultado

---

## ONDE COMEÇAR A DEBUGAR O BUG DO DECODE

1. **Adicione logging detalhado em `decodeChunks`** — imprima cada chunk (type, timestamp,
   byteLength) e cada callback (onFrame, onError, onDone). Veja exatamente onde para.

2. **Teste com melt SOZINHO primeiro** — melt é o mais simples (só remove key chunks).
   Se melt falha, o problema é no decoder config ou no encoder. Se melt funciona, teste bloom
   sozinho, depois corrupt sozinho, depois combinações.

3. **Verifique se `decoderDescription` está sendo capturado corretamente** — é capturado no
   1º callback de `output` do encoder. Se por algum motivo vier null, o decoder não configura.

4. **Tente SEM `hardwareAcceleration: 'prefer-hardware'`** para ver se o erro muda. Se mudar
   para "frame_num validation failed", confirma que é o problema conhecido de bloom/corrupt
   em software decoder. Se der erro diferente, é outra coisa.

5. **Feature-detect GPU** com `VideoDecoder.isConfigSupported({codec, codedWidth, codedHeight,
   hardwareAcceleration: 'prefer-hardware'})` antes de configurar o decoder. Se retornar false,
   avise o usuário que bloom/corrupt não vão funcionar.

6. **Considere não usar `Mediabunny EncodedVideoPacketSource` para preview** — em vez disso,
   decodifique chunks diretamente com `VideoDecoder.decode(new EncodedVideoChunk(...))`. É o que
   `decodeChunks` já faz, mas pode haver detalhe.

7. **Compare com o commit antes do sistema de camadas** — o usuário disse que "antes funcionava
   com 3 efeitos bitstream". O git está em `https://github.com/alexs-master/dataMoshingTool`.
   Branche `feat/datamosh-core-mvp` tem o código antigo (pré-camadas). Diff pode revelar regressão.

---

## CONTATOS E REFERÊNCIAS

- **Repositório:** https://github.com/alexs-master/dataMoshingTool
- **Loop Lab (referência, NÃO base a editar):** `../procedural-loop-lab/index.html`
- **Mediabunny docs:** https://mediabunny.dev/guide/quick-start
- **WebCodecs muxing:** https://webcodecsfundamentals.org/basics/muxing/
- **Datamosh técnica:** https://glitchology.com/datamoshing/
- **ASDF Pixel Sort:** GitHub `kimasendorf/ASDFPixelSort`

---

## RESUMO EXECUTIVO PARA O CLAUDE

O usuário está frustrado porque o decode de vídeo com efeitos bitstream parou de funcionar
depois de várias refatorações. O código está bem documentado (DEVLOG.md tem entradas
detalhadas de cada mudança, PROCESSO.md tem a narrativa completa, este arquivo tem o resumo).

**Comece lendo este arquivo, depois o `00-MASTER.md`, depois o `DEVLOG.md` (topo para baixo)
para entender o estado atual.**

O bug principal é decode falhando com bitstream ativo. Hipóteses não testadas estão listadas
acima. O usuário disse que antes funcionava — provavelmente uma das refatorações quebrou algo
sutil. Diff contra `feat/datamosh-core-mvp` branch pode revelar.

Respeite as **DECISÕES TRAVADAS** e evite as **ARMADILHAS JÁ CAÍDAS** — o usuário já passou
por muita frustração com regressões.
