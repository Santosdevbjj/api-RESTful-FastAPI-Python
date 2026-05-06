# 🏋️‍♂️ WorkoutAPI — REST Assíncrona para Gestão de Atletas de CrossFit

![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-brightgreen?logo=fastapi)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-blue?logo=postgresql)
![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)
![CI](https://github.com/Santosdevbjj/api-RESTful-FastAPI-Python/actions/workflows/ci.yml/badge.svg)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)

> API RESTful assíncrona para gerenciamento de atletas, categorias e centros de treinamento de CrossFit.  
> Arquitetura em camadas (Repository + Service + Schema), migrations versionadas, ambientes dev/prod isolados e pipeline CI/CD com GitHub Actions.

---

## 🧭 Índice

- [1. Problema de Negócio](#1-problema-de-negócio)
- [2. Contexto](#2-contexto)
- [3. Premissas da Solução](#3-premissas-da-solução)
- [4. Estratégia da Solução](#4-estratégia-da-solução)
- [5. Technical Insights](#5-technical-insights)
- [6. Resultados e Capacidades da API](#6-resultados-e-capacidades-da-api)
- [7. Como Executar o Projeto](#7-como-executar-o-projeto)
- [8. Referência de Endpoints](#8-referência-de-endpoints)
- [9. Estrutura do Projeto](#9-estrutura-do-projeto)
- [10. Pipeline CI/CD](#10-pipeline-cicd)
- [11. Aprendizados](#11-aprendizados)
- [12. Próximos Passos](#12-próximos-passos)
- [Autor](#autor)

---

## 1. Problema de Negócio

Competições de CrossFit envolvem múltiplos atletas distribuídos em categorias e centros de treinamento. Gerenciar esse cadastro sem uma API estruturada gera três problemas recorrentes:

- **Duplicidade de dados**: sem controle de integridade no CPF, o mesmo atleta pode ser cadastrado múltiplas vezes, corrompendo rankings e relatórios de competição.
- **Inconsistência de relacionamentos**: atletas associados a categorias ou centros inexistentes geram falhas silenciosas em sistemas que consomem esses dados downstream.
- **Falta de rastreabilidade**: sem migrations versionadas, alterações de schema em produção tornam-se operações manuais e não auditáveis.

O objetivo deste projeto é entregar uma API que **previne esses problemas na camada de infraestrutura**, liberando as camadas de negócio para focar em regras de domínio.

---

## 2. Contexto

Este projeto foi desenvolvido no **Bootcamp Luizalabs — Back-end com Python**, com escopo expandido para demonstrar padrões de produção além do exigido pelo bootcamp.

O domínio modela três entidades centrais de uma plataforma de CrossFit:

- **Atleta**: entidade principal, identificada unicamente por CPF, associada a uma categoria e um centro de treinamento.
- **Categoria**: agrupa atletas por nível ou peso (ex: Elite, Rx, Scaled).
- **Centro de Treinamento**: local físico onde o atleta treina — necessário para logística de competição.

A decisão de usar FastAPI com I/O assíncrono reflete o cenário real: plataformas de esporte recebem picos de requisições no momento de abertura de inscrições, onde latência e throughput são críticos.

---

## 3. Premissas da Solução

- O CPF é a chave de unicidade do atleta — violações de integridade devem retornar mensagem clara com o valor duplicado, não um erro genérico 500.
- Ambientes de desenvolvimento e produção têm requisitos diferentes: dev utiliza `uvicorn --reload` com volume montado; prod utiliza `gunicorn` com 4 workers e `restart: unless-stopped`.
- Migrations devem ser executadas automaticamente no startup do container, garantindo que o schema esteja sempre sincronizado sem intervenção manual.
- O pipeline CI deve reproduzir o ambiente de produção usando Docker Compose, não mocks ou SQLite em memória.

---

## 4. Estratégia da Solução

A arquitetura foi organizada em camadas com responsabilidades bem definidas:

```
Request → Router → Service → Repository → Model (SQLAlchemy) → PostgreSQL
                ↓
            Schema (Pydantic) — validação de entrada e serialização de saída
                ↓
         Exception Handler — trata IntegrityError antes de virar HTTP 500
```

**Camadas e decisões:**

| Camada | Responsabilidade | Decisão técnica |
|--------|-----------------|-----------------|
| **Router** (`app/api/v1/`) | Recebe HTTP, valida query params, chama Service | FastAPI `APIRouter` com prefix e tags |
| **Service** (`app/services/`) | Orquestra regras de negócio | Isola lógica do repository para facilitar testes |
| **Repository** (`app/repositories/`) | Acesso ao banco | `selectinload` para carregar relacionamentos em uma única query assíncrona |
| **Schema** (`app/schemas/`) | Contrato da API | Pydantic v2 com `from_attributes=True` para serialização ORM |
| **Model** (`app/models/`) | Mapeamento ORM | SQLAlchemy 2.0 com `ForeignKey` e `relationship` bidirecionais |
| **Migrations** (`alembic/`) | Versionamento de schema | `async_engine_from_config` — migrations rodam no mesmo engine assíncrono da app |
| **Exception Handler** | Tratamento de integridade | Extrai o CPF duplicado via regex na mensagem do PostgreSQL e retorna HTTP 303 |

**Ambientes Docker:**

| Arquivo | Propósito |
|---------|-----------|
| `docker-compose.yml` | Base compartilhada (db + healthcheck) |
| `docker-compose.override.yml` | Dev: volume montado + uvicorn reload |
| `docker-compose.prod.yml` | Prod: gunicorn 4 workers + restart policy |

---

## 5. Technical Insights

**Por que `asyncpg` em vez de `psycopg2`?**  
`psycopg2` é síncrono e bloqueia a event loop do Python quando executa queries. Com `asyncpg` + `SQLAlchemy async`, a API mantém throughput alto sob carga concorrente — essencial em picos de inscrição de competições.

**Por que o `IntegrityError` retorna HTTP 303 e não 409?**  
O handler captura a exceção antes que o FastAPI a transforme em 500, extrai o CPF duplicado da mensagem interna do PostgreSQL via regex (`Key (cpf)=(valor) already exists`) e retorna uma resposta semântica. HTTP 303 foi a escolha do bootcamp para indicar que o recurso já existe em outro local — em produção enterprise, 409 Conflict seria mais aderente ao padrão REST.

**Por que `selectinload` em vez de `joinedload`?**  
`joinedload` gera um `JOIN` que pode multiplicar linhas quando há múltiplos relacionamentos. `selectinload` emite queries `SELECT ... WHERE id IN (...)` separadas, mais previsíveis em termos de cardinalidade e mais eficientes para listas paginadas.

**Por que três arquivos Docker Compose em vez de um?**  
O padrão de override do Docker Compose permite que `docker-compose.override.yml` seja aplicado automaticamente em dev (sem flag `-f`) enquanto prod exige a composição explícita `docker-compose -f docker-compose.yml -f docker-compose.prod.yml`. Isso evita configurações de desenvolvimento vazando para produção.

**Makefile como interface de operações:**  
O Makefile encapsula todos os comandos complexos (`make dev-up`, `make migrate-up`, `make deploy`) eliminando a dependência de conhecimento dos flags do Docker Compose por parte de quem executa o projeto.

---

## 6. Resultados e Capacidades da API

A API entrega as seguintes capacidades em ambiente containerizado:

- **Cadastro de atletas com validação de unicidade**: CPF duplicado retorna mensagem descritiva em vez de erro genérico, reduzindo tempo de debug em integrações.
- **Listagem paginada com filtros**: endpoint `GET /v1/atletas/` aceita filtros por `nome` (contains) e `cpf` (exato) com paginação `limit/offset`, adequado para interfaces de backoffice com grandes volumes.
- **Relacionamentos carregados sem N+1**: a query de listagem carrega categoria e centro de treinamento em uma única roundtrip ao banco via `selectinload`.
- **Pipeline CI reproduzível**: o workflow do GitHub Actions sobe o ambiente completo via Docker Compose, roda migrations, lint e testes — o mesmo fluxo que qualquer developer executaria localmente com `make deploy-check`.
- **Ambientes isolados**: `make dev-up` e `make prod-up` são operações distintas e independentes, eliminando o risco de configuração de desenvolvimento em produção.

---

## 7. Como Executar o Projeto

### Pré-requisitos

- [Docker](https://www.docker.com/) e [Docker Compose](https://docs.docker.com/compose/)
- [Git](https://git-scm.com/)
- (Opcional) Python 3.11+ para execução local sem Docker

### Ambiente de Desenvolvimento

**1. Clonar o repositório**
```bash
git clone https://github.com/Santosdevbjj/api-RESTful-FastAPI-Python.git
cd api-RESTful-FastAPI-Python
```

**2. Configurar variáveis de ambiente**
```bash
cp .env.example .env
# Edite .env conforme necessário (as defaults já funcionam para dev local)
```

**3. Subir o ambiente completo**
```bash
make dev-up
# Equivale a: docker-compose up --build
# Inclui: build da imagem, start do PostgreSQL, healthcheck, migrations automáticas e uvicorn --reload
```

**4. Acessar a documentação interativa**

Acesse 👉 [http://localhost:8000/docs](http://localhost:8000/docs) — Swagger UI gerado automaticamente pelo FastAPI.

### Ambiente de Produção

```bash
make prod-up
# Equivale a: docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
# Gunicorn com 4 UvicornWorkers, restart: unless-stopped, sem volume montado
```

### Execução Local (sem Docker)

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
# Configure DATABASE_URL no .env apontando para um PostgreSQL local
alembic upgrade head
uvicorn app.main:app --reload
```

### Comandos Úteis (Makefile)

| Comando | Ação |
|---------|------|
| `make dev-up` | Sobe ambiente de desenvolvimento |
| `make prod-up` | Sobe ambiente de produção |
| `make test` | Executa Pytest no container |
| `make lint` | Roda flake8 + black check |
| `make migrate-up` | Aplica migrations pendentes |
| `make migrate-create d="nome"` | Cria nova migration Alembic |
| `make deploy-check` | Simula pipeline: lint + testes |
| `make clean` | Remove containers, volumes e imagens |

---

## 8. Referência de Endpoints

### Atletas

**Criar Atleta**
```bash
curl -X POST http://localhost:8000/v1/atletas/ \
  -H "Content-Type: application/json" \
  -d '{
    "nome": "João Silva",
    "cpf": "12345678901",
    "categoria_id": 1,
    "centro_id": 1
  }'
```

**Listar com filtros e paginação**
```bash
curl "http://localhost:8000/v1/atletas/?nome=João&limit=10&offset=0"
```

**Filtrar por CPF**
```bash
curl "http://localhost:8000/v1/atletas/?cpf=12345678901"
```

**Atualizar Atleta**
```bash
curl -X PUT http://localhost:8000/v1/atletas/1 \
  -H "Content-Type: application/json" \
  -d '{"nome": "João Souza"}'
```

**Deletar Atleta**
```bash
curl -X DELETE http://localhost:8000/v1/atletas/1
```

> A coleção Postman completa está disponível em `postman/apiWorkoutPython.postman_collection.json`  
> Inclui exemplos prontos para Atletas, Categorias e Centros de Treinamento.

---

## 9. Estrutura do Projeto

```
api-RESTful-FastAPI-Python/
├── app/
│   ├── api/
│   │   └── v1/
│   │       ├── routes_atletas.py     # Endpoints de Atleta (CRUD + paginação)
│   │       ├── routes_categorias.py  # Endpoints de Categoria
│   │       └── routes_centros.py     # Endpoints de Centro de Treinamento
│   ├── core/
│   │   ├── config.py                 # Settings via Pydantic BaseSettings + .env
│   │   └── logging.py                # Configuração de logging
│   ├── db/
│   │   ├── base.py                   # DeclarativeBase do SQLAlchemy
│   │   ├── session.py                # AsyncEngine + AsyncSession factory
│   │   └── init_db.py                # Inicialização de tabelas (dev utility)
│   ├── exceptions/
│   │   └── handlers.py               # IntegrityError → HTTP 303 + CPF extraído
│   ├── models/
│   │   ├── atleta.py                 # ORM: Atleta (cpf unique, FK categoria/centro)
│   │   ├── categoria.py              # ORM: Categoria
│   │   └── centro_treinamento.py     # ORM: CentroTreinamento
│   ├── repositories/
│   │   └── atleta_repo.py            # Queries async com selectinload
│   ├── schemas/
│   │   ├── atleta.py                 # Pydantic: AtletaCreate, AtletaRead
│   │   ├── categoria.py              # Pydantic: CategoriaCreate, CategoriaRead
│   │   └── centro_treinamento.py     # Pydantic: CentroCreate, CentroRead
│   ├── services/
│   │   └── atleta_service.py         # Orquestração de regras de negócio
│   └── main.py                       # App FastAPI: routers, CORS, exception handlers
├── alembic/
│   └── env.py                        # Migrations async com async_engine_from_config
├── docker/
│   ├── Dockerfile                    # python:3.11-slim
│   └── gunicorn_conf.py              # Configuração Gunicorn para produção
├── tests/
│   ├── conftest.py                   # Fixture AsyncClient
│   └── test_atleta.py                # Testes de endpoint
├── postman/
│   └── apiWorkoutPython.postman_collection.json
├── .github/workflows/ci.yml          # CI: build → migrations → lint → testes
├── docker-compose.yml                # Base: db com healthcheck
├── docker-compose.override.yml       # Dev: volume + uvicorn reload
├── docker-compose.prod.yml           # Prod: gunicorn 4 workers + restart policy
├── Makefile                          # Interface de operações
├── requirements.txt
└── .env.example
```

---

## 10. Pipeline CI/CD

O workflow `.github/workflows/ci.yml` executa em cada push para `main` e `develop`:

```
Checkout → Docker Buildx → Cache de layers → docker-compose up
    → pg_isready (healthcheck) → alembic upgrade head
    → black --check + flake8 → pytest -v → docker-compose down -v
```

A decisão de usar Docker Compose no CI (em vez de rodar Pytest diretamente no runner) garante que o ambiente de teste seja **idêntico ao de produção** — mesma imagem, mesmo PostgreSQL 15, mesmas variáveis de ambiente.

---

## 11. Aprendizados

**O que foi mais desafiador:** configurar o Alembic para rodar com engine assíncrono (`async_engine_from_config`) foi o ponto de maior atrito. A documentação oficial do Alembic ainda tem muitos exemplos síncronos, e a integração com `asyncpg` exigiu entender a diferença entre `run_sync` e `run_migrations_online` no contexto assíncrono.

**O que faria diferente hoje:** o handler de `IntegrityError` faz parsing da mensagem de erro do PostgreSQL via regex — funciona, mas é frágil se a mensagem mudar entre versões do driver. Uma abordagem mais robusta seria checar a existência do CPF antes do `INSERT` dentro de uma transação serializable, eliminando a dependência de parsing de strings de erro.

**Principal aprendizado arquitetural:** separar Repository do Service parece overhead em projetos pequenos, mas o benefício aparece imediatamente nos testes — é possível mockar o `AtletaRepository` e testar o `AtletaService` sem tocar no banco. Isso reduz o tempo de execução dos testes unitários em ordens de magnitude.

**Sobre Docker Compose com múltiplos arquivos:** o padrão `base + override + prod` é uma das decisões que mais economiza tempo operacional. Não há risco de subir produção com volume montado ou sem política de restart por esquecimento.

---

## 12. Próximos Passos

- [ ] Implementar endpoints completos de Categorias e Centros de Treinamento (atualmente retornam placeholder)
- [ ] Adicionar autenticação JWT para proteger endpoints de escrita
- [ ] Expandir cobertura de testes com fixtures de banco isolado por teste (PostgreSQL em container dedicado no CI)
- [ ] Adicionar endpoint de busca por atleta com paginação cursor-based para suportar volumes maiores
- [ ] Implementar cache Redis para listagens frequentes em período de competição
- [ ] Configurar observabilidade com OpenTelemetry + traces distribuídos

---

## Autor

**Sérgio Santos**  
Senior Data Engineer & Cloud Architect | 15+ anos em sistemas críticos de Banking & Fintech

[![Portfólio](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)
[![GitHub](https://img.shields.io/badge/GitHub-Santosdevbjj-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Santosdevbjj)

---

## Licença

Distribuído sob a licença MIT. Veja [LICENSE](LICENSE) para mais detalhes.

