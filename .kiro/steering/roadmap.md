---
inclusion: manual
---

# Roadmap de Implementação

Sequência de implementação do projeto, da fundação até a distribuição. Cada fase
vira uma ou mais specs. A ordem é **horizontal (por camada)**: primeiro o schema
onde o dado vive, depois quem escreve nele (extensão), depois quem lê dele
(worker), e por fim o empacotamento.

**Princípio ordenador:** o banco é o contrato. Uma camada só começa quando a
camada da qual ela depende está estável. Isso evita retrabalho nos dois artefatos
(extensão e worker), já que ambos dependem do formato definido no banco.

> Este arquivo é referência de planejamento. Puxe-o com `#` ao criar ou executar
> uma spec para saber em que fase o trabalho se encaixa e o que precisa existir
> antes.

## Estado atual

> Atualize esta seção conforme as fases avançam.

## Fases

### Fase 0 — Fundações do repositório

Esqueleto que as steerings `structure` e `tech` descrevem. Nada compila ou testa
sem isso.

- Estrutura de diretórios: `extension/`, `worker/src/postgres_ai/`, `docker/`,
  `tests/unit/`, `tests/integration/`, `examples/`.
- `docker-compose.yml` de desenvolvimento (Postgres + `pgvector`).
- `.env.example` com variáveis documentadas, sem valores reais.
- `worker/pyproject.toml` (empacotamento do pacote `postgres_ai`, layout `src/`).
- `extension/Cargo.toml` e `extension/ai.control` (metadados da extensão).
- CI mínima executando lint e testes.

**Critério de pronto:** `docker compose up -d` sobe o Postgres; `pytest` e
`ruff check .` rodam (mesmo sem testes de negócio ainda); a extensão compila com
pgrx.

### Fase 1 — Schema `ai` e fila (o contrato)

Peça central: o contrato entre extensão e worker. Definir mal aqui força
retrabalho nos dois lados.

- DDL do schema `ai`.
- Tabela de fila de trabalho pendente no schema `ai`.
- Formato da tabela de destino `<origem>_embedding`: `embedding_uuid` (PK),
  FK para a origem, `chunk_seq`, `chunk` (texto), `embedding` (vector).

**Critério de pronto:** DDL versionada em `extension/sql/`; formato da fila e da
tabela de destino documentado e estável.

### Fase 2 — `ai.create_vectorizer` (extensão, escrita)

Com o schema pronto, a extensão passa a produzir trabalho na fila. Ainda sem
consumidor, mas já testável.

- Função `ai.create_vectorizer(origem::regclass, destination => ...)`.
- Trigger na tabela de origem que enfileira trabalho a cada insert.
- Criação automática da tabela de destino `<origem>_embedding`.

**Critério de pronto:** inserir numa tabela de origem registra trabalho na fila
(verificável em teste de integração); a tabela de destino é criada com o esquema
padrão.

### Fase 3 — Worker consumindo a fila (worker, leitura)

Fecha o ciclo. Ao fim desta fase existe um caminho ponta a ponta funcionando com
Ollama.

- Leitura da fila (polling primeiro; LISTEN/NOTIFY só se necessário —
  simplicidade acima de completude).
- Estratégia de chunking.
- Geração de embedding e gravação na tabela de destino.
- Primeiro provedor real: **Ollama** (local, sem credencial para testar).
- Assinaturas SQL espelhadas `ai.ollama_embedding` / `ai.ollama_vectorizer` na
  extensão.

**Critério de pronto:** inserir numa tabela de origem resulta em embeddings
gravados na tabela de destino, ponta a ponta, via Ollama.

### Fase 4 — Segundo provedor (OpenAI)

Só faz sentido depois que o mecanismo genérico funciona com um provedor. Valida
que a arquitetura de provedores plugáveis está boa.

- Assinaturas SQL `ai.openai_embedding` / `ai.openai_vectorizer` na extensão.
- Cliente `worker/src/postgres_ai/providers/openai.py`.

**Critério de pronto:** o mesmo fluxo ponta a ponta funciona trocando o provedor
para OpenAI, sem alterar a mecânica de fila/chunking.

### Fase 5 — Empacotamento e distribuição

Empacota algo que já funciona.

- Imagem Docker do Postgres com a extensão pré-instalada (`docker/postgres/`).
- Imagem do worker (`docker/worker/`).
- Exemplo de consumo em `examples/rag_basic/` (validação de que a API pública
  está simples).

**Critério de pronto:** um projeto consumidor sobe as imagens e monta um fluxo de
RAG seguindo o exemplo, sem tocar no código interno da ferramenta.

## Como fatiar em specs

- Uma spec por fase é o padrão. Fases grandes (ex.: Fase 3) podem virar mais de
  uma spec se necessário.
- Antes de abrir uma spec, confirme que o critério de pronto da fase anterior foi
  atingido — a ordem horizontal existe justamente para isso.
- Provedores são espelhados: toda spec que adiciona ou altera um provedor toca os
  dois lados (assinatura SQL na extensão + cliente no worker).
