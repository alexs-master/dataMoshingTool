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
