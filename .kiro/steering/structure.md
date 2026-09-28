---
inclusion: always
---

# Estrutura do Projeto

## Modelo de repositório

Monorepo com **dois artefatos distribuíveis independentes**, versionados em
conjunto:

1. **Extensão PostgreSQL (`extension/`)** — escrita em Rust com pgrx. É o que o
   usuário instala no Postgres dele. Expõe as funções públicas do schema `ai`
   (ex.: `ai.create_vectorizer`, `ai.ollama_embedding`). É uma **extensão fina**:
   não gera embeddings nem faz chunking, apenas registra o trabalho pendente em
   uma fila dentro do próprio Postgres e expõe a API SQL.
2. **Worker Python (`worker/`)** — pacote Python publicável (PyPI) que roda no
   ambiente do projeto consumidor. Faz o trabalho pesado: observa a fila da
   extensão, gera embeddings via provedores (Ollama, OpenAI) e grava na tabela
   de destino `<origem>_embedding`.

O contrato entre os dois é o **próprio banco**: a extensão grava trabalho pendente
numa fila no schema `ai`, e o worker consome essa fila (LISTEN/NOTIFY ou polling).
O Postgres é sempre a fonte da verdade; o worker é sem estado.

> A ferramenta é feita para ser consumida por **outros projetos**. A fronteira
> pública (API SQL da extensão + pacote do worker) importa mais que a organização
> interna. Mantenha separado o que o usuário final recebe do que é apenas
> ferramenta de desenvolvimento deste repositório.

## Árvore de diretórios

```
postgres_ai_implementation/
├── extension/                    # Artefato 1: extensão Postgres (Rust + pgrx)
│   ├── src/
│   │   ├── lib.rs                # ponto de entrada do pgrx
│   │   ├── vectorizer/           # ai.create_vectorizer, fila, triggers
│   │   ├── providers/            # apenas assinaturas SQL: ai.ollama_*, ai.openai_*
│   │   └── schema/               # DDL do schema `ai`, tabelas de destino/fila
│   ├── sql/                      # DDL/migrations versionados da extensão
│   ├── Cargo.toml
│   └── ai.control                # metadados da extensão
│
├── worker/                       # Artefato 2: worker Python (publicável)
│   ├── src/
│   │   └── postgres_ai/          # pacote importável
│   │       ├── queue/            # leitura da fila (LISTEN/NOTIFY ou polling)
│   │       ├── chunking/         # estratégias de chunk
│   │       ├── embedding/        # geração de embeddings
│   │       ├── providers/        # clientes reais: ollama, openai (plugáveis)
│   │       └── config.py         # configuração via env (modelos Pydantic)
│   └── pyproject.toml            # empacotamento para PyPI
│
├── docker/                       # Artefato 3: distribuição em imagem
│   ├── postgres/                 # imagem Postgres + extensão pré-instalada
│   └── worker/                   # imagem do worker
│
├── tests/
│   ├── unit/                     # testes unitários
│   └── integration/              # testes de integração (extensão + worker + PG real)
│
├── examples/                     # como OUTROS projetos consomem a ferramenta
│   └── rag_basic/                # ex.: montar uma rag_generation_function
│
├── docs/                         # documentação (pt-BR)
├── docker-compose.yml            # ambiente de DESENVOLVIMENTO deste repo
├── .env.example                  # variáveis documentadas, sem valores reais
├── README.md
└── LICENSE
```

## Onde cada tipo de código mora

- **Nova função SQL pública** (`ai.<algo>`): assinatura/registro em
  `extension/src/`, com a DDL correspondente em `extension/sql/`.
- **Novo provedor** (ex.: um terceiro modelo): assinatura SQL em
  `extension/src/providers/` e implementação real do cliente em
  `worker/src/postgres_ai/providers/`. Os dois lados espelham o mesmo provedor.
- **Lógica de embedding/chunking/fila**: sempre no `worker/`, nunca na extensão.
- **Exemplos de consumo**: `examples/` é parte do produto, não rascunho. Servem
  como validação de que a API está simples.
- **Ambiente de dev vs. distribuição**: `docker-compose.yml` da raiz é só para
  desenvolver este repositório; `docker/` contém as imagens entregues ao usuário.

## Convenções de nomenclatura

### Geral

- Identificadores, arquivos e comentários técnicos em **inglês**; documentação e
  mensagens ao usuário final em **português do Brasil**.
- Diretórios sempre em `snake_case` (ou minúsculas simples).

### Funções e schema SQL (extensão)

- Toda função pública fica sob o schema `ai`.
- Funções de provedor seguem `ai.<provedor>_<operação>`
  (ex.: `ai.ollama_embedding`, `ai.openai_vectorizer`).
- Tabela de destino: nome derivado da origem com sufixo `_embedding`
  (ex.: `blogs` → `blogs_embedding`), com o esquema de colunas padrão definido em
  `product.md` (`embedding_uuid` PK, FK para a origem, `chunk_seq`, `chunk`,
  `embedding`).

### Rust (`extension/`)

- Módulos e arquivos em `snake_case` (ex.: `create_vectorizer.rs`).
- Tipos e traits em `PascalCase`; funções e variáveis em `snake_case`.
- Um provedor por módulo dentro de `providers/` (ex.: `providers/ollama.rs`).

### Python (`worker/`)

- Pacote importável: `postgres_ai` (layout `src/` obrigatório para pacote
  publicável, evita import acidental do diretório de trabalho).
- Módulos e arquivos em `snake_case`; classes em `PascalCase`; funções e
  variáveis em `snake_case`.
- Um arquivo por provedor em `providers/` (ex.: `providers/ollama.py`,
  `providers/openai.py`).
- Tipagem obrigatória em toda função pública (parâmetros e retorno).

### Testes

- Espelham a origem em `tests/unit` e `tests/integration`.
- Arquivos de teste Python: prefixo `test_` (ex.: `test_chunking.py`).
- Não criar testes fora desses diretórios sem necessidade explícita.
