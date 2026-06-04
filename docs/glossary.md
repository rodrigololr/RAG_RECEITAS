# Glossário

Termos técnicos que aparecem na doc. Ordem alfabética.

---

## Alucinação

Quando o LLM "inventa" uma resposta que parece plausível mas não tem base nos dados. RAG reduz alucinação ancorando respostas no contexto recuperado.

**Exemplo**: perguntar "qual a origem do bolo de fubá?" sem ter essa info nas receitas → LLM pode inventar uma história. Com RAG, ele responde "não encontrei nas minhas receitas".

---

## Chunk (ou "trecho")

Pedaço de texto em que dividimos um documento. Em RAG, quanto menores os chunks, mais preciso o retrieve, mas menos contexto cada um carrega.

**Estratégia nossa**: chunking **estruturado por receita** (não por tamanho fixo). 1 receita gera 2-4 chunks (macros, ingredientes, preparo, dicas).

Ver [pipeline.md](pipeline.md#etapa-2--chunk-dividir).

---

## Chunker (ou Text Splitter)

Função que divide texto em chunks. Em LangChain: `RecursiveCharacterTextSplitter`. No nosso caso: parser customizado por receita.

---

## Cosine similarity (similaridade cosseno)

Métrica que mede o quanto 2 vetores apontam na mesma direção. Varia de -1 a 1 (na prática embeddings ficam entre 0 e 1).

- **1.0** = vetores idênticos
- **0.0** = ortogonais (sem relação)
- **-1.0** = opostos

É a métrica padrão p/ comparar embeddings.

---

## Embedding (vetorização, encodificação)

Representação numérica de um texto. Uma lista de floats (ex: 384 números) que captura o **significado** do texto.

Textos com significado similar → embeddings próximos no espaço vetorial.

**Exemplo**:
- "frango grelhado" → `[0.012, -0.034, 0.087, ...]`
- "peito de frango assado" → `[0.015, -0.029, 0.091, ...]`  (perto do anterior)
- "bolo de chocolate" → `[-0.123, 0.234, -0.156, ...]`  (longe)

O modelo que faz isso é o **embedder** (sentence-transformers no nosso caso).

---

## Embedding model (modelo de embedding)

Rede neural que transforma texto em vetor. Ex: `paraphrase-multilingual-MiniLM-L12-v2`.

Escolhido por: idioma (PT-BR), tamanho (471MB), dimensão (384), velocidade.

---

## GGUF

Formato de arquivo binário p/ modelos de LLM. Substituiu o antigo GGML. Suportado por `llama.cpp` e `llama-cpp-python`.

Vantagens:
- Quantização embutida (Q4_K_M, Q5_K_M, Q8_0)
- Auto-contido (1 arquivo = modelo inteiro)
- Carregamento rápido

---

## Hallucination

Ver **alucinação**.

---

## Hugging Face

Plataforma onde a maioria dos modelos open-source fica hospedada. Acesse em https://huggingface.co.

Usamos o `huggingface-cli` p/ baixar o GGUF do Qwen2.5.

---

## Instruct model (modelo instruction-tuned)

LLM ajustado p/ seguir instruções (em vez de só completar texto). Ex: Qwen2.5-**Instruct**, Llama-3.1-**Instruct**.

Diferença prática:
- **Base model** (sem instruct): completa o prompt. "O céu é" → "azul. As estrelas..."
- **Instruct model**: segue comando. "Explique o céu" → resposta explicativa.

Sempre prefira versão Instruct p/ RAG.

---

## LLM (Large Language Model)

Modelo de linguagem grande. É a "inteligência" que gera texto. No nosso caso: **Qwen2.5-3B-Instruct** rodando via **llama.cpp**.

---

## llama.cpp / llama-cpp-python

- **llama.cpp**: engine em C++ p/ rodar LLMs locais, criado por Georgi Gerganov.
- **llama-cpp-python**: bindings Python p/ llama.cpp. É o que vamos usar no notebook.

Vantagem: roda em CPU, sem precisar GPU.

---

## Metadata (metadados)

Dados **sobre** os dados. No nosso caso, cada chunk tem metadata descrevendo de qual receita/seção veio, tempo de preparo, etc.

**Uso**: filtrar retrieve por critérios. "Só receitas com tempo < 30min" = `where={"tempo_min": {"$lt": 30}}`.

---

## Prompt

Texto que enviamos p/ o LLM. Em RAG, montamos o prompt = system message + contexto recuperado + pergunta do usuário.

---

## Quantization (quantização)

Técnica de **comprimir modelo** p/ ocupar menos memória/disco, com perda mínima de qualidade.

| Sufixo | Bits/param | Tamanho Qwen 3B | Qualidade |
|--------|-----------|------------------|-----------|
| F16 | 16 | ~6GB | 100% |
| Q8_0 | 8 | ~3.2GB | ~99% |
| **Q4_K_M** ⭐ | ~4.5 | **~1.9GB** | ~97% |
| Q2_K | 2-3 | ~1.2GB | ~85% |

Escolhemos **Q4_K_M** pelo equilíbrio.

---

## RAG (Retrieval-Augmented Generation)

Arquitetura onde o LLM gera respostas **aumentadas** por informação recuperada de uma base.

```
query → retrieve (busca) → contexto + prompt → LLM → resposta
```

Reduz alucinação, mantém o LLM atualizado com seus dados, e não precisa retreinar.

---

## Recall / Precision

Métricas de qualidade do retrieve:

- **Recall**: dos documentos relevantes, quantos recuperei? (alto = bom)
- **Precision**: dos que recuperei, quantos são relevantes? (alto = bom)

Trade-off: aumentar `top_k` → recall sobe, precision desce.

---

## Reranker

Modelo (geralmente cross-encoder) que recebe top-K chunks do retrieve e re-ordena por relevância.

**Quando usar**: quando o retrieve "quase acerta" mas põe o chunk certo na posição 4-5 em vez de 1.

**Custo**: +1 modelo, +1 inferência por query. P/ 10-50 docs não precisamos.

---

## Retriever (ou Vector Store Retriever)

Componente que faz a busca por similaridade. No nosso caso: `Chroma.as_retriever(k=3)`.

---

## Sentence-Transformers

Biblioteca Python da UKPLab que facilita usar modelos de embedding. Carrega do Hugging Face, expõe API simples `.encode()`.

---

## Similarity (similaridade)

Ver **cosine similarity**.

---

## Top-K

Parâmetro do retrieve: quantos chunks retornar. Default nosso: **3**.

Mais que isso = mais ruído no prompt. Menos = pode perder contexto.

---

## Vector DB (banco vetorial)

Banco especializado em armazenar e buscar vetores por similaridade. Em vez de buscar por valor exato (como SQL), busca por **proximidade**.

Exemplos: Chroma, FAISS, Qdrant, Weaviate, Pinecone, LanceDB.

---

## Vetor (vector)

Lista de números. No nosso caso, lista de 384 floats que representa um texto.

Também chamado de: embedding, representation, encoding.

---

## Janela de contexto (context window)

Quantos tokens o LLM consegue "ver" de uma vez (input + output somados).

Qwen2.5-3B-Instruct: 32k tokens. Mais que suficiente p/ nosso uso típico (prompt ~2k + resposta ~400 = ~2.4k).
