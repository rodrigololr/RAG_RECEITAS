# Documentação — RAG de Receitas

Guia completo do projeto, organizado pra você seguir task por task.

## Índice

| Doc | Conteúdo | Status |
|-----|----------|--------|
| [architecture.md](architecture.md) | Visão geral do sistema, componentes, fluxo de dados | T001-T003 |
| [pipeline.md](pipeline.md) | Como RAG funciona, cada etapa do pipeline | T005-T011 |
| [setup.md](setup.md) | Instalar libs, baixar modelos (T001-T003) | **comece aqui** |
| [tasks.md](tasks.md) | Lista completa de tarefas T001-T013 com critérios | sempre |
| [decisions.md](decisions.md) | Stack escolhida e por quê (LLM, embed, vector DB) | referência |
| [glossary.md](glossary.md) | Termos técnicos (embedding, chunk, cosine sim...) | referência |

## Ordem de leitura sugerida

1. **[decisions.md](decisions.md)** — entender o que escolhemos e por quê
2. **[architecture.md](architecture.md)** — ver o sistema inteiro
3. **[setup.md](setup.md)** — executar T001-T003 (setup)
4. **[tasks.md](tasks.md)** — guia de T004 em diante
5. **[pipeline.md](pipeline.md)** — consultar conforme avança
6. **[glossary.md](glossary.md)** — consultar quando travar num termo

## Estado atual

- [x] Greenfield. Venv criado. Notebook vazio.
- [x] T001 — instalar libs
- [x] T002 — baixar LLM
- [ ] T003 — baixar embedding
- [ ] T004-T013 — pipeline + UAT

## Comandos rápidos

```bash
# Ativar ambiente
cd /home/antonio/Documentos/FInaL_de_semana/Rag_receitas
source .venv/bin/activate

# Abrir notebook
jupyter notebook agent_receitas.ipynb
```

## Reportar progresso

A cada task concluída, me diga: ✅ ou ❌ + erros (se houver).
Aí eu libero a próxima.
