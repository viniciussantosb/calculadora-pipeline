# Calculadora de Preços — Pipeline CI/CD e DevSecOps


Laboratório prático de **CI/CD e DevSecOps** desenvolvido com Python, Docker e GitHub Actions.

A aplicação é uma calculadora de preços propositalmente pequena. O objetivo principal do projeto não é a complexidade da aplicação, mas a construção de uma pipeline capaz de executar testes, validar a qualidade do código, analisar vulnerabilidades e verificar a imagem Docker antes de uma entrega.

> Projeto desenvolvido durante a disciplina **CI/CD, Pipelines e Testes Automatizados**, da pós-graduação em **DevOps & Cloud Platform Engineering com IA — PUC Minas**.

## Objetivos

- Automatizar testes unitários e de integração.
- Executar verificações de qualidade e cobertura de código.
- Detectar credenciais adicionadas indevidamente ao repositório.
- Realizar análise estática de segurança no código Python.
- Identificar vulnerabilidades em dependências e na imagem Docker.
- Publicar relatórios da pipeline como artefatos.
- Executar verificações diferentes em paralelo no GitHub Actions.

## Funcionalidades da aplicação

A aplicação possui funções simples de negócio para:

- calcular descontos;
- calcular impostos;
- calcular o preço final de um produto;
- salvar e consultar preços em um banco PostgreSQL.

## Pipeline implementada

```mermaid
flowchart TD
    A[Push ou pull request] --> T[Testes]
    A --> S[Segurança]
    A --> Q[Qualidade]
    T --> I[Testes de integração]
    S --> B[Build e scan da imagem]
    B --> M[Smoke test na main]
```

Os três workflows são disparados pelo mesmo evento e executam de forma independente.

### Testes — `tests.yml`

| Job | Responsabilidade |
| --- | --- |
| `unit-tests` | Executa testes unitários com Pytest, paralelismo e cache de dependências. |
| `integration-tests` | Inicia um PostgreSQL como service container e testa o repositório com uma conexão real. |
| `smoke-test` | Verifica a integridade básica do módulo após os testes e somente na branch `main`. |

### Segurança — `security.yml`

| Job | Responsabilidade |
| --- | --- |
| `security-scan` | Executa Gitleaks, Bandit e Trivy filesystem. |
| `build-and-scan` | Constrói a imagem Docker e analisa suas vulnerabilidades com Trivy. |
| `dast-smoke` | Sobe o contêiner e executa uma verificação funcional somente na branch `main`. |

> Apesar do nome `dast-smoke` utilizado no laboratório, essa etapa é tecnicamente um **smoke test funcional do contêiner**, e não um DAST completo contra uma aplicação HTTP.

### Qualidade — `quality.yml`

| Job | Responsabilidade |
| --- | --- |
| `lint-and-format` | Verifica lint e formatação com Ruff. |
| `test-coverage` | Executa os testes unitários, exige cobertura mínima de 80% e publica os relatórios. |

## Camadas de segurança

| Ferramenta | Finalidade | Critério utilizado |
| --- | --- | --- |
| Gitleaks | Detectar tokens, senhas e chaves presentes no histórico Git. | Falha quando encontra uma credencial não permitida. |
| Bandit | Analisar estaticamente o código Python em busca de padrões inseguros. | Severidades média e alta. |
| Trivy filesystem | Verificar vulnerabilidades nas dependências do projeto. | Vulnerabilidades altas e críticas com correção disponível. |
| Trivy image | Verificar o sistema operacional e os pacotes da imagem Docker. | Vulnerabilidades altas e críticas com correção disponível. |

## Qualidade e testes

- Testes unitários isolados com Pytest.
- Testes de integração com PostgreSQL real em service container.
- Execução paralela dos testes unitários com `pytest-xdist`.
- Cache das dependências Python para reduzir o tempo dos workflows.
- Verificação de lint e formatação com Ruff.
- Cobertura mínima obrigatória de 80%.
- Relatórios de cobertura em HTML e XML.
- Publicação do relatório do Bandit como artefato, inclusive em caso de falha.

## Tecnologias utilizadas

- Python 3.12
- Pytest
- Pytest-xdist
- Pytest-cov
- PostgreSQL
- Docker
- GitHub Actions
- Ruff
- Gitleaks
- Bandit
- Trivy

## Estrutura do projeto

```text
calculadora-pipeline/
├── .github/
│   └── workflows/
│       ├── quality.yml
│       ├── security.yml
│       └── tests.yml
├── src/
│   ├── __init__.py
│   ├── calculadora.py
│   └── repositorio.py
├── tests/
│   ├── integration/
│   │   └── test_repositorio.py
│   └── unit/
│       └── test_calculadora.py
├── Dockerfile
├── pytest.ini
├── requirements.txt
└── ruff.toml
```

## Como executar localmente

### Pré-requisitos

- Python 3.12
- Git
- Docker

### 1. Clonar o repositório

```bash
git clone https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git
cd SEU_REPOSITORIO
```

### 2. Criar o ambiente virtual

No PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

No Linux ou macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Instalar as dependências

```bash
pip install -r requirements.txt
```

### 4. Executar os testes unitários

```bash
pytest tests/unit/ -m unit -n auto --tb=short
```

### 5. Executar lint e verificar a formatação

```bash
ruff check src/ tests/
ruff format --check src/ tests/
```

### 6. Verificar a cobertura

```bash
pytest tests/unit/ -m unit --cov=src.calculadora --cov-report=term --cov-fail-under=80
```

### 7. Executar os testes de integração

Inicie o PostgreSQL:

```bash
docker run --name calculadora-postgres \
  -e POSTGRES_DB=calculadora \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  -d postgres:16-alpine
```

No PowerShell, configure a conexão e execute os testes:

```powershell
$env:DATABASE_URL = "postgresql://postgres:postgres@localhost:5432/calculadora"
pytest tests/integration/ -m integration --tb=short
```

No Linux ou macOS:

```bash
export DATABASE_URL="postgresql://postgres:postgres@localhost:5432/calculadora"
pytest tests/integration/ -m integration --tb=short
```

### 8. Construir e executar a imagem Docker

```bash
docker build -t calculadora-pipeline:local .
docker run --rm calculadora-pipeline:local
```

## Artefatos gerados

Após uma execução dos workflows, a aba **Actions** disponibiliza:

- relatório JSON do Bandit;
- relatório HTML de cobertura;
- relatório XML de cobertura;
- relatórios dos testes unitários e de integração.

## Decisões e limitações técnicas

### Workflows independentes

O GitHub Actions não permite utilizar `needs` para criar uma dependência entre jobs localizados em workflows diferentes. Por isso, `tests.yml`, `security.yml` e `quality.yml` executam e informam seus resultados de maneira independente.

O job `build-and-scan`, localizado em `security.yml`, aguarda o `security-scan`, mas não aguarda tecnicamente os jobs dos outros dois workflows. Uma coordenação rígida exigiria consolidar os jobs em um único workflow ou utilizar outro mecanismo de orquestração.

### Aplicação propositalmente simples

A calculadora não representa uma aplicação completa de produção. Sua simplicidade permite concentrar o laboratório nas práticas de CI/CD, testes, qualidade e segurança.

### Observabilidade

Esta versão ainda não possui uma stack de observabilidade. Não foram implementados traces, métricas ou centralização de logs.

## Próximos passos

- [ ] Exportar traces da pipeline com OpenTelemetry.
- [ ] Adicionar um OpenTelemetry Collector.
- [ ] Coletar métricas com Prometheus.
- [ ] Criar painéis no Grafana.
- [ ] Centralizar logs com Loki.
- [ ] Configurar os workflows como verificações obrigatórias para pull requests.
- [ ] Adicionar uma aplicação HTTP para executar um DAST real.

## Aprendizados

Este laboratório permitiu praticar:

- separação entre testes unitários e testes de integração;
- utilização de infraestrutura temporária dentro do CI;
- execução paralela de verificações independentes;
- implementação de quality gates;
- segurança integrada desde o início da pipeline;
- análise de dependências e imagens de contêiner;
- publicação de relatórios para investigação de falhas.

## Autor

**Vinícius Santos Barreto**

- LinkedIn: [Vinicius Barreto](https://www.linkedin.com/vinicius-s-barreto)

