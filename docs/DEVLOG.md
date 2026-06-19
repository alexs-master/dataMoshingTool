# 📓 DEVLOG — Datamosh Tool

> Log de progresso. **Atualizar a cada etapa concluída** (curto: o que foi feito, decisões,
> gotchas, e o próximo passo). Serve para retomar sem perder o fio em caso de perda de contexto.
> Formato: entrada mais recente no TOPO.

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
