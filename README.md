```plaintext
/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── database.md
│   ├── deployment.md
│   ├── ci-cd.md
│   ├── testing.md
│   ├── roadmap.md
│   └── diagrams/
│       ├── architecture.drawio
│       ├── dataflow.png
│       ├── ci-cd-pipeline.png
│       └── k8s-deployment.png
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    └── pvc.yaml
```

---

# README.md

## 1. Visão Geral

O projeto demonstra apenas a parte do design de um microsserviço que pode receber e retornar grandes volumes de dados, o sistema foi pensado para ser rápido  respeitando as faixas determinadas de (<500 ms de latência) e escalável, com práticas de observabilidade garantindo a conformidade LGPD. Nesse tipo de arquiteturo eu procuro sempre fazer uma análise aprofundade dos requisitos de infra da aplicação pois, quando esses requisitos estão bem determinado e as análises preditivas em relação a possíveis sobrecargas na infra estão bem documentatas e constando no SLO do cliente, caso ocorra algum problema em relação ao deploy de uma nova funcionalidade, a maior probabilidade é que esse problema esteja relacionada a alguma área não coberta pelos testes, e nesse caso a correção do problema fica com um foco mais direcionado.

Principais pontos:

* **POST /data**
  Recebe transporte JSON, valida e faz a persistencia no banco.
* **GET /data**
  Busca os registros com paginação e “enumero” de resultados, aplicando masking onde precisar por LGPD.

Em alguns casos que em que trabalhei os testes locais podem gerar tempo de resposta maior que em produção, em funçaõ disso o  projeto inclui recomendações de índices e cache para facilitar.

### Funcionalidades Essenciais

* Receber dados de pedido (ou contexto similar)
* Validar campos obrigatórios (id de usuário, itens, preço total)
* Persistir em PostgreSQL (JSONB)
* Recuperar dados em alta escala, mascarando dados sensíveis por segurança

## 2. Estrutura do Projeto

* **docs/**: local da documentação tecnica (arquitetura, banco, CI/CD, testes, deployment, roadmap).
* **docs/diagrams/**: os diagramas em  PNG (fluxo de dados, deploy, pipeline).
* **docker/**: os arquivos de Docker para criar ambiente local e deploy (Dockerfile, docker-compose.yml).
* **k8s/**: o manifests Kubernetes (deployment, service, configmap, secret, hpa, pvc).

> Lembrando que os arquivos não tem código e são apenas para ilustrar a estrutura
## 3. Decisões Técnicas Principais

* **Arquitetura em Camadas**

  * API Layer (FastAPI)
  * Service Layer (lógica do negócio e masking LGPD)
  * Persistence Layer (PostgreSQL via SQLAlchemy)

* **Banco de Dados**
  Escolhi o banco PostgreSQL e esplico os motivos em `docs/database.md` (suporte a JSONB, transações ACID, índices GIN).

  * Usei JSONB para gerenciar payloads flexíveis e índices para queries frequentes para otimização.

* **Containerização**
  Docker e Docker Compose (`docker/Dockerfile` e `docker/docker-compose.yml`) para um ambiente consistente tanto no desenvolvimento quanto em produção.

  * No Dockerfile, criei estágio de build e runtime para reduzir o tamanho da imagem.

* **Orquestração**
  Kubernetes (`k8s/`):

  * Deployment com 3 réplicas, probes de saúde e limites de CPU/memória.
  * Service tipo ClusterIP, ConfigMap/Secret para variáveis de ambiente.
  * HPA para ajustar réplicas conforme uso de CPU (entre 2 e 10).
  * PVC é opcional se optar por banco dentro do cluster.

* **CI/CD**
  GitHub Actions conforme `docs/ci-cd.md`:

  * Hooks pre-commit (Black, isort, Ruff, Mypy, Bandit, Safety) + commitlint.
  * Testes (unitários, integração, cobertura) + envio para Codecov.
  * Build e push de imagem Docker + scan de vulnerabilidades (Trivy).
  * Deploy em Kubernetes via kubectl.

* **Testes Automatizados**
  Estrutura em `docs/testing.md`:

  * Unitários: validações, lógica de service, masking LGPD.
  * Integração: endpoints ponta-a-ponta usando banco de teste em container.
  * Performance: Locust para simular carga de usuários (latência < 500 ms).
  * Contract/Schema: Schemathesis para validar OpenAPI gerado.

* **Observabilidade**

  * Contém as métricas com Prometheus expostas em `/metrics` (contadores e histogramas) para análise.
  * Os logs estruturados em JSON (mascarando dados sensíveis antes de gravar para segurança).
  * Tracing com OpenTelemetry para spans de requisição HTTP e queries ao banco.
  * Captura de exceções no Sentry para relatórios de erro e etc.

* **Segurança e LGPD**

  * Optei por Autenticação JWT, roles (admin, user), rate limiting (Redis).
  * Masking de dados: CPF, endereço, e-mails mascarados no Service Layer.
  * Para mais conformidade, endpoints para exclusão/anonimização de dados (direito à eliminação).
  * Tabela de auditoria (`audit_logs`) para registrar acesso e ações.

## 4. Instruções Rápidas de Execução

1. **Clonar o repositório**

   ```bash
   git clone <https://github.com/CharlesHenriquedf/testePremierSoft
   >
   cd <REPO>
   ```

2. **Ambiente Python**

   * Criar virtualenv (Python ≥ 3.11).
   * Instalar dependências:

     ```
     pip install -r requirements.txt
     pre-commit install
     ```

3. **Banco de Dados (Desenvolvimento)**

   ```bash
   docker-compose -f docker/docker-compose.yml up -d
   ```

   * Cria contêiner PostgreSQL com volume.

4. **Executar Aplicação**

   ```bash
   uvicorn app.main:app --reload
   ```

   * Disponível em `http://localhost:8000`.

5. **Testar Endpoints**

   * **POST /data**: envie JSON de pedido (veja estrutura em `docs/database.md`).
   * **GET /data**: experimente paginação, veja se dados sensíveis vêm mascarados.

6. **Deploy em Kubernetes**

   ```bash
   kubectl apply -f k8s/
   ```

   * Certifique-se de ter `Secret` e `ConfigMap` configurados corretamente.

7. **Pipeline CI/CD**

   * O arquivo de workflow está em `.github/workflows/ci-cd.yaml` (detalhes em `docs/ci-cd.md`).
   * Em cada push para `develop` ou `main`, a pipeline roda lint, testes, build e deploy (se na branch `main`).

## 5. Diagramas Técnicos

Optei por organizar todos em `docs/diagrams/`:

* `docs/diagrams/architecture.drawio` e `docs/diagrams/architecture.png` (arquitetura em camadas).
* `docs/diagrams/dataflow.png` (fluxo de dados entre API, Service e DB).
* `docs/diagrams/ci-cd-pipeline.png` (visualização da pipeline CI/CD).
* `docs/diagrams/k8s-deployment.png` (deploy no Kubernetes).

## 6. Roadmap e Otimizações Futuras

Os detalhes em `docs/roadmap.md`:

* **Curto Prazo:**  além de adicionar cache Redis, também melhorar dashboards Prometheus/Grafana.
* **Médio Prazo:** chaos engineering, canary releases.
* **Longo Prazo:** particionamento de banco, auto-archiving, custos em nuvem (AWS).

---
