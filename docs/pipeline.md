# Pipeline RAG — passo a passo

## O que é RAG (revisão curta)

**RAG** = Retrieval-Augmented Generation.

LLM sozinho "inventa" do que aprendeu no treino. RAG **busca** trechos relevantes na sua base e **injeta** no prompt, antes do LLM gerar a resposta.

Resultado: respostas ancoradas nos seus dados, sem alucinar.

## O pipeline em 7 etapas

```
┌──────────────────────────────────────────────────────────────────┐
│ FASE 1 — INDEXAÇÃO (offline, roda 1x)                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [1] LOAD         carrega PDF/MD/TXT  →  string de texto         │
│       ↓                                                           │
│  [2] CHUNK        divide texto        →  lista de chunks + meta  │
│       ↓                                                           │
│  [3] EMBED        texto               →  vetor 384-dim            │
│       ↓                                                           │
│  [4] STORE        vetor + meta        →  Chroma (persiste em db/)│
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│ FASE 2 — QUERY (online, cada pergunta)                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [5] RETRIEVE     pergunta + embed    →  top-K chunks            │
│       ↓                                                           │
│  [6] PROMPT       contexto + pergunta →  prompt completo          │
│       ↓                                                           │
│  [7] GENERATE     prompt              →  LLM  →  resposta        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## Etapa 1 — Load (carregar)

**O que faz**: ler arquivos da pasta `data/receitas/` e extrair texto.

**Entradas**: `*.pdf`, `*.md`, `*.txt`
**Saídas**: lista de `(nome_arquivo, texto_completo)`

**Bibliotecas**:
- MD/TXT: leitura direta (open padrão) ou `UnstructuredMarkdownLoader`
- PDF: `pypdf` ou `PyPDFLoader` (langchain)
- DOCX: `docx2txt` (se precisar)

**Pseudocódigo**:
```
para cada arquivo em data/receitas/:
    texto = ler(arquivo)
    documentos.append({nome: arquivo, texto: texto})
```

## Etapa 2 — Chunk (dividir)

**O que faz**: dividir o texto em pedaços menores. Diferente de RAG genérico, aqui usamos **chunking estruturado por receita** (não por 500 chars cegos).

**Por quê estruturado?**
- Receita tem seções: macros, ingredientes, preparo, dicas
- Chunk cego pode quebrar no meio de "3 ovos, 1 xícara de..." → contexto perdido
- Chunk estruturado mantém a seção inteira = retrieval mais preciso

**Estratégia (escolhida)**:
```
1 receita = 1 chunk "macros"  (nome, descrição, tempo, porção)
          + 1 chunk "ingredientes"
          + 1 chunk "preparo"
          + 0..N chunks "dicas"
```

**Metadados obrigatórios em cada chunk**:
```python
{
    "receita_id":  string único,
    "nome":        string,
    "categoria":   "doce" | "salgado" | "fitness" | "vegano" | ...,
    "tempo_min":   int,
    "porcoes":     int,
    "dificuldade": "facil" | "medio" | "dificil",
    "tags":        lista de strings,
    "secao":       "macros" | "ingredientes" | "preparo" | "dicas",
    "fonte":       caminho do arquivo
}
```

**Pseudocódigo**:
```
para cada documento:
    receita = parse_receita(texto)  # separa seções por regex/markdown
    chunks.append(chunk(receita.macros,  metadata={...secao: "macros"}))
    chunks.append(chunk(receita.ingredientes, metadata={...secao: "ingredientes"}))
    chunks.append(chunk(receita.preparo, metadata={...secao: "preparo"}))
    se receita.tem_dicas:
        chunks.append(chunk(receita.dicas, metadata={...secao: "dicas"}))
```

**Tamanho típico**:
- Macros: ~100-300 chars
- Ingredientes: ~200-800 chars
- Preparo: ~500-3000 chars (pode ser maior; LLM aguenta)
- Dicas: ~200-500 chars

## Etapa 3 — Embed (vetorizar)

**O que faz**: transforma texto em vetor numérico (lista de floats).

**Modelo**: `paraphrase-multilingual-MiniLM-L12-v2`
- Dimensão: 384
- Tamanho: ~471MB
- Idiomas: 50+ (PT-BR ok)

**O que é o vetor?**
- Lista de 384 números floats
- Textos semanticamente similares → vetores "próximos" no espaço
- "frango grelhado" ≈ "peito de frango assado" (vetores próximos)
- "frango grelhado" ≠ "bolo de chocolate" (vetores distantes)

**Pseudocódigo**:
```
model = carregar_modelo_embedding("paraphrase-multilingual-MiniLM-L12-v2")
para cada chunk:
    vetor = model.encode(chunk.texto)
    chunk.embedding = vetor  # shape (384,)
```

## Etapa 4 — Store (armazenar)

**O que faz**: salvar vetores + metadados no Vector DB.

**Vector DB**: `Chroma` (persiste em `db/`)

**Operações**:
- `collection.add(...)` — adiciona chunks (com id, embedding, documento, metadata)
- `collection.query(...)` — busca por similaridade
- `collection.get(...)` — recupera por id/filtro

**Persiste em disco**: ao fechar/reabrir o notebook, dados continuam lá.

**Pseudocódigo**:
```
client = chromadb.PersistentClient(path="db/")
collection = client.get_or_create_collection("receitas")
collection.add(
    ids       = [chunk.id for chunk in chunks],
    embeddings= [chunk.embedding for chunk in chunks],
    documents = [chunk.texto for chunk in chunks],
    metadatas = [chunk.metadata for chunk in chunks]
)
```

## Etapa 5 — Retrieve (buscar)

**O que faz**: dada uma pergunta, achar os K chunks mais relevantes.

**Algoritmo**:
1. Embed da pergunta (mesmo modelo da etapa 3)
2. Calcular similaridade cosseno entre vetor-pergunta e todos os vetores
3. Retornar top-K com maior similaridade

**Filtros opcionais** (via metadata):
- `where={"categoria": "vegano"}`
- `where={"tempo_min": {"$lt": 30}}`
- `where={"tags": {"$contains": "frango"}}`

**Parâmetros chave**:
- `k`: quantos chunks retornar (default 3-5)
- `score_threshold`: corte mínimo de similaridade (0.0 a 1.0)

**Pseudocódigo**:
```
pergunta = "o que faço com frango?"
vetor_query = model.encode(pergunta)
resultados = collection.query(
    query_embeddings=[vetor_query],
    n_results=3,
    where={"categoria": {"$ne": "doce"}}  # opcional
)
chunks = resultados["documents"][0]
metas  = resultados["metadatas"][0]
scores = resultados["distances"][0]
```

## Etapa 6 — Prompt (montar)

**O que faz**: criar o prompt que vai pro LLM.

**Estrutura**:
```
<system>
Você é um assistente de receitas. Use SOMENTE o contexto abaixo.
Se a resposta não estiver no contexto, diga "não encontrei nas minhas receitas".
Responda em português brasileiro.
</system>

<contexto>
[1] Receita: Bolo de fubá
    Ingredientes: 2 xícaras de fubá, 3 ovos, 1 xícara de açúcar...

[2] Receita: Frango grelhado
    Ingredientes: 500g peito de frango, sal, limão...
</contexto>

<user>
O que faço com frango?
</user>
```

**Por que "use SOMENTE o contexto"?**
- Reduz alucinação
- Força o modelo a ancorar a resposta
- Se não souber, fala que não sabe (em vez de inventar)

## Etapa 7 — Generate (gerar)

**O que faz**: passar o prompt pro LLM e receber resposta.

**LLM**: `Qwen2.5-3B-Instruct` (GGUF) via `llama-cpp-python`

**Parâmetros de geração**:
- `max_tokens`: 512 (tamanho máx da resposta)
- `temperature`: 0.3 (respostas mais determinísticas; RAG não quer criatividade)
- `top_p`: 0.9
- `stop`: ["<|im_end|>", "User:"] (critérios de parada)

**Pseudocódigo**:
```
llm = Llama(
    model_path="models/qwen2.5-3b-instruct-q4_k_m.gguf",
    n_ctx=4096,        # janela de contexto
    n_threads=8        # CPU threads
)
resposta = llm.create_chat_completion(
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user",   "content": user_prompt}
    ],
    temperature=0.3,
    max_tokens=512
)
print(resposta["choices"][0]["message"]["content"])
```

## Quando rodar cada fase?

| Fase | Quando | Custo |
|------|--------|-------|
| **1 (indexação)** | 1x. Re-rodar só ao adicionar receita nova. | Embed N chunks (~5-30s p/ 50 receitas) |
| **2 (query)** | Toda pergunta. | Embed 1 query (instant) + LLM generate (5-30s) |

## Trade-offs e tuning

| Parâmetro | Efeito | Default nosso |
|-----------|--------|---------------|
| `chunk_size` ↑ | Chunks maiores = mais contexto, menos precisão | N/A (estruturado) |
| `chunk_overlap` | Ajuda em chunk cego, irrelevante p/ estruturado | N/A |
| `top_k` ↑ | Mais contexto p/ LLM, mas mais ruído | 3-5 |
| `temperature` ↑ | Mais criativo (ruim p/ RAG) | 0.2-0.4 |
| `embedding model` | Melhor modelo = retrieve mais preciso | MiniLM → upgrade BGE se precisar |

## Próximas otimizações (depois do MVP)

- **Reranker**: modelo que re-ordena top-K por relevância (ex: `cross-encoder`)
- **HyDE**: gerar doc hipotético antes de buscar (melhora recall)
- **Query expansion**: gerar variações da pergunta antes de buscar
- **Multi-query**: buscar 3x com ângulos diferentes e mesclar
- **Chat memory**: lembrar de turnos anteriores (conversational RAG)

Por ora, MVP foca em **fazer funcionar bem o pipeline básico**.
