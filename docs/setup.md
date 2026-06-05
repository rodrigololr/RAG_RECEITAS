# Setup — T001 a T003

Tudo que você precisa rodar pra deixar o ambiente pronto. Sem código no notebook ainda, só terminal.

## Pré-requisitos

- **Python 3.10+** (já tem no venv)
- **Linux/macOS** (Windows: usar WSL2 é mais simples)
- **~5GB livres em disco** (modelos + cache)
- **8GB RAM mínimo** (16GB recomendado p/ Qwen 3B)

Conferir versão:
```bash
python --version
uname -a
```

---

## T001 — Instalar bibliotecas

### Passo 1: ativar o venv

```bash
cd /home/antonio/Documentos/FInaL_de_semana/Rag_receitas
source .venv/bin/activate
```

Você deve ver `(.venv)` no prompt do terminal.

### Passo 2: atualizar pip

```bash
pip install --upgrade pip
```

### Passo 3: instalar libs principais

```bash
pip install chromadb sentence-transformers langchain langchain-community
```

### O que cada lib faz

| Lib | Papel |
|-----|-------|
| `chromadb` | Vector DB local, persiste em pasta |
| `sentence-transformers` | Gerar embeddings (vetores) dos textos |
| `langchain` + `langchain-community` | Orquestrar o pipeline (loaders, splitters, chains) |

### Verificação

```bash
pip list | grep -E "chromadb|sentence-transformers|langchain"
```

Esperado: 5 linhas com nome e versão. Exemplo:
```
chromadb                  0.5.x
llama-cpp-python          0.2.x
pypdf                     4.x.x
sentence-transformers     3.x.x
langchain                 0.3.x
langchain-community       0.3.x
```

### Problemas comuns & soluções

#### ❌ `llama-cpp-python` falha ao compilar

Mensagem típica: `ERROR: Failed building wheel for llama-cpp-python`.

Causa: falta compilador C++ e `cmake`.

**Solução A — instalar deps de build (Linux):**
```bash
sudo apt update
sudo apt install -y cmake build-essential
pip install llama-cpp-python --no-cache-dir
```

**Solução B — wheel pré-compilado (CPU only, mais rápido):**
```bash
pip install llama-cpp-python --extra-index-url https://abetlen.github.io/llama-cpp-python/whl/cpu
```

**Solução C — wheel genérico:**
```bash
pip install llama-cpp-python
```
Se falhar, voltar pra A ou B.

#### ❌ `chromadb` reclama de SQLite

Mensagem típica: `Your system has an unsupported version of sqlite3`.

**Solução:**
```bash
pip install pysqlite3-binary
```

#### ❌ `sentence-transformers` baixa modelo na 1ª execução

Normal. Ele baixa do Hugging Face na 1ª vez (~470MB). Conexão de internet necessária só na 1ª execução.

#### ❌ Versão do Python < 3.10

```bash
python --version  # precisa ser 3.10+
```

Se for menor, criar venv novo:
```bash
deactivate
rm -rf .venv
python3.10 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install llama-cpp-python chromadb sentence-transformers langchain langchain-community pypdf
```

---

## T002 — Baixar LLM (modelo GGUF)

### Modelo escolhido: Qwen2.5-3B-Instruct (Q4_K_M)

**Por quê:**
- 3B parâmetros → roda em CPU razoável (~6GB RAM durante uso)
- Quantização Q4_K_M → equilíbrio qualidade/tamanho (~1.9GB)
- Multilíngue → PT-BR razoável
- Instruct → já calibrado p/ seguir comandos
- Janela de contexto: 32k tokens (mais que suficiente)

### Passo 1: instalar Hugging Face CLI

```bash
pip install huggingface_hub
```

### Passo 2: baixar o modelo

```bash
huggingface-cli download google/gemma-3-gaia-pt-br-4b-it-GGUF \
  gemma-3-gaia-pt-br-4b-it-q4_k_m.gguf \
  --local-dir models/ \
  --local-dir-use-symlinks False
```

**Saída esperada**: barra de progresso + `Download complete`. Demora ~3-10 min dependendo da internet.

### Passo 3: verificar

```bash
ls -lh models/gemma-3-gaia-pt-br-4b-it-q4_k_m.gguf
```

Esperado: arquivo entre 1.8GB e 2.2GB.

### Alternativa: download manual

Se preferir baixar pelo browser:
1. Acessar: https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/tree/main
2. Localizar `gemma-3-gaia-pt-br-4b-it-q4_k_m.gguf`
3. Clicar em "Download"
4. Mover para `Rag_receitas/models/`

### Alternativas (se quiser comparar depois)

| Modelo | Tamanho | RAM p/ rodar | Qualidade PT-BR | Quando usar |
|--------|---------|--------------|-----------------|-------------|
| **Qwen2.5-3B-Instruct (Q4_K_M)** ⭐ | 1.9GB | 6GB | Boa | **Padrão nosso** |
| Phi-3-mini-4k-instruct (Q4) | 2.3GB | 7GB | Muito boa | Se precisar melhor raciocínio |
| Llama-3.1-8B-Instruct (Q4_K_M) | 4.9GB | 10GB | Ótima | Se tiver máquina forte |
| TinyLlama-1.1B (Q4) | 700MB | 3GB | Frágil em PT | Só p/ testes rápidos |
| gemma-3-gaia-pt-br-4b-it-q4_k_m.gguf | 2.3GB | 7GB | Muito boa | Se precisar melhor raciocínio |

Recomendação: **comece com Qwen 3B**. Trocar depois é mudar 1 linha no notebook.

---

## T003 — Baixar modelo de embedding

### Modelo escolhido: paraphrase-multilingual-MiniLM-L12-v2

**Por quê:**
- 471MB, roda rápido
- Suporta 50+ idiomas (PT-BR ok)
- Dimensão: 384
- Bem testado em RAGs pequenos

### Como funciona

Diferente do LLM, o modelo de embedding **não precisa baixar manualmente**. Ele baixa sozinho na 1ª vez que você chamar `SentenceTransformer(...)` no Python.

O cache fica em `~/.cache/huggingface/`.

### Passo 1: pré-baixar (opcional mas recomendado)

```bash
python -c "from sentence_transformers import SentenceTransformer; m = SentenceTransformer('sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2'); print('OK, dim:', m.get_sentence_embedding_dimension())"
```

**Saída esperada**: `OK, dim: 384`. Demora ~30s (download) + 5s (instanciação).

### Passo 2: verificar cache

```bash
ls ~/.cache/huggingface/hub/
```

Esperado: pasta `models--sentence-transformers--paraphrase-multilingual-MiniLM-L12-v2/`.

### Alternativas (qualidade > velocidade)

| Modelo | Tamanho | Dim | PT-BR | Quando usar |
|--------|---------|-----|-------|-------------|
| **MiniLM-L12-multilingual** ⭐ | 471MB | 384 | Bom | **Padrão nosso** |
| `BAAI/bge-m3` | 2.3GB | 1024 | Muito bom | Se retrieve vier fraco |
| `intfloat/multilingual-e5-large` | 1.1GB | 1024 | Muito bom | Meio termo |

Recomendação: **comece com MiniLM**. Trocar = mudar 1 linha.

---

## Checklist final do setup

- [x] `pip list` mostra as 5+ libs instaladas (T001)
- [x] `models/gemma-3-gaia-pt-br-4b-it-q4_k_m.gguf` existe, ~1.9-2.2GB (T002)
- [x] Cache HF tem o MiniLM (T003)
- [x] Estrutura de pastas criada:
  ```
  Rag_receitas/
  ├── .venv/
  ├── agent_receitas.ipynb
  ├── docs/                      ← criado nesta etapa
  ├── models/
  │   └── qwen2.5-3b-instruct-q4_k_m.gguf
  └── data/
      └── receitas/              ← vazio, preenchemos em T004
  ```

Criar pastas:
```bash
mkdir -p models data/receitas
```

---
