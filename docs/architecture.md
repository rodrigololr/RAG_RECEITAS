# Arquitetura

## Visão geral

```
                    ┌─────────────────────────────────────┐
                    │           USUÁRIO                   │
                    │   (pergunta via notebook)           │
                    └──────────────┬──────────────────────┘
                                   │ query
                                   ▼
                    ┌─────────────────────────────────────┐
                    │         PIPELINE RAG                │
                    │  (notebook: agent_receitas.ipynb)   │
                    └──────┬───────────────────┬──────────┘
                           │                   │
                  retrieve │                   │ generate
                           ▼                   ▲
            ┌──────────────────────┐   ┌──────────────────────┐
            │   CHROMA (Vector DB) │   │  LLAMA.CPP (LLM)     │
            │   embeddings + meta  │   │  Gemma3-Gaia-PT-BR   │
            │   persistido em db/  │   │  GGUF em models/     │
            └──────────┬───────────┘   └──────────────────────┘
                       │ indexação
                       ▼
            ┌──────────────────────┐
            │  SENTENCE-TRANSF.    │
            │  modelo MiniLM PT-BR │
            └──────────┬───────────┘
                       │ embeddings
                       ▼
            ┌──────────────────────┐
            │   CHUNKS estruturados│
            │   (receita + seções) │
            └──────────┬───────────┘
                       │ origina de
                       ▼
            ┌──────────────────────┐
            │   data/receitas/     │
            │   PDF, MD, TXT       │
            └──────────────────────┘
```

## Componentes

| Componente | Arquivo/Pasta | Papel |
|------------|---------------|-------|
| **Documentos** | `data/receitas/` | Receitas-origem (PDF/MD/TXT) |
| **Loader** | notebook | Lê arquivos, extrai texto |
| **Chunker** | notebook | Divide por receita + seções (ingredientes, preparo) |
| **Embedder** | `sentence-transformers` (MiniLM) | Texto → vetor 384-dim |
| **Vector DB** | `Chroma` em `db/` | Armazena vetores + metadata |
| **Retriever** | notebook | Query → top-K chunks similares |
| **LLM** | `llama-cpp` com Qwen2.5-3B | Gera resposta final |
| **Orquestrador** | notebook (LangChain LCEL) | Cola tudo |

## Fluxo de dados (duas fases)

### Fase 1 — Indexação (offline, roda 1x ou ao adicionar receita)

```
PDF/MD/TXT  →  texto puro  →  chunks estruturados  →  vetores  →  Chroma
   data/        loader          chunker              embedder     db/
```

### Fase 2 — Query (online, cada pergunta)

```
pergunta  →  embedding  →  top-K chunks  →  prompt  →  LLM  →  resposta
             (query)        (Chroma)       (monta)   (llama.cpp)
```

## Modelo de dados de um chunk

Cada chunk salvo no Chroma tem:

| Campo | Tipo | Exemplo |
|-------|------|---------|
| `id` | string | `receita-001-ingredientes` |
| `document` | string (texto) | "2 xícaras de farinha, 3 ovos..." |
| `embedding` | list[float] (384) | `[0.012, -0.034, ...]` |
| `metadata` | dict | `{"receita": "Bolo de fubá", "secao": "ingredientes", "tempo_min": 45}` |

## Metadados que vamos usar (filtros futuros)

```python
metadata = {
    "receita_id":   "bolo-fuba-001",
    "nome":         "Bolo de fubá",
    "categoria":    "doce",                 # doce | salgado | fitness | vegano
    "tempo_min":    45,
    "porcoes":      8,
    "dificuldade":  "facil",                # facil | medio | dificil
    "tags":         ["milho", "cafe-da-tarde"],
    "secao":        "ingredientes",         # ingredientes | preparo | macros | dicas
    "fonte":        "data/receitas/bolo-fuba.md"
}
```

Isso permite filtros tipo: *"receitas veganas com tempo < 30min"*.

## Princípios de design

1. **Local-first**: tudo roda offline após o setup. Sem chamada de API externa.
2. **Chunking estruturado**: respeita a natureza das receitas (1 receita = 1 unidade lógica).
3. **Modelo trocável**: se Qwen não servir, troca por Phi-3 ou Llama-3.1 sem reescrever pipeline.
4. **Crescimento incremental**: começa com 5 receitas, escala até 50 sem mudar arquitetura.

## Diagrama de pastas final

```
Rag_receitas/
├── .venv/                                  # ambiente Python
├── agent_receitas.ipynb                    # onde tudo roda
├── docs/                                   # este diretório
│   ├── README.md
│   ├── architecture.md
│   ├── pipeline.md
│   ├── setup.md
│   ├── tasks.md
│   ├── decisions.md
│   └── glossary.md
├── data/
│   └── receitas/                           # suas receitas (PDF/MD/TXT)
│       ├── bolo-de-fuba.md
│       ├── frango-grelhado.md
│       └── ...
├── models/
│   └── qwen2.5-3b-instruct-q4_k_m.gguf    # LLM local (~2GB)
└── db/                                     # Chroma persiste aqui (criado auto)
```
