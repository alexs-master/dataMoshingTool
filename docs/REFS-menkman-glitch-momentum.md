# 🎛️ Rosa Menkman — "The Glitch Moment(um)" (Network Notebook #04, INC, 2011)

> PDF que a cliente enviou (`refs_menkman_glitch-momentum.pdf`, ~25k palavras).
> É o texto **teórico** fundador da glitch art. Junto com a bíblia técnica (datamoshing.com),
> mostra o que a cliente valoriza: **glitch REAL, não filtro decorativo.** Este é o NORTE
> conceitual do projeto e provavelmente o critério de avaliação dela.

---

## ⭐ A tese central (= justifica nosso requisito inquebrável)

Menkman define glitch como "a (actual and/or simulated) break from an expected or conventional
flow of information [...] that results in a perceived accident or error." Ela **admite** que
existe glitch simulado — mas a crítica dela é que o glitch-como-filtro perde o "moment(um)"
crítico e vira só **moda/gênero**:

- *"Every form of glitch, whether breaking a flow or designed to look like it breaks a flow,
  will eventually become a new fashion."*
- *"What is now a glitch will become a fashion."*
- *"Glitch transitions between artifact and filter — between radical breakages and
  commodification processes."*

→ **Para a tool:** fazer o glitch **de verdade** (mexer no encode/decode real) é o que mantém o
"momentum" — exatamente o que o usuário exigiu (nada de overlay/CSS). Não é purismo: é o ponto
da obra. Nosso pipeline WebCodecs (corromper o bitstream real, decoder errando) está do lado
"artifact/real", não do lado "filter".

### Linhas do *Glitch Studies Manifesto* (dentro do livro) — úteis p/ voz/UI/copy
- *"The dominant, continuing search for a noiseless channel has been — and will always be —
  no more than a regrettable, ill-fated dogma."*
- *"Fight genres, interfaces and expectations!"*
- *"Surf the vortex of technology, the in-between, the art of artifacts!"*
- *"Become a nomad of noise artifacts!"*

---

## 🗺️ O framework dela → mapa para FEATURES da tool

Menkman (via Shannon/Weaver) diz que a transmissão de informação pode ser interrompida em
**três ocasiões** — e cada uma é uma família de efeitos da nossa ferramenta:

| Ocasião (Menkman) | O que é | Feature na tool |
|---|---|---|
| **Encoding/Decoding (compression artifacts)** | Erros de de/compressão (codec, DCT, macroblocos, quantização) | **Datamosh de vídeo** (I-frame removal, P-dup, corrupção de delta) + **JPG/DCT glitch** na imagem |
| **Feedback** | Loops recursivos de sinal | (Opcional/bônus) **feedback/echo** de frames — o Loop Lab já tem efeitos de feedback reusáveis |
| **Glitch (quebra inesperada)** | Corrupção/databending direto dos bytes | **Corrupção de bytes** (mdat/delta no vídeo; scan do JPG; RGB shift na imagem) |

Ela cita explicitamente **datamoshing** como estudo de caso de "glitch cultivation", e liga o
artefato ao **DCT / macroblocos / quantização** do codec — exatamente o substrato que
exploramos. Termos dela que viram vocabulário da UI: *noise artifact, compression artifact,
encoding/decoding, macroblock, feedback, databending, lossy/lossless, container vs codec.*

---

## 🏷️ Banco de nomes (para presets / modos / UI — opcional, alinha com a cliente)

Do vocabulário Menkman: **Moment(um)** · **Noise Artifact** · **Macroblock Bloom** ·
**Compression Artifact** · **Vortex** · **Nomad** · **Ill-fated Channel** · **Encoded/Decoded** ·
**Lossy** · **Databend**. (Combinam com os "melt/bloom" da bíblia técnica.)

---

## 📚 Sumário do livro (pra saber o que tem dentro, se precisarmos citar)

Glitch Studies Manifesto · A Technological Approach To Noise · Linear Progression and the Myth of
Perfect Transmission · Noise Artifacts · Encoding/Decoding: Compression Artifacts · A Vernacular
of File Formats · Orderly Chaos: Feedback Artifacts · The Other Noise Artifact: Glitch · The
Perception of Glitch · The Meaning of Noise · The Glitch Moment(um): A Void in Techno-Culture ·
Technorealism and the Accident of Art · A Phenomenology of Glitch Art.

Leituras correlatas que ela cita (se a cliente quiser fundo teórico): *Glitch Studies Manifesto*
(2010), *A Vernacular of File Formats* (2010, handbook prático de compressão/glitch), Shannon &
Weaver (teoria da informação), Iman Moradi (*Glitch Aesthetics*, 2004).

---

## ✅ Como isso muda a build (concreto)

1. **Confirma a direção:** glitch real, não filtro. Já é o nosso plano — agora com respaldo teórico
   citável na entrega (bom p/ defesa do trabalho com a cliente).
2. **Sugere enquadrar as features em 3 famílias** (compressão / feedback / corrupção) na UI —
   dá coerência conceitual e cobre tanto vídeo quanto imagem.
3. **Dá vocabulário/voz** para nomear modos e presets, alinhando o produto à linhagem glitch-art
   que a cliente referencia (não é só técnico, é cultural).
4. Nada disso muda o cronograma de sexta; é camada de enquadramento/curadoria por cima do núcleo técnico.
