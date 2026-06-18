# 📖 A BÍBLIA — datamoshing.com (Sterling Crispin / "Phil"), minerado do arquivo

> A cliente indicou **datamoshing.com** como a referência que usou ("foi a bíblia").
> ℹ️ **O site ESTÁ no ar** (confirmado por print da cliente, 2026-06). Correção: meu fetch via
> `curl` do ambiente sandbox recebia só a página placeholder "IIS Windows Server" (703 bytes) /
> 404 — provavelmente bloqueio de CDN para IPs de datacenter. Então minerei o conteúdo idêntico
> do **Wayback Machine** (snapshots ~2016–2021); confere com o print da cliente. Mantemos a cópia
> aqui como backup offline da fonte canônica.
>
> Como ler: cada técnica vem com **como ela mapeia para a NOSSA ferramenta** (browser/WebCodecs).

---

## Catálogo completo de tutoriais do site

**Vídeo:**
1. How to datamosh videos *(Avidemux — I-frame removal + P-frame duplication/Bloom)* ← o principal
2. How to datamosh videos with data corruption *(hex edit do átomo `mdat` em H.264)*
3. How to datamosh videos with automation *(AutoHotkey repetindo remoção de I-frames)*

**Imagem:**
4. How to glitch images using pixel sorting *(Processing — ASDF Pixel Sort do Kim Asendorf)*
5. How to glitch JPG images with data corruption *(hex edit de JPG)*
6. How to glitch images using RGB channel shifting
7. How to glitch images with WordPad
8. How to glitch images using Processing scripts
9. How to glitch images using audio editing software

> O site cobre **fotos E vídeos** — exatamente o nosso escopo.

---

## 1. Vídeo — I-frame removal (método canônico, Avidemux)

Conceito (palavras do autor): I-frame = imagem completa; P/B-frame = só diferenças.
"If an I-frame is corrupted, removed or replaced the data contained in the following
P-frames is applied to the wrong picture." Remover I-frames → movimento da cena nova
aplicado na imagem da cena anterior. Duplicar P-frames → **Bloom** (movimento acentuado).

Passo a passo do autor:
- Usar **Avidemux 2.5.6** (versões novas "corrigem" o glitch — não servem).
- Video: trocar **Copy → MPEG-4 ASP (Xvid)**.
- Configure → aba **Frame** → **Maximum I-frame Interval: 300 → 99999999**. Salvar com novo nome.
- Reabrir, trocar Video de volta para **Copy** (não re-encoda; só edita e salva).
- Slider mostra `Frame Type: I (00)` / `P (00)`. ↑/↓ pula entre I-frames.
- **Manter o 1º frame como I** (pra começar a tocar). Selecionar um I-frame: `mark A` →
  `→` (próximo frame) → `mark B` → seleção é só aquele I-frame → **Delete**.
- Repetir indo só **pra frente** (voltar trava o Avidemux). Salvar.

**→ MAPA p/ nossa tool:** é exatamente o nosso fluxo `keyFrame:true` só no frame 0,
GOP longo, e descarte de chunks `type==='key'`. "Max I-frame Interval 99999999" = nosso
"1 keyframe + resto delta". A escolha **MPEG-4 ASP (não H.264)** confirma nosso RISCO #4:
decoders H.264 tendem a limpar o glitch via deblocking/concealment (ver §2 e §6).

---

## 2. Vídeo — data corruption (hex edit do `mdat`, em H.264!)

Esta técnica **usa H.264 em MP4/MOV** (ao contrário da #1). Pontos do autor:
- Sempre editar **uma cópia**, nunca o original.
- MP4/MOV = estrutura de **átomos**. `ftypqt` aparece no 5º byte (assinatura).
- O átomo **`mdat`** ("media data") contém os bytes brutos de frame+áudio.
  Procurar a string `mdat` no arquivo. Dentro do `mdat` os bytes parecem **aleatórios**;
  os outros átomos são **estruturados**.
- **Corromper bytes SÓ dentro do `mdat`** → distorção visual. Mexer nos átomos estruturados
  (timing, índices) → arquivo injogável. (mdat → chunks → NAL units → slices.)

**→ MAPA p/ nossa tool (IMPORTANTE):** isto é **mais alinhado com a nossa stack** do que o
método Avidemux/MPEG-4. No nosso pipeline WebCodecs, os bytes do `mdat` **são exatamente
os `chunk.data`** que capturamos. Corromper bytes de chunks **delta** (preservando os `key`
e a estrutura do container, que o Mediabunny remuxa) = a versão programática e segura desta
técnica. E como a própria bíblia faz mosh em **H.264 via corrupção**, isso **valida que o
caminho H.264 funciona** — só que por corrupção, não (só) por remoção de I-frame.

---

## 3. Vídeo — automação (AutoHotkey)

Macro que repete a remoção de I-frames no Avidemux (↑ = próximo I-frame; `[`/`]` = mark A/B;
`→`; `Delete`), em loop de N vezes. Irrelevante p/ browser, mas confirma o **padrão de
operação em lote** ("remover os próximos N I-frames") que viramos um parâmetro na UI.

---

## 4. Imagem — pixel sorting (ASDF Pixel Sort, Kim Asendorf)

O autor usa o **ASDF Pixel Sort do Kim Asendorf** (Processing) como script de referência.
Conceito: isolar uma linha (horizontal/vertical) de pixels e **reordenar posições** por
critério — **luminosidade, matiz (hue) ou saturação**. Faz isso em **trechos** delimitados
por **thresholds** (começa a ordenar quando passa de um limiar de luma/hue/sat e para quando
cai abaixo), varrendo colunas e linhas.

**→ MAPA p/ nossa tool:** É **O** algoritmo de referência do pixel-sort. Reimplementar em JS
sobre `ImageData` (o Loop Lab já tem o scaffolding `getImageData` nos `EFFECTS`). Parâmetros:
direção (col/linha), critério (luma/hue/sat), threshold de início/fim, sentido. Determinístico
com nossa seed. É a espinha da entrada de **imagem**.

---

## 5. Imagem — JPG data corruption / RGB channel shifting / WordPad / audio

- **JPG corruption:** hex-editar bytes do JPG (depois do header) → blocos coloridos/deslocados.
  Mapa: ler bytes do arquivo, corromper região de scan, re-decodificar via `Image`/canvas.
- **RGB channel shifting:** deslocar canais de cor → franjas cromáticas. Mapa: trivial em
  `ImageData` (offset por canal). Bom efeito barato e real.
- **WordPad / audio editing:** databending abrindo a imagem como texto/áudio e re-salvando.
  Mapa: nicho; opcional. (Equivale a corromper bytes com "regras" de outro formato.)

---

## 6. Conclusões que REFINAM nosso plano

1. **datamoshing.com está no ar** (acessível pela cliente); temos cópia offline minerada aqui como backup.
2. **Duas técnicas de vídeo, não uma:** (a) **I-frame removal** e (b) **byte-corruption do mdat/delta**.
   Implementar AS DUAS. A corrupção é o que garante glitch em **H.264** (a bíblia faz assim).
3. **Risco H.264 confirmado e resolvido:** se a remoção de I-frame ficar "limpa" demais por
   deblocking, a corrupção de bytes de delta entrega o visual sujo — sem trocar de codec.
   MPEG-4 ASP/AVI via ffmpeg.wasm continua como plano C, agora menos provável de ser preciso.
4. **Pixel sort = ASDF Pixel Sort (Kim Asendorf)** como referência canônica do lado imagem.
5. O escopo "fotos E vídeos" do nosso projeto **espelha exatamente** a divisão Vídeo/Imagem da bíblia.

---

## Fontes (todas via Wayback; domínio vivo está parqueado/offline)

- How to datamosh videos: `http://datamoshing.com/2016/06/26/how-to-datamosh-videos/`
- Data corruption (vídeo): `http://datamoshing.com/2016/06/17/how-to-datamosh-videos-with-data-corruption/`
- Automation: `http://datamoshing.com/2017/02/12/how-to-datamosh-videos-with-automation/`
- Pixel sorting: `http://datamoshing.com/2016/06/16/how-to-glitch-images-using-pixel-sorting/`
- JPG corruption: `http://datamoshing.com/2016/06/15/how-to-glitch-jpg-images-with-data-corruption/`
- RGB channel shifting: `http://datamoshing.com/2016/06/29/how-to-glitch-images-using-rgb-channel-shifting/`
- Site vivo (cliente acessa direto). Backup via `https://web.archive.org/web/2021id_/<url-acima>` (curl -A Mozilla; do nosso sandbox o domínio direto retornou placeholder IIS)
- ASDF Pixel Sort (Kim Asendorf): GitHub `kimasendorf/ASDFPixelSort`
