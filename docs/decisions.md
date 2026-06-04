# Decisões de stack

Registro do que escolhemos e por quê. Serve pra lembrar quando quiser trocar algo.

## Requisitos do projeto

| Requisito | Escolha |
|-----------|---------|
| Volume de receitas | 10-50 |
| Fonte de dados | Arquivos locais (PDF/MD/TXT) |
| LLM | Local (sem custo, privado) |
| Interface | Jupyter Notebook |
| Internet após setup | Não (deve rodar offline) |

## Decisões por camada

### 1. LLM (modelo de linguagem)

**Escolhido**: `Gemma-3-Gaia-PT-BR` (Q4_K_M, GGUF)
[text](../../../../.cache/huggingface/hub/models--cnmoro--Gemma-3-Gaia-PT-BR-4b-it-Q4_K_M-GGUF/snapshots/aeaaac6f89eb9439b692accf0cab2683c80d9d84/gemma-3-gaia-pt-br-4b-it-q4_k_m.gguf)

**Por quê:**
- **Local-first**: roda 100% offline, sem chave de API
- **Tamanho**: 2.5GB, cabe em máquinas modestas
- **PT-BR aceitável**: modelo multilíngue da google
- **GGUF**: formato moderno suportado por `llama.cpp` e `llama-cpp-python`

**Trade-offs aceitos:**
- 4B é pequeno. Respostas criativas/longas podem ser fracas.
- Para receitas (respostas curtas, factuais), 4B basta.

**Quando trocar:**
- Se respostas saírem ruins (alucinações) → subir o modelo
- Se precisar de raciocínio mais forte (ex: "converta receita para 3 pessoas")

---

### 2. Embedding (modelo de vetorização)

**Escolhido**: `paraphrase-multilingual-MiniLM-L12-v2` (sentence-transformers)

**Por quê:**
- **Multilíngue**: 50+ idiomas, PT-BR ok
- **Leve**: 471MB, inferência rápida em CPU
- **Dimensão 384**: bom p/ Chroma local, queries rápidas
- **Maduro**: muito usado em produção, bem documentado
- **Sentence similarity**: especializado em "esses 2 textos querem dizer a mesma coisa?"

**Trade-offs aceitos:**
- Não é nativo PT-BR (multilíngue). Para receitas, performance é suficiente.
- Vetor 384-dim é menor que alternativas (768, 1024). Pode perder nuance em queries complexas.

**Alternativas consideradas:**

| Modelo | Por que não |
|--------|-------------|
| **BAAI/bge-m3** | 2.3GB, melhor em PT mas pesado. Upgrade futuro |
| **intfloat/multilingual-e5-large** | 1.1GB, meio termo |
| **BERTimbau (portuguese-bert)** | Nativo PT, mas não otimizado p/ similarity |
| **OpenAI text-embedding-3-small** | Excelente, mas viola "local-first" |

**Quando trocar:**
- Se retrieve vier fraco (chunks errados nas top-3) → `bge-m3` (qualidade > velocidade)
- Se o modelo demorar muito (>2s por query) → manter MiniLM

---

### 3. Vector DB

**Escolhido**: `Chroma` (persistência local)

**Por quê:**
- **Zero-config**: cria pasta, começa a usar
- **Persistência nativa**: salva em DuckDB + Parquet
- **API simples**: `add()`, `query()`, `get()`
- **Filtros por metadata**: suporta `where={...}`
- **Leve p/ 10-50 docs**: ideal pro nosso volume

**Trade-offs aceitos:**
- Não escala bem acima de ~100k chunks (não é nosso caso)
- Single-node apenas (não é nosso caso)
- Sem réplicas (não é nosso caso)

**Alternativas consideradas:**

| Vector DB | Por que não |
|-----------|-------------|
| **FAISS** | Rápido mas low-level. Precisaria wrapper |
| **LanceDB** | Bom, mas comunidade menor |
| **Qdrant / Weaviate / Pinecone** | Overkill, exigem Docker ou cloud |
| **Postgres + pgvector** | Pesado p/ um side-project |
| **In-memory (numpy)** | Não persiste, perde dados ao fechar notebook |

**Quando trocar:**
- Se passar de ~500 receitas → migrar p/ Qdrant ou Postgres+pgvector
- Se precisar de alta concorrência → Pinecone / Weaviate

---

### 4. Framework de orquestração

**Escolhido**: `LangChain` (LCEL — LangChain Expression Language)

**Por quê:**
- **Didático**: chains legíveis, fácil de entender o pipeline
- **LCEL**: sintaxe declarativa (`prompt | llm | output_parser`)
- **Integrações prontas**: loaders (PDF, MD), splitters, retrievers
- **Comunidade ativa**: fácil achar ajuda

**Trade-offs aceitos:**
- LangChain tem churn alto (breaking changes entre versões)
- Abstração adiciona overhead (camada a mais)

**Alternativa considerada**: `LlamaIndex`
- Mais focado em RAG puro, melhor abstração p/ indexação
- Comunidade menor em PT
- Decide usar LangChain pela didática

**Quando trocar:**
- Se precisar de chains mais complexas (multi-agent) → LlamaIndex ou Haystack
- Se quiser controle total → escrever pipeline na mão (sem framework)

---

### 5. Loader de documentos

**Escolhido**: `pypdf` (PDF) + leitura direta (MD/TXT)

**Por quê:**
- **MD/TXT**: Python nativo (`open()`). Sem dependência.
- **PDF**: `pypdf` é leve, sem compilação nativa

**Alternativa**: `unstructured` (lib mais robusta, lida com tabelas/imagens)
- Overhead desnecessário p/ receitas (são texto)

---

### 6. Formato de chunking

**Escolhido**: estruturado por receita (macros + seções)

**Por quê:**
- Receitas têm **estrutura semântica** (ingredientes ≠ preparo)
- Chunk cego (500 chars) quebra a receita no meio
- Estruturado → retrieve mais preciso, metadata rica

**Trade-offs aceitos:**
- Implementação própria (regex/Markdown parser) em vez de splitter pronto
- Mais código que `RecursiveCharacterTextSplitter`

**Alternativa**: `RecursiveCharacterTextSplitter` (LangChain)
- Genérico, bom p/ texto sem estrutura
- Não aproveita a natureza das receitas

**Schema do chunk:** ver [architecture.md](architecture.md#modelo-de-dados-de-um-chunk).

---

## Parâmetros de operação (defaults)

| Parâmetro | Valor | Por quê |
|-----------|-------|---------|
| `n_ctx` (LLM) | 4096 | Prompt + 3-5 chunks típicos < 2k tokens |
| `n_threads` (LLM) | 8 | Boa p/ CPU 8+ cores. Ajustar p/ seu hardware |
| `temperature` (LLM) | 0.3 | RAG quer determinismo, não criatividade |
| `max_tokens` (LLM) | 512 | Resposta típica de receita cabe em 200-400 tokens |
| `top_p` (LLM) | 0.9 | Nucleus sampling padrão |
| `top_k` (retrieve) | 3 | 3-5 chunks é sweet-spot p/ RAG pequeno |
| `chunk_size` (chunks) | N/A (estruturado) | — |
| `chunk_overlap` (chunks) | N/A (estruturado) | — |

---

## Decisões pós-build

> Esta seção será preenchida em T013, após rodar o pipeline e ver o que funciona/não funciona na prática.

(Data de preenchimento: ___________)

- **Modelo de embedding se comportou bem?** Sim/Não
- **LLM respondeu coerentemente?** Sim/Não
- **Necessário trocar algum modelo?** Qual?
- **Problemas de performance?** Quais?
- **Ajustes aplicados?**
