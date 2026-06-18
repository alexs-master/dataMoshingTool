# 📓 DEVLOG — Datamosh Tool

> Log de progresso. **Atualizar a cada etapa concluída** (curto: o que foi feito, decisões,
> gotchas, e o próximo passo). Serve para retomar sem perder o fio em caso de perda de contexto.
> Formato: entrada mais recente no TOPO.

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
