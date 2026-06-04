# Tasks — Lista completa

13 tarefas, dividas em 3 fases. Cada uma tem: objetivo, comandos (alto nível), verificação.

## Como usar este doc

1. Encontre a task atual
2. Leia a seção dela inteira
3. Execute os passos
4. Marque ✅ quando terminar
5. Me reporte → eu libero a próxima

Critério "pronto": ✅ só quando a **verificação** passar.

---

## Fase 0 — Setup (T001 a T003)

Detalhes completos em [setup.md](setup.md). Resumo aqui.

### T001 — Instalar bibliotecas

**O que**: instalar `llama-cpp-python`, `chromadb`, `sentence-transformers`, `langchain`, `pypdf`.

**Comandos**:
```bash
cd /home/antonio/Documentos/FInaL_de_semana/Rag_receitas
source .venv/bin/activate
pip install --upgrade pip
pip install llama-cpp-python chromadb sentence-transformers langchain langchain-community pypdf
```

**Verificação**:
```bash
pip list | grep -E "llama-cpp-python|chromadb|sentence-transformers|langchain|pypdf"
```
Esperado: 5+ linhas.

**Status**: ⬜ pendente

---

### T002 — Baixar LLM (Qwen2.5-3B-Instruct)

**O que**: baixar `qwen2.5-3b-instruct-q4_k_m.gguf` (~1.9GB) para `models/`.

**Comandos**:
```bash
pip install huggingface_hub
huggingface-cli download Qwen/Qwen2.5-3B-Instruct-GGUF \
  qwen2.5-3b-instruct-q4_k_m.gguf \
  --local-dir models/ \
  --local-dir-use-symlinks False
```

**Verificação**:
```bash
ls -lh models/qwen2.5-3b-instruct-q4_k_m.gguf
```
Esperado: arquivo 1.8-2.2GB.

**Status**: ⬜ pendente

---

### T003 — Baixar modelo de embedding (MiniLM)

**O que**: pré-baixar `paraphrase-multilingual-MiniLM-L12-v2` (~470MB).

**Comandos**:
```bash
python -c "from sentence_transformers import SentenceTransformer; m = SentenceTransformer('sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2'); print('OK, dim:', m.get_sentence_embedding_dimension())"
```

**Verificação**: imprime `OK, dim: 384`.

**Status**: ⬜ pendente

---

## Fase 1 — Dados & Indexação (T004 a T008)

### T004 — Criar 3-5 receitas-exemplo

**O que**: colocar 3-5 receitas em MD dentro de `data/receitas/`. Eu te mando template + 2 exemplos preenchidos.

**Entrega**: `data/receitas/*.md` (mínimo 3 arquivos).

**Verificação**:
```bash
ls data/receitas/
```
Esperado: 3+ arquivos `.md`.

**Status**: ⬜ pendente

---

### T005 — Testar loaders (ler arquivos)

**O que**: no notebook, célula que lê cada arquivo de `data/receitas/` e imprime o texto extraído.

**O que esperar no print**:
- Texto da receita em PT-BR
- Sem código estranho (quebras de linha preservadas, sem "binary garbage")

**Verificação**: print do primeiro chunk de cada receita.

**Status**: ⬜ pendente

---

### T006 — Testar chunking estruturado

**O que**: célula que pega o texto e divide em chunks (macros, ingredientes, preparo, dicas), cada um com metadata.

**O que esperar**:
- Lista de dicts: `{texto, metadata}`
- Cada receita gera 2-4 chunks
- Metadata tem `nome`, `secao`, `tempo_min`, etc.

**Verificação**: print dos chunks da primeira receita.

**Status**: ⬜ pendente

---

### T007 — Testar embedding (vetorizar)

**O que**: célula que pega 1 chunk e transforma em vetor 384-dim.

**O que esperar**:
- Print do shape: `(384,)` ou `(1, 384)`
- Print dos 5 primeiros valores do vetor

**Verificação**: vetor gerado, shape correto.

**Status**: ⬜ pendente

---

### T008 — Persistir no Chroma

**O que**: célula que salva todos os chunks (vetores + metadata) na coleção `receitas` do Chroma em `db/`.

**O que esperar**:
- Confirmação de N chunks adicionados
- Path `db/` criado com arquivos do Chroma

**Verificação**:
```bash
ls -la db/
```
Esperado: arquivos `.parquet`, `.sqlite3`, etc.

**Status**: ⬜ pendente

---

## Fase 2 — Retrieval & LLM (T009 a T013)

### T009 — Testar retrieve (busca)

**O que**: célula com query-exemplo → top-3 chunks similares.

**Query de teste**: "o que faço com frango?"

**O que esperar**:
- Print dos 3 chunks retornados
- Score/distância de cada um
- Metadata visível (nome da receita, seção)

**Verificação**: chunks retornados contêm palavra/frasco relacionado a frango.

**Status**: ⬜ pendente

---

### T010 — Carregar LLM (smoke test sem RAG)

**O que**: célula que carrega Qwen2.5 via `llama-cpp` e faz 1 pergunta simples.

**Pergunta de teste**: "Olá, quem é você?"

**O que esperar**:
- Resposta em PT-BR
- Tempo de geração: 5-30s (CPU)

**Verificação**: resposta coerente.

**Status**: ⬜ pendente

---

### T011 — Montar pipeline RAG completo

**O que**: encadear retrieve → prompt → generate em uma função. Query completa.

**Query de teste**: "o que faço com frango?"

**O que esperar**:
- Resposta menciona receita(ns) da base
- Resposta cita ingredientes ou passos das receitas recuperadas

**Verificação**: resposta coerente + baseada nas suas receitas.

**Status**: ⬜ pendente

---

### T012 — Validar UAT (2 queries reais)

**O que**: rodar 2 queries de validação.

**Queries**:
1. "**o que faço com frango?**" → deve retornar ≥1 receita com frango
2. "**posso substituir leite por leite de coco no bolo de fubá?**" → deve responder baseado na receita recuperada

**O que esperar**:
- Q1: lista de receitas que contêm frango
- Q2: resposta sobre substituição, citando a receita de origem

**Verificação**: ambas funcionam, respostas em PT-BR, ancoradas nas suas receitas.

**Status**: ⬜ pendente

---

### T013 — Documentar decisões finais

**O que**: eu atualizo `docs/decisions.md` com aprendizados reais do build (modelos, parâmetros, problemas).

**Entrega**: `docs/decisions.md` atualizado com seção "Decisões pós-build".

**Verificação**: arquivo existe, seção preenchida.

**Status**: ⬜ pendente

---

## Critérios de aceitação finais (UAT)

O sistema está pronto quando:

- [ ] Carrega 5+ receitas e responde perguntas sobre elas
- [ ] "o que faço com frango?" → retorna receitas com frango
- [ ] "posso substituir X por Y na receita Z?" → responde baseado na receita
- [ ] Respostas em PT-BR
- [ ] Roda 100% offline após setup
- [ ] Documentação em `docs/` reflete o que foi feito

---

## Status geral

| Fase | Tarefas | Concluídas |
|------|---------|-----------|
| 0 — Setup | T001-T003 | 0/3 |
| 1 — Dados & Indexação | T004-T008 | 0/5 |
| 2 — Retrieval & LLM | T009-T013 | 0/5 |
| **Total** | **T001-T013** | **0/13** |

## Reportar progresso

A cada task: me diga ✅/❌ + comando/erro. Eu libero a próxima.
