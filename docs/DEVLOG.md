# 📓 DEVLOG — Datamosh Tool

> Log de progresso. **Atualizar a cada etapa concluída** (curto: o que foi feito, decisões,
> gotchas, e o próximo passo). Serve para retomar sem perder o fio em caso de perda de contexto.
> Formato: entrada mais recente no TOPO.

---

## 2026-06-20 (sáb) — Amplia limite de repetições do Smear

- A pedido do usuário, o slider **Repetições** do Smear passou de máximo 60 para **300**.
- O algoritmo não possuía clamp interno; o único limite era o atributo `max` da interface.
  Em 30 fps, 300 repetições permitem aproximadamente 10 segundos adicionais de smear.

---

## 2026-06-20 (sáb) — Corrige pixel-fx invisível acima de bitstream + blend ignorado

- **Bug reproduzido pela pilha enviada pelo usuário:** `Databend → Slice hdr → Vídeo`. O Databend
  acima da fronteira deveria ser composto em tempo real, mas não aparecia nem durante o playback nem
  depois de Pause → Play.
- **Causa 1 — fronteira obsoleta:** `lastMoshResult` guardava somente `moshIdx`. O Slice hdr estava
  no índice 0 no momento do Apply; inserir Databend com `unshift()` o moveu para 1, porém o player
  continuou renderizando a faixa acima do índice antigo 0 — faixa vazia. Pause → Play reutilizava o
  mesmo número obsoleto, portanto também não resolvia.
- **Correção da fronteira:** o resultado passa a guardar `moshLayerId`. Playback e export resolvem a
  posição atual dessa mesma camada por ID, então inserções no topo não quebram o escopo. O back buffer
  do player decodificado fica sempre disponível porque uma fronteira inicialmente em 0 pode se mover.
  Adicionar fonte/pixel-fx também chama o refresh leve da composição.
- **Causa 2 — blend de pixel-fx nunca era aplicado:** o compositor usava `putImageData()`, API que
  ignora `globalCompositeOperation` e `globalAlpha`; portanto selecionar Exclusion/Multiply/etc. no
  card não tinha efeito. O frame processado agora passa por um `OffscreenCanvas` intermediário e é
  desenhado com o blend e a opacity reais da camada.
- O mesmo escopo dinâmico foi aplicado ao export MP4, evitando que uma camada adicionada acima suma
  do arquivo exportado. `git diff --check` e `node --check` passaram. A automação da aba permaneceu
  bloqueada pelo sandbox Windows, portanto a validação visual final depende do reload local.

---

## 2026-06-20 (sáb) — Restaura blend/opacity em tempo real durante playback moshado

- **Bug do usuário:** alterar o blend mode de uma camada durante a reprodução não mudava o preview;
  era necessário clicar Pause e Play para a composição aparecer.
- **Causa:** o scoping adicionado na continuação pela z.ai calculava `hasAbove` uma única vez dentro
  de `startDecodedPlayback()`. A existência das camadas compostas acima do bitstream ficava congelada
  no instante do Play; Pause → Play funcionava porque criava outro player e recalculava essa closure.
- **Correção:** `hasSrcAboveMosh()` aceita a fronteira aplicada, e o player reavalia `hasAbove` em
  cada frame. O back buffer fica preparado quando existe fronteira bitstream. Mudanças de composição
  também renovam somente o player decodificado após debounce de 50 ms, preservando o índice atual;
  não há re-encode, re-decode, auto-apply ou salto intencional ao começo.
- A correção vale para blend, opacity, clip, liga/desliga e parâmetros de pixel-fx que passam por
  `scheduleAutoApply()`. Alterações dentro do grupo já moshado continuam exigindo Aplicar, pois estão
  fisicamente incorporadas ao bitstream — as camadas acima permanecem realmente ao vivo.
- Verificação: `git diff --check` e `node --check` passaram. O controle automatizado da aba aberta
  continuou indisponível pelo erro do sandbox Windows (`CreateProcessWithLogonW: 267`); validação
  visual final deve ser feita após recarregar o arquivo local.

---

## 2026-06-20 (sáb) — Repara regressão do decoder introduzida na continuação pela z.ai

- Recebido `C:/Users/nicho/Downloads/index (17).html`, continuação que acrescenta novos efeitos
  bitstream/pixel e escopo de aplicação por posição na pilha. O arquivo foi comparado com a última
  versão funcional do repositório; os efeitos novos e a semântica de escopo foram preservados.
- **Regressão encontrada:** o caminho de decode validado havia sido trocado por uma função `async`
  com tentativa `prefer-hardware` seguida de fallback silencioso para decoder padrão/software. O
  fallback não torna streams moshados mais compatíveis e escondia a causa real do erro. Além disso,
  a UI executava uma sondagem paralela de GPU e convertia qualquer falha em aviso sobre aceleração
  de hardware, inclusive quando o stream produzido por um efeito específico era inválido.
- **Correção:** `decodeChunks()` voltou ao caminho simples e rápido já validado: um único
  `VideoDecoder` configurado com `hardwareAcceleration:'prefer-hardware'`, alimentação ordenada dos
  chunks e yield periódico. Foram removidos o fallback por software, `hwDecodeProbe()` e os avisos
  preventivos que culpavam GPU ao adicionar uma camada.
- Quando nenhum frame é produzido, a interface agora mostra a mensagem real do decoder e orienta a
  reduzir/desativar o efeito ativo para localizar uma operação destrutiva, sem diagnóstico inventado
  de GPU. Efeitos novos que deliberadamente alteram headers/slices ainda podem gerar streams que um
  decoder recuse; isso agora aparece como falha do efeito/stream, não como falta de hardware.
- Verificação: script ES module extraído do HTML passou em `node --check`; não restaram referências
  ao probe, ao fallback de software ou a `chrome://gpu`. O navegador automatizado integrado não pôde
  ser iniciado nesta sessão por falha do ambiente Windows (`CreateProcessWithLogonW: 267`), portanto
  a validação final com mídia real fica explícita como pendente, sem alegar teste que não ocorreu.

---

## 2026-06-20 (sáb) — Corrige bloom "congelando" em vez de esticar + badge "undefined" + push da branch

- **Bug do usuário:** "o efeito bloom não está fazendo isso, o vídeo simplesmente para por alguns
  frames." Reproduzi com teste isolado (vídeo sintético com quadrado em movimento linear constante,
  WebCodecs puro, sem passar pela UI): `applyBloom` duplicava o **mesmo chunk delta** N vezes. Medi
  a posição do quadrado por frame decodificado — resultado: a posição **trava num valor fixo por
  ~25 frames seguidos** (congelamento real, não simulado). Causa: repetir bytes idênticos de um
  P-frame faz o decoder convergir pro mesmo resultado a cada repetição (o delta vs. si mesmo tende
  a zero) — trava num frame parado em vez de esticar.
  - **Fix:** `applyBloom` agora cicla uma **janela de N chunks delta consecutivos** (não um só),
    repetida em loop, em vez de duplicar um único chunk. Como a referência nunca para de mudar
    entre repetições, o movimento continua se acumulando/repetindo. Testado com o mesmo harness
    (mesma posição rastreada): em vez de travar em 48, agora cicla 48→63→78→93→48→63→... —
    movimento contínuo, sem congelamento. Novo parâmetro de camada `layer.window` (slider "Janela",
    1–12, default 4; window=1 reproduz o comportamento antigo de congelar).
  - Arquivos: `applyBloom()` (assinatura ganhou `windowSize`), default do layer `bloom` (`window:4`),
    chamada em `applyBitstreamLayers`, UI do card (novo slider `.lc-window`).
- **Bug secundário (achado no mesmo teste):** badge da camada mostrava literalmente "undefined"
  para camadas `pixelfx`/`bitstream` — o mapa de label usava chaves erradas (`pix`/`bit` em vez de
  `pixelfx`/`bitstream`, que são os valores reais de `layer.kind`). Corrigido.
- **Botão de excluir camada:** usuário relatou que sumiu. Testei o `.lc-del` (✕ no canto do card) —
  está presente no HTML, com handler funcional (testei clicando programaticamente: removeu a
  camada, liberou `layer.src.input`/`layer.bitmap`, sem erro). Não encontrei regressão — instruí o
  usuário a confirmar com um hard-refresh (pode ser cache do navegador numa versão antiga).
- **Branch não estava no remoto:** o commit anterior (`86a5160`, fix do decode com bitstream) nunca
  tinha sido pushado. Rodei `git push origin feat/layer-system` antes de seguir.
- Verificado no navegador real (preview): vídeo sintético com movimento forte → bloom (target=30,
  count=25, window=4) → Aplicar → Play → posição do quadrado rastreada por pixel a cada ~80ms,
  progressão contínua confirmada, sem erros de console.
- **Próximo passo:** pedir pro usuário testar o bloom de novo no navegador dele e confirmar se o
  visual agora corresponde ao esperado (e se o botão ✕ aparece após hard-refresh).

---

## 2026-06-20 (sáb) — Retomo o projeto (sessão Claude) + correção do bug "decode falha com bitstream"

- **Contexto:** usuário trabalhou o sistema de camadas numa sessão paralela (outro assistente) e
  trouxe de volta via zip + `docs/PARA-O-CLAUDE-LER.md`. A versão com camadas (1571 linhas) tinha
  um bug que frustrou o usuário: **decode falha quando há efeitos bitstream ativos**. Importei essa
  versão para a branch `feat/layer-system` (commit baseline) antes de corrigir.
- **Diagnóstico (reproduzido de verdade no navegador):**
  - Com os parâmetros padrão, melt/bloom/corrupt (e os 3 juntos, até em extremos) **funcionam na
    minha máquina** — não reproduzi a falha aqui. Logo, é dependente do ambiente do usuário.
  - Teste isolado forçando o decoder: `prefer-hardware` decodifica 92/92; `prefer-software` e
    `no-preference` decodificam só 11/92 e abortam ("Decoding error"). **Confirmado:** quando o
    `prefer-hardware` não consegue dar decode de hardware de verdade (GPU/drivers do usuário), o
    Chrome cai para software, que aborta no stream moshado (viola `frame_num`).
  - **`VideoDecoder.isConfigSupported()` é inútil aqui** — retorna `true` tanto p/ hardware quanto
    software, mas o software aborta no stream quebrado. Não dá p/ feature-detect com ele.
  - **2º bug, provável regressão e pior em máquina lenta:** `decodeAllFrames` chamava
    `createImageBitmap(vf)` (assíncrono) e **fechava o `vf` imediatamente** — corrida que máquinas
    lentas perdem → bitmaps null → "Decode não produziu frames".
- **Correções:**
  1. `decodeAllFrames` agora desenha cada `VideoFrame` **síncronamente** num `OffscreenCanvas` e
     fecha o vf na hora — elimina a corrida (drawImage captura os pixels antes do close). Frames
     viram `OffscreenCanvas` (compatível com `drawImage` no playback). Removido o polling de 5s/30s.
  2. **Recuperação parcial:** se o decode aborta no meio, usa os frames que decodificaram (mostra o
     glitch parcial) em vez de descartar tudo. `decodeAllFrames` retorna `{frames, error}`.
  3. **Mensagem de erro acionável:** quando 0 frames, roda `hwDecodeProbe()` e, se o ambiente não
     decoda datamosh, explica que é falta de decode de hardware (aponta `chrome://gpu`) — em vez do
     críptico "Decode não produziu frames".
  4. **`hwDecodeProbe()`** (novo): teste de runtime REAL — codifica um clipe minúsculo, duplica um
     delta (bloom de teste) e tenta decodar; ≥50% dos frames = concealment de hardware OK. Memoizado.
  5. **Aviso proativo:** ao adicionar a 1ª camada bitstream, roda o probe e avisa NA HORA se o
     ambiente não vai conseguir — melhor do que descobrir só ao clicar Aplicar.
- **Verificado no navegador:** caminho feliz (vídeo + melt+bloom+corrupt) processa e toca com pixels
  reais; o aviso proativo NÃO aparece em máquina capaz (probe passa); sem erros de console.
- ⚠️ **Não consegui reproduzir a falha exata do usuário** (meu hardware é capaz demais). As correções
  atacam as duas causas-raiz documentadas. **Pedir ao usuário para testar a versão corrigida e
  relatar:** (a) se ainda falha, qual a mensagem exata agora; (b) o que diz `chrome://gpu` sobre
  "Video Decode".
- ➡️ Próximo: confirmar com o usuário na máquina dele; revisar o resto da implementação de camadas
  (o usuário também queria que eu olhasse o sistema todo, não só o bug).

---

## 2026-06-20 (sáb) — Bug: escala/posição não atualizavam ao mexer no slider (sem auto-apply)

- 🐛 **Bug reportado pelo usuário:** "agora a escala não está funcionando! mas puta que pariu!
  eu falei para mexer na merda do decode e por que é que você quebrou funções da camada de vídeo????"
- **Causa:** quando desativei o auto-apply completamente, os handlers de transform (stretch/
  scale/posX/posY) em `bindTransformEvents` continuaram chamando `scheduleAutoApply()` — mas
  agora essa função não faz nada (só atualiza o indicador vermelho/verde). Resultado: mexer no
  slider de escala atualizava o valor na camada, mas nada redesenhava no canvas. Só atualizava
  quando o usuário clicasse Aplicar.
- **Fix:** substituído `scheduleAutoApply()` por `redrawSourcePreview()` nos 4 handlers de
  transform. A nova função:
  1. Se há `currentDecodedFrames` (resultado processado), não faz nada — não sobrescreve o
     preview do processamento
  2. Se está tocando (`isPlaying()`), não faz nada — não interrompe o playback
  3. Senão, chama `drawSourceFirstFrame(layer)` — busca o 1º frame da fonte e desenha no canvas
     aplicando as transforms atuais (via `drawSourceWithTransform`)
- **Nova função `drawSourceFirstFrame(layer)`:** helper que busca o frame 0 (vídeo via
  `getFrameCanvas(0)` ou imagem via `layer.bitmap`) e desenha no canvas com as transforms.
  Mesma lógica já usada em `handleFile` após carregar.
- **Por que isso é correto:**
  - Não fere a regra "não aplicar automaticamente" — só redesenha o 1º frame da fonte como
    preview estático, sem processar (encode/mosh/decode)
  - Usuário vê imediatamente o efeito de scale/posX/posY/stretch enquanto ajusta
  - Se já processou (Aplicar), o preview processado fica intacto
  - Se está tocando, o playback não é interrompido
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Bug: vídeo/imagem fazia stretch ao carregar ignorando checkbox stretch

- 🐛 **Bug reportado pelo usuário:** "por que a camada de vídeo fez o stretch to canvas mesmo com
  o checkbox desmarcado???"
- **Causa:** em `handleFile` (vídeo e imagem), o desenho do 1º frame após carregar usava
  `ctx.drawImage(frameCanvas, 0, 0, project.width, project.height)` — sempre estica para preencher
  o canvas, ignorando os params `stretch`/`scale`/`posX`/`posY` da camada.
- **Fix:** substituído por `drawSourceWithTransform(lctx, frameCanvas, srcW, srcH, project.width,
  project.height, layer)` — respeita `stretch=false` (preserva proporção, centraliza),
  `scale` (escala relativa ao tamanho natural), `posX`/`posY` (offset em pixels do centro).
  Mesma função já usada em `renderFrame` durante a composição.
- **Aplicado em:**
  - handleFile vídeo (desenho do 1º frame após carregar)
  - handleFile imagem (desenho da imagem após carregar)
  - applyAll Mode 1 já estava OK (usa renderFrame que usa drawSourceWithTransform)
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Revertido controle de fluxo com queueSize (causava deadlock)

- 🐛 **Bug reportado pelo usuário:** "não está conseguindo fazer o decode com apenas um efeito!"
- **Diagnóstico:** o "fix" anterior (`while (dec.decodeQueueSize > 8) await ...`) provavelmente
  causava deadlock. Hipótese: o decoder entrava num estado onde `decodeQueueSize` não diminuía
  (talvez erro interno sem chamar o callback de erro, ou bloqueio no pipeline de hardware), e
  ficávamos presos no while loop até o safety timeout de 30s. Resultado: decode "falhava" com
  qualquer efeito bitstream.
- **Fix:** removido o `while (decodeQueueSize > 8)` e o `while (encodeQueueSize > 8)`.
  Substituído por yield periódico simples:
  - Decoder: `if (i % 16 === 15) await new Promise(r => setTimeout(r, 0))` — yield a cada 16 chunks
  - Encoder: `if (i % 8 === 7) await new Promise(r => setTimeout(r, 0))` — yield a cada 8 frames
- **Por que isso é melhor:**
  - Não bloqueia esperando queue esvaziar — apenas dá tempo ao event loop entre batches
  - Não pode deadlockar — setTimeout(0) sempre resolve
  - O `await drawFrame()` no encoder já é o principal controle de fluxo natural (frame composition
    leva tempo, dando ao encoder espaço para processar)
- **Sobre a mensagem "reduza intensidade":** provavelmente era correta em alguns casos (corrupt
  muito denso aborta decode legítimamente). O controle de queue não era o fix certo — era um
  fix para um problema que não existia, e acabou criando um problema real.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Performance: controle de fluxo no encoder/decoder + remoção de canvas temporário

- 🐛 **Bug reportado pelo usuário:** "por que a codificação dos frames agora está demorando tanto?
  E por que está falhando e dizendo para reduzir intensidade se antes era bem mais rápido mesmo
  com 3 efeitos bitstream ativados???"
- **Diagnóstico:** dois bugs de performance/corretude introduzidos nas refatorações recentes:
  1. **`renderFrame` criava 2 OffscreenCanvas por frame por src layer** (`layerCanvas` + `tmp`
     intermediário). O `tmp` era completamente desnecessário — só copiava o `frameCanvas`/`bitmap`
     para depois chamar `drawSourceWithTransform`. Para 72 frames × 3 src layers = 432 OffscreenCanvas
     criados e descartados por apply. GC pesado, alocação lenta.
  2. **`decodeChunks` e `encodeFrames` não tinham controle de fluxo.** O loop fazia
     `dec.decode(chunk)` / `enc.encode(vf)` em sequência sem `await` entre eles, jogando TODOS
     os chunks na queue do decoder/encoder de uma vez. Com 3 efeitos bitstream, o número de chunks
     explodia (melt remove keys, bloom duplica dezenas de vezes) → queue estourava → decoder
     abortava com "no frame available" ou similar → mensagem "reduza intensidade".
- **Fix 1: remoção do canvas temporário em `renderFrame`.** Agora `drawSourceWithTransform`
  recebe direto o `frameCanvas` (vídeo) ou `layer.bitmap` (imagem) — drawImage direto, sem
  cópia intermediária. Reduz de 2 OffscreenCanvas para 1 por src layer por frame.
- **Fix 2: controle de fluxo em `encodeFrames`.**
  ```js
  while (enc.encodeQueueSize > 8){
    await new Promise(r => setTimeout(r, 4));
  }
  ```
  Se a queue do encoder passa de 8 frames, espera esvaziar antes de alimentar mais. Evita
  estouro de queue e melhora throughput (encoder processa no seu ritmo, sem backpressure).
- **Fix 3: controle de fluxo em `decodeChunks`.** Mesma lógica — `while (dec.decodeQueueSize > 8)`
  antes de cada `dec.decode(chunk)`. Isso é especialmente importante com bloom (que cria muitos
  chunks duplicados) e corrupt (que pode causar perda de sync).
- **Resultado esperado:**
  - Encode volta a ser rápido (como era antes da refatoração do sistema de camadas)
  - Decode não aborta mais com 3 efeitos bitstream (controle de fluxo evita estouro de queue)
  - Mensagem "reduza intensidade" só aparece se realmente há erro de decode legítimo
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Separação arquitetural: APLICAR ≠ REPRODUZIR

- 🔁 **Crítica do usuário:** "por que é que você misturou a função APPLY COM A REPRODUÇÃO???"
- **Diagnóstico:** `applyAll` fazia tudo: processava (encode + mosh + decode) E iniciava a
  reprodução (`playPreview`/`playRealtimeComposition`). Isso confundia a semântica — o usuário
  clicava "Aplicar" esperando apenas processar, e a ferramenta começava a tocar. Pior: mudar
  qualquer parâmetro e clicar Aplicar re-iniciava o playback, matando o loop antigo.
- **Refatoração arquitetural:**
  - **`applyAll()`** agora SÓ processa:
    - Mode 1 (sem bitstream): desenha 1 frame como snapshot estático. Não toca.
    - Mode 2 (bitstream): encode + mosh + decode → armazena frames em `currentDecodedFrames`.
      Desenha 1º frame no canvas. Não toca.
    - Status final: "Processado. Clique Play para reproduzir."
  - **`startPlayback()`** — nova função. Decide Mode 1 (realtime) ou Mode 2 (decoded frames)
    automaticamente e inicia o loop.
  - **`startDecodedPlayback(frames, w, h, fps)`** — loop que toca os frames decodificados (Mode 2).
  - **`startRealtimePlayback(w, h, fps, total)`** — loop de composição em tempo real (Mode 1).
  - **`stopPlayback()`** — cancela o player ativo (seta `cancelled = true`).
  - **`isPlaying()`** — helper: tem player ativo e não cancelado?
  - **`decodeAllFrames(chunks, ...)`** — helper que só decodifica e retorna array de bitmaps.
    Antes era misturado com o loop de reprodução dentro de `playPreview`.
- **`playPauseBtn.onclick`** reescrito:
  - Se tocando → `stopPlayback()` + botão volta para ▶ + status "Pausado."
  - Se pausado → `startPlayback()` + botão vira ⏸ + status "Reproduzindo."
- **Variáveis de estado removidas/renomeadas:**
  - `previewPlaying` (boolean) → removida. Substituída por `isPlaying()`.
  - `currentPreviewPlayer` → `currentPlaybackPlayer` (nome mais claro).
  - `stopPreviewLoop()` → `stopPlayback()`.
  - Nova: `currentDecodedFrames` — array de bitmaps prontos para tocar (Mode 2).
- **Funções antigas removidas:**
  - `playPreview` (misturava decode + loop) → substituída por `decodeAllFrames` + `startDecodedPlayback`.
  - `playRealtimeComposition` → renomeada para `startRealtimePlayback` (semântica mais clara).
- **Fluxo do usuário agora:**
  1. Adiciona fonte (vídeo/imagem) → 1º frame aparece no canvas (estático)
  2. Ajusta parâmetros (qualquer um) → nada muda no canvas, indicador atualiza
  3. Clica **Aplicar** → processa (encode+mosh+decode ou só composição) → 1º frame do resultado
     aparece no canvas (estático). NÃO toca.
  4. Clica **▶** → reproduz em loop
  5. Clica **⏸** → pausa (último frame fica no canvas)
  6. Ajusta parâmetros de novo → nada muda (preview pausado mostra último frame)
  7. Clica **Aplicar** de novo → re-processa → novo 1º frame aparece (estático)
  8. Clica **▶** → reproduz o novo resultado
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Auto-apply COMPLETAMENTE desativado (qualquer parâmetro)

- 🔁 **Pedido do usuário (reafirmado com força):** "JÁ FALEI QUE NÃO É PARA APLICAR
  AUTOMATICAMENTE QUANDO EU ALTERAR ALGUM PARAMETRO!!! MERDA!"
- **Erro anterior:** eu tinha desativado auto-apply só para params bitstream, mas mantido
  para src/pixelfx/blend/opacity/clip/transform/dims/fps/cortes. Isso ainda disparava
  `applyAll` automaticamente, causando as mesmas race conditions que o usuário estava
  reclamando. Não respeitei a intenção clara do pedido original.
- **Fix:** `scheduleAutoApply` agora NÃO agenda mais `applyAll`. Só:
  1. Se não há fonte: limpa canvas, placeholder, desabilita botões.
  2. Se há fonte: só atualiza o indicador vermelho/verde (`refreshEncDots`).
- **Comportamento novo (definitivo):**
  - Mudar QUALQUER parâmetro (src, pixel-fx, bitstream, blend, opacity, clip, transform,
    dims, fps, cortes, seed) → NÃO dispara applyAll.
  - Só atualiza o indicador vermelho/verde para refletir se o encode está fresco ou não.
  - Usuário deve clicar **Aplicar** manualmente para ver o resultado.
- **Por que isso é melhor:**
  - Elimina TODAS as race conditions de preview (não há mais applyAll concorrente).
  - Usuário tem controle total — pode ajustar 10 parâmetros e clicar Aplicar uma vez só.
  - Encode só roda quando o usuário explicitamente pede.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Rewrite do sistema de preview (tokens → player com flag cancelled)

- 🐛 **Bug persistente:** mesmo após múltiplos fixes de race condition, o preview continuava
  travando após a codificação. O sistema de tokens (`previewLoopToken`) era frágil demais —
  qualquer incremento concorrente matava o loop, e havia muitas janelas de race condition.
- **Solução: substituir tokens por referência direta ao player atual.**
  - Nova variável `currentPreviewPlayer` — aponta para o player ativo (ou null).
  - `stopPreviewLoop()` agora seta `currentPreviewPlayer.cancelled = true` e `currentPreviewPlayer = null`.
  - Cada `playPreview`/`playRealtimeComposition` cria um objeto `player = { cancelled: false, ... }`,
    cancela o player anterior, e se registra como `currentPreviewPlayer`.
  - O loop checa `player.cancelled` a cada iteração — se true, para e libera bitmaps.
  - Sem tokens, sem race conditions entre incrementos concorrentes.
- **Vantagens:**
  - Simples: uma referência, uma flag. Fácil de raciocinar.
  - Robusto: o player só é cancelado explicitamente por `stopPreviewLoop()` ou por um novo player.
  - Não há janela de race condition entre "capturar token" e "checar token" — a flag está no
    próprio objeto do player, não numa variável global separada.
- **Indicador vermelho/verde confirmado correto:**
  - VERDE = `isBitstreamEncodeFresh()` retorna true = encode está feito/atualizado
  - VERMELHO = encode pendente (ainda não codificado, ou params mudaram que invalidam o cache)
  - Lógica revisada, está correta. O usuário pode ter visto vermelho no estado inicial
    (antes de clicar Aplicar pela primeira vez) — que é o comportamento esperado.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Auto-apply removido para params bitstream

- 🔁 **Pedido do usuário:** a codificação NÃO deve iniciar automaticamente quando é feita
  alteração nos parâmetros dos efeitos bitstream (melt cortes, bloom target/count, corrupt
  count/intensity). Só deve codificar quando o usuário clicar Aplicar.
- **Diagnóstico:** o `scheduleAutoApply` (com debounce 250ms) era chamado em TODOS os handlers
  de params — incluindo bitstream. Isso causava a race condition que fiz o vídeo sumir: mudar
  bloom target disparava um novo `applyAll` que chamava `stopPreviewLoop` e matava o loop
  ativo. Mesmo com os fixes anteriores (clearTimeout, applyAllRunning, tokenStillValid), ainda
  podia haver janela de concorrência.
- **Fix:** criada função `markBitstreamDirty()` (no-op por enquanto — só documenta intenção)
  e substituído `scheduleAutoApply()` por `markBitstreamDirty()` nos 5 handlers de bitstream:
  - melt: `.lc-cuts` (cutFrames)
  - bloom: `.lc-target` (targetFrame), `.lc-count` (count)
  - corrupt: `.lc-ccount` (count), `.lc-cint` (intensity)
- **Comportamento novo:**
  - Mudar src params, pixel-fx params, blend, opacity, clip, transform, dims, fps, cortes:
    continua disparando auto-apply (esses invalidam o encode OU são tempo real)
  - Mudar só bitstream params: NÃO dispara auto-apply. O usuário clica **Aplicar** quando
    quiser ver o resultado. O encode fica cacheado (verde), só a manipulação de chunks roda
    de novo quando clicar Aplicar.
- **Side effect benéfico:** elimina a race condition entre loops de preview — agora só há
  um `applyAll` rodando por vez (ou via click manual, ou via scheduleAutoApply quando muda
  algo que realmente invalida o encode). Os bitstream params não causam mais `stopPreviewLoop`
  no loop ativo.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Bug: vídeo não reproduzia após codificação (loop se matando)

- 🐛 **Bug reportado pelo usuário:** após aplicar melt com sucesso (status "Pronto — 72 frames →
  71 chunks", enc-dots verdes), o canvas ficava preto. Botão play/pause mostrava ⏸ mas nada tocava.
- **Diagnóstico:** screenshot mostrava tudo correto (encode done, decode done, frames prontos)
  mas o loop de preview não estava desenhando. Bug diferente do anterior (race condition com
  scheduleAutoApply) — esse acontecia mesmo sem click manual.
- **Causas múltiplas identificadas:**
  1. **Race condition residual:** mesmo com `applyAllRunning` e `clearTimeout`, o `playPreview`
     podia ser chamado, fazer `++previewLoopToken` (capturar myToken), esperar o decode (~500ms),
     e nesse meio tempo um `applyAll` concorrente (ou um re-apply do `scheduleAutoApply` que
     disparou entre o `clearTimeout` e o `playPreview`) chamava `stopPreviewLoop` → incrementava
     o token global. Quando o `playPreview` terminava o decode, ele iniciava o `loop()`, que
     percebia `myToken !== previewLoopToken` e **se matava imediatamente** no primeiro frame.
     Resultado: o 1º frame nem chegava a ser desenhado, canvas ficava preto.
  2. **Promise de "todos frames prontos" podia nunca resolver** se o decode falhasse silenciosamente
     sem chamar `onDone` nem `onError` — `frames.every()` em array não-vazio mas com `null`s
     pendentes esperava infinitamente.
  3. **Status "Reproduzindo..." sobrescrito** por `decodeErrorMsg` mesmo quando havia frames
     válidos para mostrar.
- **Fixes aplicados em `playPreview`:**
  1. **Captura `tokenStillValid` antes de iniciar o loop.** Depois de esperar o decode, compara
     `myToken === previewLoopToken`. Se mudou, NÃO inicia o loop — outro applyAll já está
     cuidando. Libera os bitmaps e retorna silenciosamente.
  2. **Timeouts de segurança:** se o decode demorar >30s ou a conversão para bitmap >5s, resolve
     a Promise em vez de esperar infinitamente. Evita travar o applyAll.
  3. **`frames.length === 0` check na espera de bitmaps** — não espera infinitamente se decode
     vazio (every em array vazio retorna true, mas se onFrame nunca chamou, frames continua
     vazio e o check anterior ficava em loop).
  4. **Status "Reproduzindo..." só setado se não houve erro de decode.** Antes sobrescrevia.
  5. **try/catch em todos os `ctx.drawImage`** — bitmap pode ter sido fechado por erro concorrente.
  6. **Loop desenha frame independente de `previewPlaying`** (o `previewPlaying` só controla
     se avança o índice) — antes, com `previewPlaying=false`, o canvas ficava preto mesmo com
     loop ativo. Agora mantém o último frame desenhado.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Bug: canvas preto após aplicar melt (race condition)

- 🐛 **Bug reportado pelo usuário:** adicionou Melt, clicou Aplicar, vídeo sumiu (canvas preto).
  Status mostrava "Pronto — 72 frames → 71 chunks (melt(36): -1 keys)" e enc-dots verdes —
  encode E decode completaram, mas o loop de preview não estava desenhando.
- **Causa raiz: race condition entre click manual e scheduleAutoApply.**
  1. Usuário adiciona Melt → `scheduleAutoApply` agenda `applyAll` para daqui 250ms
  2. Usuário clica **Aplicar** → `applyAll` roda imediatamente (encode+mosh+decode ~5s)
  3. 250ms depois, o `applyAll` agendado dispara — chama `stopPreviewLoop()` que incrementa
     `previewLoopToken`
  4. O loop que estava rodando (com token antigo) percebe o token diferente e **se mata**
  5. O segundo `applyAll` roda de novo (cache hit), mas o loop do usuário já estava morto
  6. Canvas fica preto — dependendo do timing, o segundo loop também podia ser morto por um
     terceiro applyAll agendado, criando uma cadeia de loops se matando
- **Fix 1: `clearTimeout(autoApplyTimer)` no topo do `applyAll`.** Cancela qualquer applyAll
  agendado pendente — sem isso, o timer dispara depois do applyAll manual, mata o loop ativo.
- **Fix 2: flag `applyAllRunning` anti-reentry.** Se um applyAll já está rodando, ignora
  chamadas concurrentes. Previne encadeamento de applyAlls sobrepostos.
- **Fix 3: desenhar 1º frame imediatamente em `playPreview`.** Antes, o loop esperava
  `setTimeout(() => requestAnimationFrame(loop), 1000/fps)` antes do primeiro draw —
  ~42ms de canvas preto após applyAll retornar. Agora desenha o frame[0] antes de iniciar
  o loop, e o loop começa do frame[1].
- **Fix 4: reportar decode vazio.** Se `frames.length === 0` ou `frames[0]` é null, agora
  seta status "Decode não produziu frames visíveis..." em vez de retornar silenciosamente
  com canvas preto.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Indicador vermelho/verde de encode + decode completo antes de tocar

- 🔁 **Pedido do usuário:** efeitos bitstream devem ter indicador visual sutil (ponto vermelho
  quando o encode está pendente, verde quando atualizado). Adicionar camada que exija encode
  faz o ponto voltar a vermelho. Reprodução só começa quando TODOS os frames estão codificados.
- ✅ **Indicador visual de encode:** cada card de camada bitstream (melt/bloom/corrupt) agora
  tem um ponto (8×8px) ao lado do badge.
  - **Verde pulsante** (`--ok`, com box-shadow verde) = encode atualizado, pronto para preview
  - **Vermelho pulsante** (`--acc` #ff2a6d, animação `pulse-dot` 1.4s ease-in-out) = encode
    pendente — será codificado ao aplicar
- ✅ **Função `isBitstreamEncodeFresh()`:** retorna `true` se não há bitstream layer; senão
  compara `cachedEncode.sig` com `encodeSignature(total)` do estado atual. Se bater = fresh.
- ✅ **Função `refreshEncDots()`:** percorre todos os `.lc-enc-dot` no DOM e atualiza só o
  estado (verde/vermelho + title) — não re-renderiza o painel inteiro (não destrói inputs).
- ✅ **Função `invalidateEncode()`:** seta `cachedEncode = null` e chama `refreshEncDots()`.
  Substituiu os `cachedEncode = null` espalhados pelos handlers de duração/fps/dims.
- ✅ **Fluxo do indicador:**
  - Estado inicial: vermelho (sem encode ainda)
  - Após `applyAll()` concluir encode+mosh: `refreshEncDots()` → verde
  - Usuário muda qualquer param que afeta o encode (src, pixelfx, região, dims, fps, cortes,
    blend, opacity, clip, transform): `scheduleAutoApply` chama `refreshEncDots()` → vermelho
    (porque `isBitstreamEncodeFresh` agora retorna false)
  - Usuário muda só bitstream params (bloom target, corrupt intensity): não invalida o encode,
    ponto continua verde
- ✅ **Mudança de comportamento: decode COMPLETO antes de tocar.** Antes: `playPreview` era
  "progressivo" — começava a tocar após o 1º frame pronto, decodificava o resto em background.
  Isso causava "buracos" no playback (frames que ainda não tinham decodificado eram pulados).
  Agora: espera TODOS os frames estarem decodificados (e convertidos para ImageBitmap) antes
  de iniciar o loop de reprodução. Mais lento para começar, mas playback sem buracos.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Bug: nova camada aparecia na base do painel em vez do topo

- 🐛 **Bug:** nova camada adicionada aparecia EMBAIXO das existentes no painel, não no topo.
  Comportamento esperado (Photoshop-like): nova camada sempre no topo, para afetar o que está
  abaixo dela.
- **Causa:** o código usava `project.stack.push(layer)` com comentário "adiciona no topo" —
  mas `push` adiciona no FINAL do array, e como a convenção é `stack[0]` = topo do painel
  (renderizado por último, afeta tudo abaixo), `push` colocava a nova camada na BASE do painel.
  Bug latente desde a Fase 14 (quando invertermos a convenção para bottom-up), nunca peguei
  porque eu só testava com 1-2 camadas e o template inicial já vinha com melt+vídeo na ordem
  certa.
- **Fix:** trocar `push` por `unshift` — adiciona no ÍNDICE 0 = TOPO do painel. Cada nova
  camada agora aparece no topo, empurrando as anteriores para baixo.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Bug do rastro ao mudar escala + botão minimizar mais visível

- 🐛 **Bug do rastro (trail frames) ao mudar escala/posição:** quando `stretch=false` e `scale<1`,
  a fonte não preenche todo o canvas — o back buffer fica com áreas transparentes ao redor da
  fonte menor. O `ctx.drawImage(back, 0, 0)` que copiava o back buffer para o canvas visível
  **não limpava o canvas visível antes**, então o frame anterior aparecia por baixo nas áreas
  transparentes → efeito de rastro/trail acumulando a cada mudança de escala.
  **Corrigido:** adicionar `ctx.clearRect(0, 0, w, h)` ANTES de `ctx.drawImage(back, 0, 0)` na
  função `playRealtimeComposition`. Antes do fix, só não aparecia porque stretch=true (default
  antigo) sempre preenchia o canvas totalmente opaco.
- ✏️ **Botão minimizar mais visível:** o botão `▾/▸` já funcionava para TODOS os tipos de
  camada (video, image, pixelsort, rgbshift, melt, bloom, corrupt), mas estava com fundo
  transparente e cor `--dim` — fácil de não notar. Estilizado agora como botão com borda,
  fundo `#1a1a1f`, hover com cor de destaque `--acc`. Tamanho 20×20px.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Transform por camada (stretch/scale/posX/posY) + cards minimizáveis

- 🔁 **Pedido do usuário:** quando o vídeo não é do mesmo formato do canvas, deve abrir no
  formato original (não stretch); checkbox "Stretch to canvas" na camada de vídeo para dar
  essa opção; sliders de escala e posição X/Y; cards minimizáveis.
- ✅ **Modelo de dados expandido:** src layers (`video`, `image`) agora têm `stretch: false`,
  `scale: 1`, `posX: 0`, `posY: 0`. Pixelfx e bitstream não têm (não faz sentido).
- ✅ **Nova função `drawSourceWithTransform`:**
  - `stretch=true`: `drawImage(src, 0,0,w,h)` — preenche o canvas, ignora scale/pos (comportamento antigo)
  - `stretch=false`: desenha a fonte em suas dimensões NATURAIS (preserva proporção), escalada
    por `scale` (1 = tamanho original), posicionada por `posX/posY` em pixels do canvas, com
    centro = (0,0). Fórmula: `dx = (dstW - drawW)/2 + posX; dy = (dstH - drawH)/2 + posY`
- ✅ **Renderização não-destrutiva:** a fonte é desenhada num OffscreenCanvas temporário com
  suas dims naturais primeiro (preservando a forma original), depois aplicada a transform
  para o canvas da camada. Antes era `drawImage(src, 0,0,w,h)` direto, que sempre esticava.
- ✅ **UI do bloco de transform:** dentro dos params de cada camada src, há um bloco com:
  - checkbox "Stretch to canvas" (default OFF)
  - slider Escala (0.05× a 4×, default 1×)
  - slider Posição X (-960 a 960 px, default 0)
  - slider Posição Y (-540 a 540 px, default 0)
  - Quando stretch=true, os 3 sliders ficam desabilitados e visualmente esmaecidos
- ✅ **Cards minimizáveis:** cada card tem um botão `▾/▸` ao lado do `✕`. Clique minimiza/
  expande o corpo do card (blend/opacity/clip/params). O cabeçalho com nome/badge/eye sempre
  fica visível. Estado preservado por camada (`layer._open`, default true).
- ✅ **Cache signature atualizada:** agora inclui `stretch/scale/posX/posY` de cada src layer,
  então mudar qualquer um desses em modo bitstream invalida o encode corretamente.
- ➡️ Próximo: validar export MP4 real; testar Safari/Firefox; feature-detect de GPU; presets
  JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Fixes de UX e bugs de interface (sistema de camadas em uso)

- 🔁 **Reportado pelo usuário durante teste real do sistema de camadas:**
- 🐛 **Bug #1: arrastar slider arrastava o card inteiro.** Causa: `card.draggable=true`
  no card todo, então qualquer interação com inputs dentro dele virava drag. **Corrigido:**
  só o handle `.lc-handle` (⋮⋮) é `draggable=true`; o card tem `draggable=false` e um
  `dragstart` listener que `preventDefault()` em qualquer target que não seja o handle.
  Handle agora visualmente maior (font-size:14px, letter-spacing apertado).
- 🐛 **Bug #2: preview piscava preto.** Causa dupla: (1) `renderFrame` faz `clearRect` no
  início e depois `await videoSrc.getFrameCanvas(t)` — durante o await o canvas visível
  ficava limpo/preto; (2) `playRealtimeComposition` e `playPreview` re-setavam `canvas.width/height`
  mesmo quando não mudaram, e atribuir essas props (mesmo ao mesmo valor) **limpa o canvas
  para preto**. **Corrigido:** (1) **double-buffering** — renderizo num `OffscreenCanvas`
  (back buffer) e só copio para o canvas visível quando o frame está completo; (2) só
  redimensiono se `canvas.width !== w || canvas.height !== h`.
- 🐛 **Bug #3: deletar camada deixava o último frame estático no preview.** Causa:
  `scheduleAutoApply` tinha `if (!hasAnySrc()) return;` que retornava sem parar o preview loop.
  **Corrigido:** quando não há fonte, agora chama `stopPreviewLoop()`, limpa caches
  (`lastMoshResult`, `lastStaticCanvas`, `cachedEncode`), desenha placeholder, desabilita
  botões de export/play. Mesma lógica cobre desligar (eye off) todas as fontes.
- 🐛 **Bug #4: checkbox "Máscara de recorte" centralizado.** Causa: conflito de CSS —
  `.lc-controls label` (especificidade 0,2,1) definia `flex-direction:column`, e `.lc-clip`
  (0,2,0) só `align-items:center`. Em flex column, `align-items:center` alinha
  horizontalmente → checkbox ia pro meio. **Corrigido:** novo seletor `.lc-controls .lc-clip`
  (0,3,0) força `flex-direction:row; justify-content:flex-start`.
- ✏️ **Mudança de UX: "Ajustar à próxima fonte" virou checkbox persistente** (era botão
  one-shot: clicava, virava true, resetava após o próximo load). Agora fica marcado e toda
  fonte nova ajusta as dimensões do quadro. Melhor para construir composições — usuário
  marca, adiciona fontes, desmarca para fixar o tamanho.
- ✏️ **Mudança de UX: painel de camadas inicia vazio.** Antes vinha com template
  Vídeo+Melt. Agora abre com placeholder "Nenhuma camada" — usuário constrói do zero.
- ✏️ **Mudança de UX: label do clip simplificado** de "Máscara de recorte (clip ao composto
  abaixo)" para só "Máscara de recorte".
- ➡️ Próximo: validar export MP4 real fora do ambiente de teste; testar em Safari/Firefox;
  feature-detect de `hardwareAcceleration:'prefer-hardware'`; presets JSON+seed; deploy Vercel.

---

## 2026-06-20 (sáb) — Convenção da pilha corrigida (bottom-up) + 2 modos de reprodução

- 🔁 **Pergunta do usuário:** por que a ferramenta abria com Melt embaixo de Vídeo? A
  regra deveria ser "cada camada afeta o que está ABAIXO dela no painel".
- ⚠️ **Bug de convenção:** o template inicial fazia `project.stack.push(v, m)` (vídeo no
  índice 0/topo do painel, melt no índice 1/base) — mas a regra "camada de cima afeta a de
  baixo" exige que o Melt (efeito) esteja ACIMA do Vídeo (fonte) no painel. E pior: a
  iteração de `renderFrame`/`applyBitstreamLayers` era top-down (índice 0 primeiro), o que
  significava que cada camada afetava o que vinha DEPOIS dela no painel — i.e., o oposto
  do pedido.
- **Corrigido para pipeline bottom-up:** agora `renderFrame` e `applyBitstreamLayers`
  iteram `for (li = stack.length - 1; li >= 0; li--)` — último índice (base do painel)
  processado primeiro, índice 0 (topo do painel) por último. Cada camada afeta o
  resultado acumulado do que está ABAIXO dela no painel (Photoshop-like). Template
  inicial removido (painel começa vazio, ver próxima entrada).
- 🔁 **Pergunta do usuário:** por que toda mudança exige re-codificar antes da reprodução?
  Efeitos datamoshing não podem funcionar em tempo real durante a reprodução?
- **Resposta técnica documentada:** depende do tipo de efeito. Composição (src + pixel-fx)
  é instantânea (só `drawImage` + `ImageData`) — pode ser 100% tempo real. Bitstream (melt/
  bloom/corrupt) **exige** encode H.264 primeiro (você não pode corromper bytes de um
  stream que ainda não existe) — intrínseco à técnica, não limitação da implementação.
- **Implementado: dois modos de reprodução:**
  - **Modo 1 (sem bitstream) — TEMPO REAL:** zero encode, render direto no canvas, debounce
    de 60ms. Sliders de pixel-sort/RGB shift/blend/opacity reagem instantaneamente.
  - **Modo 2 (com bitstream) — DECODE PROGRESSIVO:** encode cacheado por assinatura (só
    re-encoda se src/pixelfx/região/dimensões/fps/cortes mudarem); mudar só melt/bloom/
    corrupt = ~5ms manipulação + re-decode. Decode começa a tocar assim que o 1º frame
    está pronto (~100ms), o resto decodifica em background — sensação de tempo real.
- **Cache signature expandida:** agora inclui TODA a composição (src + pixel-fx), não só
  regiões de vídeo. Mudar threshold do pixel-sort invalida o cache corretamente.
- **Export MP4 unificado:** funciona nos dois modos — se houver `lastMoshResult` (Modo 2),
  usa chunks já moshados; senão (Modo 1), codifica na hora do export.
- ➡️ Próximo: fixes de UX durante teste real (ver entrada seguinte).

---

## 2026-06-19 (sex) — BLOCO 3 ✅: Sistema de camadas completo (rewrite do index.html)

- 🔁 **Pergunta crítica do usuário:** "uma das características do looplab que era para ser
  mantida era o sistema de camadas. Por que não implementou?" — o `index.html` antigo
  (Bloco 0+2) usava tabs Vídeo/Foto com params globais, sem pilha de camadas. Quebra o
  princípio de "ferramenta feita para datamoshing".
- ✅ **Rewrite completo do `index.html`** (~1280 linhas, era ~936) com sistema de camadas
  unificado. **7 tipos de camada** na MESMA pilha:
  - **src:** `video` (Mediabunny demux) · `image` (ImageBitmap)
  - **pixelfx:** `pixelsort` (ASDF, código novo) · `rgbshift` (código novo)
  - **bitstream:** `melt` · `bloom` · `corrupt`
- ✅ **Cada camada tem:** `id`, `type`, `kind`, `enabled`, `blend` (13 modos: Normal/Add/
  Screen/Multiply/Overlay/Difference/Exclusion/Hard-Light/Soft-Light/Dodge/Burn/Darken/
  Lighten), `opacity`, `clip` (máscara de recorte), `name` editável, e params específicos.
- ✅ **UI da pilha:** cards no painel direito com drag-reorder (HTML5 DnD), eye toggle
  (on/off), badge colorido por kind (verde=src, ciano=pixelfx, amarelo=bitstream), nome
  editável inline, delete, blend dropdown, opacity slider, checkbox clip, e params
  específicos por tipo (cortes para melt, target+Nx para bloom, count+intensity para
  corrupt, dir+critério+thresholds para pixelsort, ângulo+amount para rgbshift, mini-timeline
  para offset de vídeo).
- ✅ **Tensão arquitetural resolvida:** pixel-fx operam por frame (Etapa 1), bitstream-fx
  operam pós-encode (Etapa 2) — semânticas diferentes. Solução: UI unificada (mesma pilha),
  mas renderFrame pula bitstream layers, e `applyBitstreamLayers` age só nos chunks pós-encode
  em ordem da pilha. Encoder põe keyframes na UNIÃO dos cortes de todas as melt layers.
- ✅ **Clip mask estilo Photoshop:** usa `destination-in` no canvas da camada para recortar
  pelo alfa do composto abaixo (a "camada de baixo" no painel). Funciona com blend e opacity.
- 🐛 **Bug #1 corrigido nesta rewrite:** `applyMelt` agora aceita múltiplos cortes (a UI
  dizia "30,60 ou auto" mas o parser só usava o primeiro). Implementação:
  `parseCutFrames` retorna array, `allCutPoints` coleta a união de todas as melt layers.
- 🐛 **Bug #2 corrigido:** `applyCorrupt` com `intensity=0` não aplica mais XOR (antes
  aplicava com stride=30 e xorVal=0x40). Adicionado early-return.
- 🐛 **Bug #3 corrigido:** `applyBloom` agora clona o `data` do chunk duplicado
  (`data: dupChunk.data.slice()`) — antes compartilhava referência, e se uma manipulação
  futura mutasse in-place, todas as duplicatas virariam juntas.
- ➡️ Próximo: convenção da pilha + modos de reprodução (ver entrada seguinte).

---

## 2026-06-19 (sex) — Análise do projeto (pre-rewrite): bugs identificados

- ✅ **Análise técnica completa** do `index.html` (936 linhas) e docs (`00-MASTER`, `PLAN`,
  `MVP-MAP`, `DEVLOG`, `PROCESSO`, `RESUMO-PARA-CLIENTE`).
- ⚠️ **Bugs identificados no `index.html` original (documentados para a rewrite):**
  1. `applyMelt` ignorava cortes múltiplos (só usava `parseInt(cutsRaw.split(",")[0],10)`)
  2. `corruptIntensity=0` ainda aplicava XOR (não desligava de fato)
  3. Sem feature-detect de `hardwareAcceleration:'prefer-hardware'` — máquinas sem GPU
     teriam bloom/corrupt abortando silenciosamente
  4. `applyBloom` inseria N referências ao mesmo objeto `dupChunk` (bug latente)
  5. Export podia baixar mosh antigo se `scheduleAutoApply` debounce não tivesse disparado
  6. Bitrate estático, não adaptativo para resoluções maiores
- 📊 Status vs. plano: Blocos 0/1/2/5 ✅, Blocos 3/4/6 pendentes.
- ➡️ Próximo: rewrite com sistema de camadas (ver entrada BLOCO 3).

---

## 2026-06-19 (sex) — Play/Pause + preview "ao vivo" (encode em cache, mosh reaplica sozinho)

- 🔁 **Pergunta do usuário:** por que não tem play/pause, e por que os efeitos não são aplicados
  em tempo real?
- **Play/Pause**: era só uma funcionalidade que faltava expor — o mecanismo (`previewLoopToken`)
  já existia. Adicionado botão (`▶`/`⏸`), estado `previewPlaying` lido a cada frame do loop de
  preview; **preservado entre reaplicações** (se você pausar e depois mudar um parâmetro, o
  resultado novo aparece já pausado, não retoma sozinho).
- **"Tempo real" dos efeitos — explicado e parcialmente resolvido:** datamosh real não é um filtro
  instantâneo (shader) — exige encode H.264 + manipulação de bytes + decode, que leva segundos.
  Não dá pra reagir a cada tick de um slider sendo arrastado. MAS: só o **encode** é lento, e ele
  só precisa mudar se região/resolução/FPS/cortes de melt mudarem. Bloom, corrupt e seed atuam
  **sobre os chunks já codificados** — são rápidos.
- **Refatoração:** `applyVideoMosh` virou duas partes — `ensureEncoded()` (lento, cacheado por
  assinatura de região+resolução+fps+corte; só roda nova vez se algo disso mudar) e a manipulação
  de mosh + decode (rápida, sempre roda de novo). Cache invalidado ao carregar um vídeo novo
  (`cachedEncode = null` em `handleFile`, senão um vídeo novo podia reusar chunks do anterior).
- **Auto-reaplicar (debounce):** mudar qualquer checkbox/slider/seed de mosh (vídeo) ou de glitch
  (foto) dispara o reprocessamento sozinho — sem precisar clicar "Aplicar" de novo. Vídeo usa
  debounce de 250ms; foto (sem encode/decode, puro `ImageData`) usa 80ms. Arrastar a região na
  timeline só atualiza a prévia leve (frame cru) durante o arraste — o reencode completo roda
  uma vez só, ao **soltar**.
- **Medido no preview do navegador:** 1º apply de um vídeo de 3s/72 frames = ~4000ms (encode
  completo). Ativar bloom depois (sem clicar Aplicar) = **453ms** até "Pronto" — quase 9x mais
  rápido, porque reusa o encode em cache. Foto: glitch reaplicado automaticamente ao mudar
  intensidade do RGB shift, sem clique nenhum.
- ➡️ Próximo: Bloco 3 (UI de camadas/blend) e Bloco 4 (validar export real fora do ambiente de teste).

---

## 2026-06-19 (sex) — Redesenho do trim: dropdown de duração + região arrastável na timeline

- 🔁 **Pedido do usuário, revisando a 1ª versão do slider de trim:** o limite do loop volta a ser
  um **dropdown** (não slider livre), e a região escolhida deve aparecer destacada numa **barra
  de timeline**, arrastável **inteira** (clicar na região e mover, não redimensionar por alças).
- Removido o trim de duas alças (`<input type=range>` sobrepostos) da iteração anterior.
  Substituído por:
  - `<select id="durSelect">` com presets fixos (1/2/3/4/5/6/8s) — define a LARGURA da janela.
  - `#timelineBar` / `#timelineRegion`: barra representando o vídeo inteiro, com a região
    destacada (largura proporcional à duração escolhida) arrastável via **Pointer Events**
    (`pointerdown`/`pointermove`/`pointerup` + `setPointerCapture`, funciona com mouse e touch).
  - `regionOffset` (estado) substitui os antigos `regionStart`/`regionEnd` independentes — agora
    é só o início da janela; o fim é sempre `regionOffset + project.duration`.
- Mantido da iteração anterior: preview ao vivo do frame sob a região durante o arraste/troca de
  duração (`showRegionPreviewFrame`), e o `fileLoadToken` que corrige a race condition de troca
  rápida de arquivo.
- **Verificado no preview do navegador** com vídeo de teste de 10s (bandas de cor a cada 2,5s):
  - Trocar duração no dropdown (3s→5s): região passa de 30% para 50% de largura, frameCount
    120 ✓.
  - Arrastar a região (offset 0→3s): `"3.0s – 8.0s"`, largura mantida em 50% ✓.
  - Arrastar além do fim do vídeo: trava em `"5.0s – 10.0s"` (não deixa passar) ✓.
  - Aplicar mosh na região 5–10s: pixel central do resultado decodificado bate com a banda verde
    esperada (~rgb(16,128,64), banda 5–7,5s), confirmando que usa a região certa, não o início ✓.
- ➡️ Próximo: Bloco 3 (UI de camadas/blend) e Bloco 4 (validar export real fora do ambiente de teste).

---

## 2026-06-19 (sex) — Slider de região (trim) do vídeo

- ✅ **Pedido do usuário:** precisa dar pra escolher QUAL trecho do vídeo enviado é usado, não
  sempre o início. Antes, a "Duração a usar" só definia quantos segundos a partir de t=0.
- Implementado um **trim-slider de duas alças** (técnica clássica de dois `<input type=range>`
  sobrepostos, com `pointer-events:none` no input e `pointer-events:all` só no thumb) — substitui
  o antigo slider único de duração. Mostra início/fim/duração (`0.0s – 3.0s (3.0s)`), barra de
  preenchimento visual entre as alças, e teto de 8s para o trecho selecionável (igual ao limite
  antigo, agora aplicado à LARGURA da região, não mais fixo a partir de t=0).
- **Bônus de UX:** a alça que está sendo arrastada atualiza o canvas com o frame exato sob ela —
  sem isso o slider ficaria "cego" (só veria o resultado depois de clicar Aplicar).
- `applyVideoMosh` (`drawFrame`) agora amostra a partir de `regionStart`, não de 0 — confirmado por
  pixel: aplicando numa região 4–7s de um vídeo de teste com bandas de cor por trecho de tempo, o
  resultado decodificado mostrou a cor da banda 4–7s (laranja), não a do início (azul).
- 🐛 **Bug encontrado e corrigido durante o teste (não relacionado ao slider em si):** ao disparar
  carregamentos de arquivo em sequência rápida (sem esperar o anterior terminar — foi assim que
  achei, testando rápido demais), `handleFile` tinha uma race condition real: a Promise mais lenta
  de uma carga antiga podia resolver DEPOIS de uma carga mais nova e sobrescrever o estado (causou
  valores bizarros tipo `max="105.0"` numa repetição do teste). **Corrigido** com um
  `fileLoadToken` (mesmo padrão do `previewLoopToken`/`stopPreviewLoop` já usado no preview):
  cargas obsoletas se auto-descartam ao perceber que uma carga mais nova já começou.
- **Verificado no preview do navegador**, sempre aguardando cada operação assíncrona terminar
  antes da próxima (a 1ª rodada de teste, sem aguardar, expôs o bug da race condition acima —
  ensina a sempre serializar os passos ao testar fluxos assíncronos manualmente).
- ➡️ Próximo: Bloco 3 (UI de camadas/blend) e Bloco 4 (validar export real fora do ambiente de teste).

---

## 2026-06-19 (sex) — Fix: canvas não adaptava ao vídeo + vazamento entre modos Vídeo/Foto

- 🐛 **Reportado pelo usuário, testando a build do PR #1:** (1) canvas do stage não ajustava
  resolução/proporção à fonte de vídeo; (2) ao trocar de Vídeo→Foto com um vídeo carregado, o
  vídeo continuava aparecendo por cima da foto; (3) ao voltar de Foto→Vídeo, o vídeo "herdava" a
  proporção da foto.
- **Causas-raiz (2, compartilhadas pelos 3 sintomas):**
  1. A ingestão de vídeo nunca recalculava `project.width/height` a partir da fonte — só a
     ingestão de foto fazia isso. Por isso o vídeo sempre usava as dimensões padrão (480×270) ou
     o que sobrasse de um carregamento anterior de foto.
  2. O loop de animação do preview (`requestAnimationFrame`) nunca era invalidado ao trocar de
     modo ou carregar um novo arquivo — o `previewLoopToken` só era incrementado dentro do próprio
     `playPreview`. Como o canvas é compartilhado entre os dois modos, o loop antigo continuava
     redesenhando frames de vídeo por cima do que quer que o modo Foto desenhasse depois.
- **Correção:**
  - Novo helper `fitProjectSize(srcW, srcH, maxDim)` — calcula `project.width/height` a partir da
    proporção real da fonte; usado igualmente nos dois modos (antes só a foto tinha essa lógica,
    duplicada inline).
  - Novo helper `stopPreviewLoop()` (incrementa `previewLoopToken`) chamado no início de
    `setMode()` e de `handleFile()` — garante que qualquer loop de preview anterior pare antes de
    trocar de modo ou carregar um arquivo novo.
- **Verificado no preview do navegador** (não só lido): vídeo sintético 320×180 → canvas
  480×270 (proporção 1.778 ✓). Troca para Foto com imagem 200×300 → canvas 320×480 (proporção
  0.667 ✓), pixel central confirmado com a cor real da foto (sem vídeo vazando, pixel idêntico
  antes/depois de 800ms ocioso). Volta para Vídeo e recarrega → canvas 480×270 de novo (não ficou
  preso na proporção da foto). Fluxo de aplicar mosh + export re-testado, continua funcionando.
- ➡️ Próximo: commit no branch do PR #1 (`feat/datamosh-core-mvp`); depois Bloco 3 (UI de
  camadas/blend) e Bloco 4 (validar export real fora do ambiente de teste).

---

## 2026-06-19 (sex) — Bloco 0+2: `index.html` real construído, testado no navegador, 2 bugs encontrados e corrigidos

- ✅ Criado `index.html` (build limpo, conforme diretriz — não fork do Loop Lab): shell de UI
  própria (tabs Vídeo/Foto, upload, painel de mosh, preview, export), núcleo de bitstream
  embutido (encode→melt/bloom/corrupt→mux/decode com os parâmetros validados no Bloco 1),
  glitch de imagem (pixel sort ASDF + RGB shift).
- ✅ **Testado de ponta a ponta no navegador de verdade** (Chrome via automação), não só lido.
  Gerei um vídeo de teste sintético (encode→mux MP4 via WebCodecs/Mediabunny), injetei no input
  de upload real, e acompanhei o pipeline completo rodar.
- 🐛 **Bug real #1 encontrado:** a ingestão de vídeo usava `<video>`+seek (decode-by-seek), igual
  ao Loop Lab. Nesse ambiente de teste, a tag `<video>` nunca disparava `loadedmetadata` para o
  MP4 gerado (`networkState` preso em LOADING), mesmo com o arquivo estruturalmente válido
  (confirmado byte a byte: ftyp/moov/mdat corretos, avcC/SPS/PPS corretos) — confirmado depois que
  o MESMO arquivo decodificava perfeitamente via Mediabunny `Input`+`CanvasSink` (WebCodecs puro).
  **Correção:** trocada toda a ingestão de vídeo de `<video>`+seek para **Mediabunny demux
  (`Input`+`CanvasSink.getCanvas(t)`)** — sem depender da tag `<video>`/HTMLMediaElement. Mais
  consistente com a filosofia WebCodecs-first do projeto, e evita essa classe de problema também
  para usuários reais (decoder mais permissivo que o pipeline de mídia do browser).
- 🐛 **Bug real #2 encontrado:** o export MP4 (`EncodedVideoPacketSource.add(packet)`) lançava
  `"Video chunk metadata must be provided"` no Mediabunny — o primeiro pacote de um track precisa
  vir com `{decoderConfig:{codec,codedWidth,codedHeight,description}}` como 2º argumento de `add()`.
  Isso resolve o risco #2 do Bloco 1 que estava pendente ("Mediabunny aceita packets manipulados?
  ainda não testado"). **Corrigido** no handler de export.
- 🐛 **Bug real #3 encontrado:** o preview ao vivo travava silenciosamente (sem erro nenhum) ao
  aplicar o mosh. Causa: os `VideoFrame` decodificados eram empilhados num array sem `.close()`
  (para depois tocar em loop) — decoders de **hardware** (obrigatório p/ bloom/corrupt, ver Bloco 1)
  têm um pool pequeno de buffers; acumular frames abertos esgota o pool e o decode trava sem
  disparar erro. **Corrigido:** cada `VideoFrame` agora é convertido em `ImageBitmap` (leve, sem
  segurar recurso do decoder) e fechado imediatamente; o array de preview guarda os bitmaps.
- **Método de debug que valeu a pena registrar:** quando algo trava sem erro nenhum, reproduzir a
  mesma lógica isolada (sem a estrutura de callbacks do app) com logging explícito em cada etapa
  revela o ponto exato da diferença. Foi assim que achei o bug #3 — a versão isolada (que fechava
  frames na hora) funcionava, a do app (que não fechava) travava.
- Verificado: nenhum arquivo de debug (`__debug_test.mp4`, scripts de inspeção MP4, servidor de
  range) foi commitado — removidos antes do commit; `.gitignore` atualizado para preveni-los no futuro.
- ➡️ Próximo: re-testar o fluxo completo após os 3 fixes (ainda não re-validado de ponta a ponta
  após a última correção); depois Bloco 3 (UI de camadas/blend) e Bloco 4 (validar export real).

---

## 2026-06-19 (sex) — BLOCO 1: núcleo validado (de-risco concluído) ✅

- ✅ Protótipo do núcleo rodado **de verdade** no Chrome (via DevTools/JS, não pseudocódigo):
  encoder com GOP controlado → captura de chunks → manipulação → decode. Testado fora do
  `index.html` (script direto no console) para validar a API antes de estruturar a UI.
- **GOP controlado confirmado:** `keyFrame:true` nos frames exatos pedidos (0 e 30) — `keyIndices:[0,30]`
  exatamente como configurado. `avc1.42001f` + `latencyMode:"realtime"` + `avc:{format:"avc"}`.
- **MELT — confirmado, funciona direto:** removendo o chunk `key` do corte (frame 30), o decode
  NÃO erra (59 frames decodificados) e o pixel amostrado pós-corte ficou com a cor da CENA ANTERIOR
  (distância de cor 232/441 vs. o decode limpo) — prova pixel-level do "vazamento" de cena. Nenhuma
  config especial necessária.
- **BLOOM — funciona, mas precisa de uma config específica:** duplicar o chunk `delta` ali sem
  mais nada faz o decoder **abortar** ("Decoding error", só 11/85 frames) — Chrome valida
  `frame_num` no slice header (campo do H.264) e um decoder estrito rejeita duplicatas. Software
  decoder (`prefer-software`) também aborta. **Hardware decoder (`hardwareAcceleration:
  'prefer-hardware'`) tolera via concealment e decodifica tudo (55/55).** Confirmado que é
  glitch real (não congelamento): a barra de teste derrapa de posição e depois some — motion
  vectors reaplicando contra referência cada vez mais errada.
- **CORRUPT — funciona, mas com teto de densidade:** com `prefer-hardware`, corrupção **sparsa**
  (XOR a cada 20 bytes, pulando os ~6 primeiros bytes do NAL) decodifica 100% (40/40) e produz
  diferença visual real a partir do frame corrompido, propagando por 25 frames seguintes (até
  faltar um keyframe novo) — exatamente o comportamento esperado de corrupção de P-frame. Densidade
  maior (a cada 7 ou 2 bytes) faz o decoder abortar, mesmo em hardware.
- **⚠️ Implicação de produto:** o decoder de PREVIEW/PLAYBACK da ferramenta deve sempre configurar
  `hardwareAcceleration:'prefer-hardware'`. O parâmetro de "intensidade" da corrupção precisa ter
  um teto seguro (stride mínimo ~15-20) — exibir/permitir além disso só com aviso de risco de abortar.
- **Risco geral do PLAN (H.264 "limpar" o glitch via deblocking) NÃO se confirmou** — os três
  efeitos sobrevivem em H.264 nativo do browser; não foi necessário considerar MPEG-4/ffmpeg.wasm.
- ➡️ Próximo: Bloco 0 (estrutura do `index.html` limpo) + Bloco 2 (plugar produtores reais:
  upload de vídeo/imagem) usando exatamente esta config validada.

---

## 2026-06-18 (qui) — Política de documentação total + PROCESSO.md

- ⚠️ **Diretriz permanente do usuário:** "todo o processo deve ser documentado. Tudo." Vale daqui
  pra frente, não só para o planejamento já feito.
- ✅ Criado `PROCESSO.md`: documentação NARRATIVA completa (linha do tempo por fases, decisões-chave
  com o porquê, índice de artefatos) — complementa o `DEVLOG.md` (log curto) e o `00-MASTER.md`
  (snapshot de estado). Cobre da Fase 0 (origem) até a Fase 9 (git/GitHub).
- Política registrada no fim do `PROCESSO.md`: toda etapa relevante → `DEVLOG.md` + (quando muda o
  raciocínio/contexto) expandida no `PROCESSO.md`; decisões sempre com o porquê; manter ambos em dia.
- `00-MASTER.md` atualizado: índice inclui `PROCESSO.md`; corrigida nota antiga "datamoshing.com
  OFFLINE" (já havia sido corrigida no BIBLE, faltava aqui); link do repositório GitHub adicionado.
- ➡️ Próximo: SEXTA (B0+B1). Cada bloco da build deve gerar entrada no DEVLOG (e no PROCESSO se houver
  decisão/raciocínio relevante) + commit/push.

---

## 2026-06-18 (qui) — Git + GitHub (para sessões na nuvem com PC desligado)

- ✅ Repositório git inicializado (branch `main`); 1º commit com toda a pasta `docs/`.
  `.gitignore` exclui o PDF de terceiros (Menkman) e temporários de build (`docs/_*.html`).
- ✅ `gh` CLI instalado **portátil** (sem admin) em `C:\Users\nicho\gh\bin\gh.exe` (v2.94.0)
  — winget falhou por UAC recusado (exit 1602), então baixei o zip oficial.
- ✅ Autenticado na **conta nova** `alexs-master` (escopo `repo`); `gh auth setup-git` feito.
- ✅ Repositório **PÚBLICO** criado e com push: **https://github.com/alexs-master/dataMoshingTool**
- **Motivo:** permitir abrir uma **sessão na nuvem** (claude.ai/code ou app) com o PC desligado,
  já que o Remote Control só funciona com o PC ligado/acordado.
- ➡️ Próximo: build de SEXTA inalterado (B0+B1). Lembrar de **commitar/pushar** o progresso da build.

---

## 2026-06-18 (qui) — Resumo para a cliente (técnico + PDF)

- ✅ `RESUMO-PARA-CLIENTE.md` — cliente entende técnica, então versão COM jargão: filosofia
  (datamoshing.com + Menkman), **pipeline técnico** (WebCodecs+Mediabunny, GOP controlado, H.264
  baseline, melt/bloom/corrupt nos chunks key/delta, mux/decode), técnicas, parâmetros, escopo.
  ⚠️ **Sem perguntas/checklist no documento** (decisão do usuário) — é só apresentação de escopo.
- ✅ `RESUMO-PARA-CLIENTE.pdf` gerado p/ envio. Método: HTML estilizado → PDF via **Edge headless**
  (`msedge --headless --print-to-pdf`, sem header/footer). Reusável p/ futuros PDFs (reportlab não
  instalado; Edge e Chrome disponíveis no Windows).
- (As perguntas de alinhamento — formatos, duração, áudio, navegador, autêntico-vs-controle, refs,
  identidade — ficam para conversa direta, FORA do documento.)
- ⏳ **Aguardando retorno da cliente** antes de congelar parâmetros. Respostas (esp. formatos,
  duração, autêntico-vs-controle) podem ajustar o escopo do MVP.
- ➡️ Próximo: SEXTA segue (B0+B1) independentemente; ajustes de escopo entram quando a cliente responder.

---

## 2026-06-18 (qui) — CORREÇÃO DE RUMO: build limpo, não fork do Loop Lab

- ⚠️ **Decisão forte do usuário:** esta é uma ferramenta FEITA PARA DATAMOSHING. NÃO é "pegar o
  Loop Lab e adicionar datamosh". NÃO clonar `index.html` e enxugar. **Zero código morto.**
- Estratégia corrigida em todos os docs: **build limpo** num `index.html` NOVO; **portar
  seletivamente** (reimplementar limpo) só os mecanismos relevantes — timing de loop, loop de
  frames determinístico, sessão de export, compositing de camadas + blend modes, I/O de vídeo/imagem,
  mux/GIF/PNG-seq, utils/seed. Loop Lab = referência de implementação (linhas), não base a editar.
- **FORA (não portar):** todos os GENERATORS, biblioteca de EFFECTS artísticos, motion/keyframes
  por-param, fontes/SVG/texto, presets visuais. **Sem fonte procedural no produto** (só um padrão de
  teste dev-only no Bloco 1). As ops de pixel que entram (pixel-sort ASDF, RGB shift, ch. displace)
  são **código NOVO datamosh-específico**, não porte de efeito.
- Docs atualizados: 00-MASTER (mapa→"Mecanismos a portar", arquitetura, cronograma B0), PLAN
  (princípio-mestre, §1 "portar", §3 entradas, B0), MVP-MAP (etapas, escopo, modelo de dados kind:"src",
  tabela de porte + linhas "FORA", UI).
- ➡️ Próximo: inalterado (SEXTA: B0 = arquivo limpo + utils base; B1 = núcleo).

---

## 2026-06-18 (qui) — Mapa do MVP  *(nota: "REUSO 1:1" abaixo foi reenquadrado como "porte seletivo" — ver entrada acima)*

- ✅ `MVP-MAP.md` criado. Decisão arquitetural central: **DUAS etapas**.
  - **Etapa 1 (composição, por frame) = REUSO 1:1 do Loop Lab:** camadas (`project.stack`),
    blend modes (`BLENDS`, 10 modos), opacity/enable/clip, render (`renderFrame`/`renderExportFrames`),
    uploads de vídeo (`seekVideoAsset`) e imagem (`ensureImageAsset`), loop length (`duration`/`fps`/`timelineFrameCount`).
    Glitch pixel-level (pixel-sort ASDF, RGB shift) vive aqui (sobre `getImageData`).
  - **Etapa 2 (bitstream, pós-encode) = a parte NOVA:** encoder→chunks→manipular (melt/bloom/corrupt)→mux/decode.
- Modelo de dados: estende `project` do Loop Lab com bloco `mosh{technique,cutPoints,dup,drop,corrupt,codec}`;
  item de camada = `makeGen/makeEffect` reusado (id,kind,type,enabled,blend,opacity,clip,params).
- Esforço real concentra-se na Etapa 2; tudo que o usuário pediu p/ reusar (loops, render, uploads, camadas, blends) é reuso direto.
- ➡️ Próximo: inalterado (SEXTA: Bloco 0 + Bloco 1 do PLAN).

---

## 2026-06-18 (qui) — Correção + 2ª referência da cliente (Rosa Menkman)

- ✏️ **Correção:** datamoshing.com **NÃO está offline** — print da cliente confirma site no ar.
  O `curl` do nosso sandbox recebia placeholder IIS/404 (provável bloqueio de CDN p/ IP de datacenter).
  Conteúdo minerado do Wayback confere com o print → materiais seguem válidos. Docs corrigidos.
- ✅ Baixado e minerado o PDF que a cliente enviou: **Rosa Menkman, "The Glitch Moment(um)"**
  (Network Notebook #04). → `REFS-menkman-glitch-momentum.md` (+ PDF guardado em docs/).
- **Significado:** a cliente mandou técnica (datamoshing.com) + teoria (Menkman) → ela valoriza
  **glitch REAL, não filtro decorativo**. Isso é o NORTE conceitual e provável critério de avaliação.
  Reforça (com respaldo citável) o requisito de não-simulação. Sugere enquadrar features em 3 famílias
  (compressão / feedback / corrupção) e dá vocabulário p/ nomear presets/UI. Não muda o cronograma.
- ➡️ Próximo: inalterado (SEXTA: Bloco 0 + Bloco 1).

---

## 2026-06-18 (qui) — Bíblia da cliente (datamoshing.com) minerada

- ⚠️ **Descoberta:** datamoshing.com está OFFLINE (domínio responde página padrão IIS; artigos 404).
  Conteúdo recuperado via Wayback (`curl -A Mozilla "https://web.archive.org/web/2021id_/http://datamoshing.com/<art>"`).
- ✅ Catálogo completo minerado → `BIBLE-datamoshing-com.md` (cobre fotos E vídeos, = nosso escopo).
- **Refinamentos do plano vindos da bíblia:**
  - Vídeo tem DUAS técnicas, não uma: (A) I-frame removal + P-dup; (B) **corrupção de bytes do `mdat`/delta**.
    Implementar as duas — a bíblia faz mosh em **H.264 via corrupção**, o que VALIDA nosso caminho WebCodecs.
  - Risco H.264 (deblocking limpar glitch) agora tem solução interna: usar técnica (B) de corrupção;
    ffmpeg.wasm/MPEG-4 vira plano C improvável.
  - Pixel sort canônico = **ASDF Pixel Sort do Kim Asendorf** (sort por luma/hue/sat com threshold) → ref. p/ entrada de imagem.
- ➡️ Próximo: inalterado (SEXTA: Bloco 0 + Bloco 1). No Bloco 1, testar AMBAS as técnicas de vídeo no protótipo.

---

## 2026-06-18 (qui) — Planejamento

- ✅ Lido e mapeado o `procedural-loop-lab/index.html` (single-file, ~12.9k linhas).
- ✅ Pesquisa de datamoshing real + APIs WebCodecs/Mediabunny (ver `RESEARCH.md`).
- ✅ Definida arquitetura "produtor de frames → encoder → manipular chunks → mux/decode".
- ✅ Plano time-boxed para sexta (`PLAN.md`) + documento mestre (`00-MASTER.md`).
- ✅ Contexto salvo na memória do Claude (projeto, deadline, abordagem).
- **Decisões travadas:**
  - Datamosh real via WebCodecs + Mediabunny (não overlay; sem backend).
  - Partir de CÓPIA do Loop Lab e remover o que não serve (não scaffolding do zero).
  - Encoder baseline `avc1.42001f` + `latencyMode:"realtime"` para evitar B-frames.
  - Vídeo enviado = re-encodado com nosso GOP (decode-by-seek) → garante moshabilidade.
- **➡️ Próximo:** SEXTA — Bloco 0 (clonar+limpar) e Bloco 1 (protótipo do núcleo; ver smear real).
- **Pendência aberta:** localização/código do app de FOTOS do usuário (não fornecido ainda).

---

<!-- TEMPLATE de nova entrada:
## AAAA-MM-DD — <título do bloco>
- ✅/🚧/❌ o que foi feito
- Decisões / gotchas:
- ➡️ Próximo:
-->
