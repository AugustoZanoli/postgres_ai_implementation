# postgres_ai_implementation

Extensão de IA para PostgreSQL que facilita RAG, vetorização e embeddings direto no
banco, usando modelos locais (Ollama) ou OpenAI. É uma alternativa mantida ao PGAI,
que foi arquivado e deixou tutoriais e documentação obsoletos.

A ideia é reduzir o código repetitivo: você registra uma tabela para ser vetorizada
e a ferramenta cuida de gerar e manter os embeddings.

```sql
CREATE TABLE blogs (
    id INT PRIMARY KEY,
    text TEXT NOT NULL
);

SELECT ai.create_vectorizer(
    'blogs'::regclass,
    destination => 'blogs_embedding'
);
```

## Como funciona

O projeto é um monorepo com dois artefatos independentes que se comunicam pelo
próprio Postgres:

- **`extension/`** — extensão fina em Rust (pgrx). Expõe a API SQL sob o schema `ai`
  e registra o trabalho pendente numa fila. Não gera embeddings.
- **`worker/`** — pacote Python que consome a fila, gera os embeddings via provedores
  e grava na tabela de destino.

O Postgres é sempre a fonte da verdade; o worker é sem estado.

## Começando

Requer Docker e Docker Compose.

```bash
# Ambiente de desenvolvimento local
docker compose up -d

# Testes
pytest

# Lint e formatação (antes de commitar)
ruff check . && ruff format .
```

Copie `.env.example` para configurar as variáveis de ambiente.

## Documentação

- Exemplos de consumo por outros projetos: `examples/`
- Documentação (pt-BR): `docs/`
- Convenções de código, estrutura e visão de produto: `.kiro/steering/`

## Convenções

### Commits

- `[bug-fix]`: Correção de bug;
- `[config]`: Criação ou alteração de configurações;
- `[db]`: Criação de migrations ou alterações de banco;
- `[docs]`: Criação ou alteração de documentações;
- `[feat]`: Criação de feature;
- `[fix]`: Correção de minors;
- `[sdd]`: Criação ou alteração de documentos de Steering, Specs e auxiliares para IA's;
