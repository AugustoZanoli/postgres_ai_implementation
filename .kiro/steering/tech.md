---
inclusion: always
---

# Stack Técnica

## Stack principal

- **Linguagem de aplicação**: Python 3.14
- **Linguagem do banco/extensão**: Rust 1.98.1
- **Banco de dados**: PostgreSQL com a extensão `pgvector`
- **Infraestrutura**: Docker / Docker Compose

## Bibliotecas chave

- **Pydantic**: validação e modelagem de dados
- **Pytest**: testes unitários e de integração
- **pgvector**: armazenamento e busca de vetores/embeddings

## Comandos comuns

```bash
# Subir ambiente local
docker compose up -d

# Rodar testes
pytest

# Lint e formatação (rodar antes de commitar)
ruff check . && ruff format .
```

## Convenções de código

- **Tipagem obrigatória** em toda função pública (parâmetros e retorno).
- **Idioma**: identificadores, código e comentários técnicos em inglês; mensagens
  destinadas ao usuário final em português do Brasil.
- **Tratamento de erros**: use exceções de domínio específicas. Nunca capture com
  `except:` genérico nem silencie exceções.
- **Simplicidade acima de completude**: prefira a API mais simples que resolva o
  caso de uso a abstrações genéricas e complexas.
- **SQL / extensão**: funções públicas da extensão ficam sob o schema `ai` e
  seguem o padrão `ai.<provedor>_<operação>` (ex.: `ai.ollama_embedding`).

## Testes

- Testes unitários em `tests/unit`; testes de integração em `tests/integration`.
- Toda nova regra de negócio exige teste cobrindo o comportamento.
- Não adicione testes fora desses diretórios sem necessidade explícita.

## Ambiente e CI

- Variáveis de ambiente documentadas em `.env.example`, sempre sem valores reais.
- Nunca comite segredos ou credenciais reais.
- A CI executa lint e testes em cada PR; garanta que ambos passem localmente antes
  de abrir o PR.
