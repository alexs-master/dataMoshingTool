# 📚 Documentação do Processo (completa)

> Registro narrativo e cronológico de **todo** o processo do projeto — contexto, o que foi feito,
> decisões e seus porquês, e resultados. Padrão do projeto: **tudo é documentado.**
>
> Relação com os outros docs:
> - `00-MASTER.md` = fonte de verdade / estado atual (snapshot).
> - `DEVLOG.md` = log curto por etapa (o que mudou).
> - **`PROCESSO.md` (este)** = a história completa com raciocínio e contexto.
> - `PLAN.md`, `MVP-MAP.md`, `RESEARCH.md`, `BIBLE-…`, `REFS-…`, `RESUMO-PARA-CLIENTE.md` = artefatos.

---

## Linha do tempo

### Fase 0 — Origem e contexto (2026-06-18)
- O usuário (Nicholas) perguntou se eu lembrava de sessões anteriores. **Constatação:** não há
  memória automática entre sessões; a pasta de memória do projeto estava vazia; o diretório
  `dataMoshingTool` não era repositório git e estava vazio.
- O usuário citou dois apps anteriores: um de **efeitos procedurais para fotos** e outro para
  **vídeos** (o **Procedural Loop Lab**, https://procedural-loop-lab.vercel.app/ , fonte em
  `../procedural-loop-lab/index.html`).
- **Pedido central:** criar uma ferramenta **só de datamoshing**.

### Fase 1 — Definição de requisitos
- Via perguntas dirigidas, ficou definido o **requisito inquebrável**: datamosh **real**,
  manipulando o **bitstream codificado** — **nada** de simulação, filtro CSS ou overlay.
- Entrada: **foto E vídeo**.
- O usuário não conhece os internos de datamoshing → autenticidade/correção é responsabilidade minha.
- O usuário interrompeu uma tentativa de scaffold (create-next-app): a fase era de **planejamento**, não código.

### Fase 2 — Análise do Procedural Loop Lab (referência de reuso)
- Li e mapeei o `index.html` do Loop Lab (single-file, ~12.9k linhas, estética terminal verde-no-preto).
- Identifiquei mecanismos relevantes: timing de loop, render determinístico (`renderFrame`/
  `renderExportFrames`), sessão de export, pilha de camadas + blend modes, I/O de vídeo (decode-by-seek)
  e imagem, mux MP4 (Mediabunny) / GIF / PNG-seq, RNG com seed.

### Fase 3 — Pesquisa de datamoshing
- Técnicas reais: **I-frame removal (melt)** e **P-frame duplication (bloom)**; substrato = compressão
  temporal (I-frames vs P/B-frames).
- Caminho browser-native definido: **WebCodecs + Mediabunny** (demux/encode → manipular chunks → mux/
  decode), sem servidor. Encoder H.264 baseline `avc1.42001f` + `latencyMode:"realtime"` p/ evitar B-frames.
- Artefato: `RESEARCH.md`.

### Fase 4 — Referências da cliente
- **datamoshing.com** (Sterling Crispin) — a "bíblia" técnica. *Detalhe de processo:* meu `curl` do
  sandbox recebia só uma página placeholder IIS/404 (provável bloqueio de CDN p/ IP de datacenter);
  **o site está no ar para a cliente** (confirmado por print). Minerei o conteúdo idêntico via Wayback.
  Catálogo cobre vídeo (I-frame removal, corrupção do `mdat`, automação) e imagem (pixel sort=ASDF/Kim
  Asendorf, corrupção de JPG, RGB shift). Artefato: `BIBLE-datamoshing-com.md`.
- **Rosa Menkman — *The Glitch Moment(um)*** (PDF enviado pela cliente). Texto teórico fundador;
  tese: glitch real (artefato do encode/decode) ≠ filtro decorativo (que vira "moda"). Framework de
  3 pontos de quebra (encoding/decoding, feedback, glitch/corrupção). Artefatos:
  `REFS-menkman-glitch-momentum.md` + PDF (mantido local, fora do git).
- **Refinamentos vindos das referências:** (a) vídeo passa a ter DUAS técnicas — remoção de I-frame
  **e** corrupção de bytes de delta (a bíblia faz mosh em H.264 via corrupção, validando nosso caminho);
  (b) pixel sort canônico = ASDF; (c) risco de "H.264 limpar o glitch" resolvido pela corrupção.

### Fase 5 — Plano e Mapa do MVP
- `PLAN.md`: plano time-boxed (entrega **sábado 2026-06-20**, build **sexta**), com pseudocódigo do
  núcleo (encoder→chunks→manipular→mux/decode) e ordem de blocos (de-risco do núcleo primeiro).
- `MVP-MAP.md`: arquitetura em **duas etapas** — (1) composição por frame; (2) bitstream pós-encode —
  escopo, modelo de dados, UI.

### Fase 6 — Correção de rumo: build LIMPO (não fork)
- **Decisão forte do usuário:** a ferramenta é FEITA PARA DATAMOSHING; **não** clonar o Loop Lab e
  enxugar; **zero código morto**. Estratégia corrigida: arquivo `index.html` **novo**, **portando
  seletivamente** (reescrevendo limpo) só os mecanismos relevantes; **fora** todos os GENERATORS e a
  biblioteca de EFFECTS artísticos; sem fonte procedural no produto. Ops de pixel que entram
  (pixel-sort, RGB shift, displace) são **código novo**. Docs atualizados (MASTER/PLAN/MVP-MAP).

### Fase 7 — Resumo para a cliente
- `RESUMO-PARA-CLIENTE.md` + **PDF**. Versão **técnica** (a cliente entende jargão): filosofia,
  pipeline (WebCodecs/Mediabunny, GOP, melt/bloom/corrupt), parâmetros, escopo.
- *Iterações pedidas pelo usuário:* (1) **remover as perguntas/checklist** do documento;
  (2) **remover a seção "fica para a 2ª etapa"**. Documento final = só apresentação do escopo.
- *Método de PDF:* HTML estilizado → **Edge headless** (`msedge --headless --print-to-pdf`,
  sem header/footer). reportlab não instalado; Edge e Chrome disponíveis no Windows.

### Fase 8 — Remote Control (continuar de outro dispositivo)
- Explicado: **Remote Control** (claude.ai/code + app) é camada de sync sobre a sessão **local** —
  ativar com `/remote-control`; requer Claude Code v2.1.51+, plano Pro/Max/Team/Enterprise.
- **PC desligado:** Remote Control **não** funciona (a sessão roda na máquina). Para continuar com o
  PC off é preciso **sessão na nuvem**, que exige o projeto num repositório → motivou a Fase 9.

### Fase 9 — Git + GitHub
- `git init` (branch `main`), `.gitignore` (exclui PDF de terceiros e temporários `docs/_*.html`),
  1º commit com toda a `docs/`.
- **`gh` CLI:** winget falhou por UAC recusado (exit 1602) → instalado **portátil** (sem admin) em
  `C:\Users\nicho\gh\bin\gh.exe` (v2.94.0).
- **Troca de conta:** login feito pelo usuário no navegador → autenticado na conta **`alexs-master`**
  (escopo `repo`); `gh auth setup-git` p/ o git usar essa conta.
- Repositório **PÚBLICO** criado e com push: **https://github.com/alexs-master/dataMoshingTool**.
- Objetivo: habilitar **sessões na nuvem** com o PC desligado.
- Também fixada **política de documentação total** (diretriz do usuário) e criado `PROCESSO.md`
  (este documento) como registro narrativo completo, complementar ao `DEVLOG.md` (log curto) e ao
  `00-MASTER.md` (snapshot de estado).

### Fase 10 — Bloco 1: validação real do núcleo de bitstream (2026-06-19)
- Em vez de só planejar, **rodei o protótipo de verdade** no navegador (via Chrome DevTools/JS,
  fora do `index.html`) para validar o risco central do projeto antes de construir qualquer UI.
- Testadas as 3 técnicas de mosh com chunks reais de um encoder H.264 (`avc1.42001f`,
  `latencyMode:"realtime"`, GOP forçado): **melt** (remover chunk `key`), **bloom** (duplicar chunk
  `delta`), **corrupt** (XOR em bytes de chunk `delta`).
- **Melt confirmado funcionando puro**, sem nenhuma config extra — prova pixel-level (distância de
  cor 232/441 entre decode limpo e moshado, exatamente onde a cena deveria ter mudado).
- **Achado não previsto:** bloom e corrupt, feitos do jeito ingênuo, fazem o `VideoDecoder` do
  Chrome **abortar** com erro ("Decoding error") em vez de produzir glitch — porque o H.264 exige
  continuidade estrita do campo `frame_num` no slice header, e um decoder rigoroso (software/CPU)
  rejeita streams "incorretos". **Solução encontrada por teste empírico:** configurar o decoder
  com `hardwareAcceleration:'prefer-hardware'` — o decoder de hardware faz *concealment* (tolera o
  erro de spec) e revela o artefato visual real (confirmado: o elemento de teste derrapa de posição
  e se desfaz, não congela — prova de que são os motion vectors se reaplicando, não um freeze).
- **Segundo achado:** a corrupção de bytes tem um **teto de densidade** — esparsa (1 a cada ~20
  bytes) sobrevive e propaga visualmente por dezenas de frames; mais densa que isso (1 a cada 7 ou
  2 bytes) derruba o decoder mesmo em modo hardware. Vira parâmetro de UI com piso de segurança.
- **Consequência para o plano:** o risco "H.264 pode limpar o glitch via deblocking" (que motivava
  um possível plano B com MPEG-4/ffmpeg.wasm) **não se confirmou da forma esperada** — o problema
  real não era o deblocking, era a validação de spec do decoder, resolvida com uma flag de
  configuração. **Plano B descartado como desnecessário.**
- Docs atualizados com os parâmetros corretos: `PLAN.md` §2b/2d/§5 (pseudocódigo corrigido,
  riscos marcados como resolvidos), `00-MASTER.md` (status, próxima ação, seção de riscos
  reescrita como "resultado"), `DEVLOG.md` (entrada detalhada com todos os números do experimento).

### Fase 11 — Bloco 0+2: `index.html` real, testado no navegador, 3 bugs corrigidos (2026-06-19)
- Construído o `index.html` de verdade (build limpo): UI própria, núcleo de bitstream embutido
  com os parâmetros do Bloco 1, glitch de imagem (pixel sort ASDF + RGB shift).
- Em vez de só escrever o código, **testei rodando no navegador de verdade** — gerei um vídeo de
  teste sintético (via WebCodecs/Mediabunny), injetei no upload real, e segui o pipeline rodar.
- **3 bugs reais encontrados e corrigidos durante o teste**, todos documentados em detalhe no
  `DEVLOG.md` (entrada 2026-06-19, Bloco 0+2):
  1. Ingestão de vídeo via `<video>`+seek nunca carregava (travava em `loadedmetadata`) mesmo com
     arquivo MP4 estruturalmente válido — confirmado comparando com decode via Mediabunny
     `Input`+`CanvasSink` (WebCodecs puro), que funcionou perfeitamente no mesmo arquivo.
     **Correção:** ingestão de vídeo migrada inteiramente para demux via Mediabunny, sem `<video>`.
  2. Export MP4 falhava porque o primeiro pacote enviado ao `EncodedVideoPacketSource` do
     Mediabunny precisa do `decoderConfig` (avcC) como metadata — resolve o risco #2 que tinha
     ficado pendente no Bloco 1.
  3. Preview ao vivo travava silenciosamente: `VideoFrame`s decodificados ficavam acumulados sem
     fechar (para tocar em loop depois) e esgotavam o pool de buffers do decoder de hardware.
     **Correção:** converter cada frame em `ImageBitmap` e fechar o `VideoFrame` na hora.
- **Lição de processo registrada:** quando algo trava sem erro, reproduzir a mesma lógica isolada
  com logging em cada etapa (fora da estrutura de callbacks do app) revela o ponto exato do bug
  por comparação — foi assim que o bug #3 foi encontrado.
- Artefatos de debug (vídeo de teste, scripts de inspeção de MP4, servidor HTTP ad-hoc) foram
  removidos antes do commit; `.gitignore` atualizado para preveni-los no futuro.

### Fase 12 — Análise técnica do projeto (2026-06-19/20, sessão de revisão)
- O usuário pediu "analise isto" sobre o arquivo `dataMoshingTool.rar` enviado (todo o projeto
  até então: `index.html` + 8 markdowns + 2 PDFs).
- **Análise completa** do `index.html` (936 linhas) e dos docs. Identifiquei a arquitetura em
  duas etapas (composição → bitstream), os acertos de engenharia (build limpo, encoder baseline,
  Mediabunny demux, cache do encode, race conditions tratadas com tokens), as três técnicas
  validadas empiricamente (melt, bloom, corrupt com `prefer-hardware`), e **6 bugs não
  documentados** quemereciam correção antes da entrega:
  1. `applyMelt` ignorava cortes múltiplos (só usava o primeiro)
  2. `corruptIntensity=0` ainda aplicava XOR
  3. Sem feature-detect de `hardwareAcceleration:'prefer-hardware'`
  4. `applyBloom` compartilhava referência ao chunk duplicado (bug latente)
  5. Export podia baixar mosh antigo se o debounce não tivesse disparado
  6. Bitrate estático, não adaptativo
- **Status vs. plano:** Blocos 0/1/2/5 ✅, Blocos 3 (UI camadas/blend), 4 (export real) e 6
  (polish/deploy) pendentes — exatamente o sistema de camadas era a peça grande que faltava.

### Fase 13 — Bloco 3 ✅: Sistema de camadas completo (rewrite do `index.html`, 2026-06-19/20)
- 🔁 **Crítica central do usuário:** "uma das características do looplab que era para ser mantida
  era o sistema de camadas. Por que não implementou? Tanto vídeo quanto foto devem ser geradores
  que aparecem no painel de camadas e possam ser empilhados e reordenados, além de terem blend
  modes e clipping mask. Os efeitos datamosh também devem ser da mesma forma."
- **Diagnóstico:** o `index.html` da Fase 11 usava tabs Vídeo/Foto com params globais — não havia
  pilha de camadas. Isso quebrava o princípio de "ferramenta feita para datamoshing" estabelecido
  no `MVP-MAP.md` (que listava explicitamente o sistema de camadas como `♻️ PORTAR`).
- **Tensão arquitetural identificada:** pixel-fx operam por frame (Etapa 1, durante a composição);
  bitstream-fx operam pós-encode (Etapa 2, sobre os chunks H.264). Semânticas fundamentalmente
  diferentes — não dá para simplesmente jogar tudo na mesma pilha e renderizar top-down.
- **Solução adotada:** **UI unificada, execução em duas fases.** Todas as camadas aparecem na
  mesma pilha (com badge colorido por kind: verde=src, ciano=pixelfx, amarelo=bitstream) e o
  usuário reordena tudo junto. Mas `renderFrame` percorre a pilha bottom-up e **pula**
  bitstream layers (que não têm efeito durante a composição); `applyBitstreamLayers` age só
  depois do encode, também bottom-up, sobre os chunks. Encoder põe keyframes na UNIÃO dos
  cortes de todas as melt layers, garantindo que cada uma tenha o que remover.
- **7 tipos de camada:** video (Mediabunny demux), image (ImageBitmap), pixelsort (ASDF, código
  novo), rgbshift (código novo), melt, bloom, corrupt. Cada uma com params específicos.
- **Cada card na UI tem:** drag handle (⋮⋮), eye toggle, badge do kind, nome editável, delete,
  blend dropdown (13 modos), opacity slider, checkbox de máscara de recorte, e params específicos.
- **Clip mask estilo Photoshop:** usa `destination-in` no canvas da camada para recortar pelo
  alfa do composto abaixo — funciona com blend e opacity.
- **Bugs corrigidos nesta rewrite** (3 dos 6 identificados na Fase 12):
  - Multi-corte melt: `parseCutFrames` retorna array, `allCutPoints` coleta união de todas melts
  - `corruptIntensity=0`: early-return, não aplica XOR
  - `applyBloom` clona `data` do chunk duplicado (`data.slice()`)
- Docs atualizados: `00-MASTER.md` (status, arquitetura com seção de camadas, definição de
  entregue), `MVP-MAP.md` (escopo com ✅, modelo de dados reescrito, tabela de porte atualizada),
  `PLAN.md` (status dos blocos), `DEVLOG.md` (entrada detalhada).

### Fase 14 — Convenção da pilha + dois modos de reprodução (2026-06-20)
- 🔁 **Crítica do usuário:** "no painel de camadas, as camadas devem afetar o que está abaixo
  delas, não acima" e "por que toda mudança exige re-codificar antes da reprodução? esses
  efeitos datamoshing não podem funcionar em tempo real?"
- **Bug de convenção:** o template inicial fazia `project.stack.push(v, m)` (vídeo no topo do
  painel, melt na base) — mas a regra "camada de cima afeta a de baixo" exige Melt ACIMA do
  Vídeo. E a iteração era top-down (índice 0 primeiro), o que significava o oposto do pedido.
- **Correção para pipeline bottom-up:** `renderFrame` e `applyBitstreamLayers` agora iteram
  `for (li = stack.length - 1; li >= 0; li--)`. Cada camada afeta o resultado acumulado do que
  está ABAIXO dela no painel (convenção Photoshop-like).
- **Resposta técnica à pergunta sobre tempo real:** depende do tipo de efeito. Composição
  (src + pixel-fx) é instantânea — pode ser 100% tempo real. Bitstream (melt/bloom/corrupt)
  **exige** encode H.264 primeiro (você não pode corromper bytes de um stream que ainda não
  existe) — intrínseco à técnica, não limitação da implementação.
- **Implementado: dois modos de reprodução:**
  - **Modo 1 (sem bitstream) — TEMPO REAL:** zero encode, render direto no canvas, debounce de
    60ms. Sliders reagem instantaneamente.
  - **Modo 2 (com bitstream) — DECODE PROGRESSIVO:** encode cacheado por assinatura (inclui
    src+pixelfx+regiões+dimensões+fps+cortes); mudar só bitstream params = ~5ms manipulação +
    re-decode. Decode começa a tocar assim que o 1º frame está pronto (~100ms), o resto
    decodifica em background.
- **Cache signature expandida:** agora inclui TODA a composição (src + pixel-fx), não só
  regiões de vídeo. Mudar threshold do pixel-sort invalida o cache corretamente.
- **Export MP4 unificado:** funciona nos dois modos — se houver `lastMoshResult` (Modo 2), usa
  chunks já moshados; senão (Modo 1), codifica na hora do export.

### Fase 15 — Fixes de UX e bugs de interface (2026-06-20, teste real do sistema de camadas)
- Teste real pelo usuário expôs uma série de bugs de UX que não apareciam só lendo o código:
- 🐛 **Bug: arrastar slider arrastava o card.** Causa: `card.draggable=true` no card todo.
  Corrigido: só o handle `.lc-handle` é draggable; `dragstart` listener faz `preventDefault()`
  em qualquer target que não seja o handle.
- 🐛 **Bug: preview piscava preto.** Causa dupla: (1) `renderFrame` faz `clearRect` e depois
  `await getFrameCanvas` — durante o await o canvas visível ficava limpo; (2) re-setar
  `canvas.width/height` (mesmo ao mesmo valor) limpa o canvas. Corrigido: **double-buffering**
  (OffscreenCanvas como back buffer, só copia quando o frame está completo) + só redimensiona
  se as dimensões realmente mudaram.
- 🐛 **Bug: deletar camada deixava o último frame estático.** Causa: `scheduleAutoApply` tinha
  early-return sem parar o preview loop. Corrigido: quando não há fonte, chama `stopPreviewLoop`,
  limpa caches, desenha placeholder, desabilita botões.
- 🐛 **Bug: checkbox "Máscara de recorte" centralizado.** Causa: conflito de CSS entre
  `.lc-controls label` (flex-direction:column, especificidade maior) e `.lc-clip`
  (align-items:center). Corrigido: novo seletor `.lc-controls .lc-clip` (especificidade maior)
  força `flex-direction:row; justify-content:flex-start`.
- ✏️ **Mudanças de UX pedidas pelo usuário:** "Ajustar à próxima fonte" virou checkbox
  persistente (era botão one-shot); painel de camadas inicia vazio (era template Vídeo+Melt);
  label do clip simplificado para "Máscara de recorte".
- **Lição geral:** o teste real pelo usuário revelou bugs que eu não teria encontrado só lendo
  o código — flicker, drag em sliders, frame estático após delete, checkbox centralizado. Isso
  reforça a importância de testar com o usuário, não só com automação.

### Fase 16 — Integração dos efeitos novos e restauração do decoder funcional (2026-06-20)
- Após atingir o limite da sessão anterior, o usuário continuou o `index.html` na z.ai. Essa versão
  acrescentou efeitos bitstream (`reorder`, `smear`, `drop`, `reverse`, `stutter`, `crossmix`,
  `keyinject`, `slicedrop`, `slicehdr`, `mblock`) e pixel-fx adicionais, além do escopo em que o
  bitstream afeta apenas a composição abaixo da camada correspondente. Esses recursos foram mantidos.
- A comparação com o último arquivo funcional revelou que o decoder também havia sido reescrito:
  `decodeChunks()` virou assíncrona, ganhou fallback automático para software/default e a interface
  passou a executar `hwDecodeProbe()` para atribuir falhas genericamente à GPU. Essa combinação
  mascarava erros reais e desviava do caminho empiricamente validado no projeto.
- O decoder foi restaurado sem reverter os efeitos novos: uma única configuração
  `prefer-hardware`, chunks entregues em ordem, yield a cada 16 chunks e captura síncrona de cada
  `VideoFrame` em `OffscreenCanvas`. Não existe mais fallback silencioso nem probe paralelo.
- A mensagem de erro passou a usar o erro efetivo do `VideoDecoder`. Isso separa duas situações:
  regressão de infraestrutura (decoder não inicia) e efeito destrutivo demais (stream inválido),
  especialmente relevante para operações experimentais em slice header/macroblocks.
- A estrutura JavaScript foi validada com `node --check`. O teste automatizado em navegador ficou
  impedido por falha de criação de processo do ambiente Windows, registrada sem transformar essa
  limitação de teste em suposição sobre o computador do usuário.

### Fase 17 — Atualização ao vivo da composição sobre o bitstream (2026-06-20)
- O novo scoping permitiu manter fontes acima do bitstream fora do encode, mas o playback guardava
  `hasAbove` como constante ao iniciar. Mudanças posteriores de blend/opacity não ativavam ou
  reconstruíam a composição; o usuário descobriu que Pause → Play fazia o efeito aparecer.
- `startDecodedPlayback()` passou a reavaliar as camadas acima a cada frame usando a fronteira do
  resultado aplicado. Uma renovação leve e debounced do player preserva o frame corrente e elimina
  a intervenção manual, sem disparar encode/decode nem contrariar a decisão de não haver auto-apply.
- A distinção semântica permanece: composição acima do bitstream é ao vivo; qualquer alteração no
  grupo abaixo, já codificado e moshado, requer o botão Aplicar para produzir novos chunks.

### Fase 18 — Fronteira estável por identidade + blend real para pixel-fx (2026-06-20)
- O teste do usuário com `Databend → Slice hdr → Vídeo` revelou que reavaliar `hasAbove` não bastava:
  o playback ainda recebia o índice numérico salvo no Apply. Como novas camadas entram no índice 0,
  o Slice hdr deslocou de 0 para 1 e o índice armazenado deixou de representar a fronteira real.
- O resultado moshado agora registra o ID da camada bitstream que definiu o escopo. Playback e export
  reencontram essa camada na pilha atual; uma camada nova acima entra imediatamente na composição sem
  invalidar ou recodificar o stream abaixo.
- A revisão encontrou um segundo bug independente: pixel-fx eram gravados por `putImageData`, que
  ignora blend mode e opacity do contexto. O resultado processado agora é composto por `drawImage`
  através de um canvas intermediário, tornando os controles Blend e Opacity funcionais também para
  Pixel sort, RGB shift, Databend, QP boost e Posterize + Dither.

---

## Decisões-chave (resumo)

| # | Decisão | Por quê | Fase |
|---|---|---|---|
| D1 | Datamosh **real** via bitstream (WebCodecs+Mediabunny), sem simulação | Requisito do usuário + tese da Menkman | 1,3 |
| D2 | Foto **e** vídeo | Pedido do usuário; espelha a divisão da bíblia | 1,4 |
| D3 | H.264 baseline + `latencyMode:"realtime"` | Evita B-frames → mosh previsível | 3 |
| D4 | DUAS técnicas de vídeo (remoção de I-frame **+** corrupção de bytes) | Garante glitch em H.264 (a bíblia faz via corrupção) | 4 |
| D5 | Pixel sort = algoritmo **ASDF (Kim Asendorf)** | Referência canônica citada na bíblia | 4 |
| D6 | **Build limpo**, não fork do Loop Lab; zero código morto | Diretriz forte do usuário | 6 |
| D7 | Sem geradores/efeitos artísticos; sem fonte procedural no produto | Ferramenta é só de datamosh | 6 |
| D8 | Resumo da cliente **sem perguntas e sem "2ª etapa"** | Pedido do usuário | 7 |
| D9 | Repositório **público** no GitHub (conta `alexs-master`) | Habilitar sessões na nuvem; escolha do usuário | 9 |
| D10 | Decoder de preview/playback sempre com `hardwareAcceleration:'prefer-hardware'` | Decoder de software aborta bloom/corrupt por validação estrita de `frame_num`; hardware tolera via concealment e revela o glitch real (achado empírico) | 10 |
| D11 | Corrupção de bytes com piso de densidade segura (stride ~15–20) na UI | Densidade maior derruba o decode mesmo em hardware (achado empírico) | 10 |
| D12 | Ingestão de vídeo via **demux Mediabunny** (`Input`+`CanvasSink`), não `<video>`+seek | `<video>` falhava em carregar mesmo MP4 válido nos testes; WebCodecs/Mediabunny decodificou o mesmo arquivo sem problema (achado empírico) | 11 |
| D13 | Frames de preview convertidos em `ImageBitmap` e o `VideoFrame` fechado na hora | Manter `VideoFrame`s abertos p/ tocar em loop esgota o pool de buffers do decoder de hardware e trava o decode sem erro (achado empírico) | 11 |
| D14 | **UI unificada, execução em duas fases** para o sistema de camadas | Pixel-fx (por frame) e bitstream-fx (pós-encode) têm semânticas diferentes; UI unificada preserva a usabilidade, execução em fases preserva a corretude técnica | 13 |
| D15 | **Pipeline bottom-up** (base do painel → topo) — cada camada afeta o que está ABAIXO dela no painel | Convenção Photoshop-like pedida pelo usuário; template inicial estava invertido | 14 |
| D16 | **Dois modos de reprodução** — tempo real (sem bitstream) + decode progressivo (com bitstream) | Composição é instantânea; bitstream exige encode. Modo tempo real elimina espera desnecessária; modo bitstream usa decode progressivo para sensação de tempo real | 14 |
| D17 | **Double-buffering** no preview (OffscreenCanvas back buffer) | `renderFrame` é async (await getFrameCanvas); desenhar direto no canvas visível causava flicker (canvas limpo durante await) | 15 |

---

## Índice de artefatos

| Arquivo | Conteúdo |
|---|---|
| `docs/00-MASTER.md` | Fonte de verdade / estado atual |
| `docs/PROCESSO.md` | **Este** — história completa do processo |
| `docs/DEVLOG.md` | Log curto por etapa |
| `docs/PLAN.md` | Plano time-boxed + pseudocódigo do núcleo |
| `docs/MVP-MAP.md` | Arquitetura 2 etapas, escopo, modelo de dados, UI |
| `docs/RESEARCH.md` | Pesquisa de datamoshing + reuso + fontes |
| `docs/BIBLE-datamoshing-com.md` | Bíblia técnica (datamoshing.com) minerada + mapa |
| `docs/REFS-menkman-glitch-momentum.md` | Teoria (Menkman) + mapa para features |
| `docs/refs_menkman_glitch-momentum.pdf` | PDF original da cliente (local; fora do git) |
| `docs/RESUMO-PARA-CLIENTE.md` (+ `.pdf`) | Resumo técnico de escopo para a cliente |

---

## Política de documentação (padrão do projeto)

Diretriz do usuário (2026-06-18): **documentar TODO o processo.** Na prática:
1. **Toda etapa** (decisão, pesquisa, correção, setup, código) entra no `DEVLOG.md` (curto) e, quando
   relevante para a narrativa/raciocínio, é expandida aqui no `PROCESSO.md`.
2. **Decisões** com seu *porquê* — não só o "o quê".
3. **Comandos/setup reproduzíveis** (ex.: caminho do `gh` portátil, método de PDF) ficam registrados.
4. Atualizar o `00-MASTER.md` quando o estado/próxima-ação mudar.
5. Manter este documento e o DEVLOG em dia a cada avanço da build.
