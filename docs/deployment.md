# Containerização e Implantação

## 1. Containerização com Docker

- **Dockerfile:** `docker/Dockerfile`  
  - Multi-stage build:  
    - Estágio de build instala dependências de compilação (libpq-dev, build-essential).  
    - Estágio final usa `python:3.11-slim`, copia apenas dependências e código.  
  - Cria usuário `appuser` para rodar sem privilégios de root.  
  - Expõe porta 8000 e define `CMD` para `uvicorn app.main:app`.

- **Docker Compose:** `docker/docker-compose.yml`  
  - Serviços:  
    - `db`: PostgreSQL, expõe porta 5432, volume `postgres_data`.  
    - `app`: constrói imagem via Dockerfile, depende de `db`, define env var `DATABASE_URL`.  
  - Comando adicional roda migrations (se usar Alembic) antes de iniciar API.

> Dica: no docker-compose, use `healthcheck` para banco, assim o serviço só inicia após DB estar pronto.

## 2. Orquestração com Kubernetes

- **Deployment:** `k8s/deployment.yaml`  
  - Define `replicas: 3`, readiness/liveness probes para `/health`, requests/limits de CPU (250 m/500 m) e memória (256 Mi/512 Mi).  
  - Variável `DATABASE_URL` vinda de `Secret` (`k8s/secret.yaml`).
  
- **Service:** `k8s/service.yaml`  
  - `type: ClusterIP`, mapeia `port: 80` para `targetPort: 8000` nos pods.

- **ConfigMap:** `k8s/configmap.yaml`  
  - Contém variáveis de ambiente não sensíveis (ex.: `APP_ENV=production`, `LOG_LEVEL=INFO`).

- **Secret:** `k8s/secret.yaml`  
  - Armazena `DATABASE_URL` codificado em base64.  
  - Exemplo de uso:  
    ```
    echo -n "postgresql://user:senha@host:5432/db" | base64
    ```

- **Horizontal Pod Autoscaler:** `k8s/hpa.yaml`  
  - Escalona entre 2 a 10 réplicas conforme uso médio de CPU ≥ 50 %.  
  - Usa `apiVersion: autoscaling/v2beta2`.

- **PersistentVolumeClaim:** `k8s/pvc.yaml` (opcional)  
  - Para caso de rodar PostgreSQL dentro do cluster como StatefulSet.  
  - Solicita 10 Gi de armazenamento com `accessModes: [ReadWriteOnce]`.

### 2.1 Sequência de Deploy

1. `kubectl apply -f k8s/configmap.yaml`  
2. `kubectl apply -f k8s/secret.yaml`  
3. `kubectl apply -f k8s/deployment.yaml`  
4. `kubectl apply -f k8s/service.yaml`  
5. `kubectl apply -f k8s/hpa.yaml`

> Observação: se o secret já estiver criado, ignore o passo 2 para não sobrescrever.

### 2.2 Observabilidade durante o Deploy

- Expor endpoint `/metrics` para Prometheus coletar métricas.  
- Logs de aplicação enviados ao stdout/stderr, capturados pelo Kubernetes logging.  
- Tracing via OpenTelemetry coletado por `otel-collector` (deployment opcional), configurado em `deployment.yaml`.

## 3. Considerações de Escalabilidade

- **Probes de Saúde:**  
  - `readinessProbe` com `initialDelaySeconds: 10`, verifica `/health`.  
  - `livenessProbe` com `initialDelaySeconds: 30`, reinicia container em caso de falha.

- **Recursos e Limites:**  
  - `resources.requests.cpu: "250m"`, `resources.requests.memory: "256Mi"`.  
  - `resources.limits.cpu: "500m"`, `resources.limits.memory: "512Mi"`.

- **Autoscaling:**  
  - HPA aumenta réplicas automaticamente quando CPU média do pod ultrapassa 50 %.  
  - Permite escalar de 2 até 10 pods conforme demanda.

---

# docs/ci-cd.md

# Pipeline CI/CD

## 1. Ferramenta Escolhida

- **GitHub Actions**  
  Utilizada porque integra diretamente com repositório, tem ações prontas, e permite rodar jobs paralelamente (lint, testes, build, deploy).  
  - Jobs chave:  
    1. Lint e Qualidade de Código  
    2. Testes Unitários e Integração (com cobertura)  
    3. Build e Push de Imagem Docker  
    4. Scan de Segurança (Trivy)  
    5. Deploy em Kubernetes

## 2. Diagrama do Pipeline

- **Arquivo de Diagrama:** `docs/diagrams/ci-cd-pipeline.png`  
- **Fonte Editável:** `docs/diagrams/ci-cd-pipeline.drawio`

Fluxo:
1. Checkout do código.  
2. Setup Python 3.11 e Node.js 16.  
3. Instala dependências Python e JS (para commitlint).  
4. Executa pre-commit hooks (Black, isort, Ruff, Mypy, Bandit, Safety, commitlint).  
5. Roda testes (`pytest --cov=app`).  
6. Envia cobertura ao Codecov.  
7. Constrói imagem Docker (`docker build`).  
8. Faz push para o registry (`docker push`).  
9. Scan de vulnerabilidades com Trivy.  
10. Deploy no cluster Kubernetes (`kubectl apply` ou `kubectl set image`).

## 3. Workflow (Sem Código Completo)

- **Arquivo no Repositório:** `.github/workflows/ci-cd.yaml`  
- **Descrição Simplificada das Etapas:**  
  1. **Lint e Testes**  
     - Executa `pre-commit run --all-files` para verificar formatação e lint.  
     - Executa `pytest --cov=app` para rodar testes e gerar relatório de cobertura.  
  2. **Build e Push**  
     - Login no Docker Hub usando segredos (`DOCKER_USERNAME`, `DOCKER_PASSWORD`).  
     - Constrói imagem: `seu-registry/data-api:${{ github.sha }}`.  
     - Push da imagem ao registry.  
     - Executa Trivy para scan de vulnerabilidades.  
  3. **Deploy**  
     - Configura `kubectl` usando `KUBE_CONFIG_DATA` (base64).  
     - Aplica manifestes K8s ou atualiza imagem em Deployment.  
     - Aguarda rollout completar (`kubectl rollout status`).

## 4. Justificativas e Alternativas

- **GitHub Actions vs Jenkins/GitLabCI**  
  - Ações prontas, runners escalonáveis, manutenção pela própria equipe GitHub.  
  - Para repositórios privados, GitLab CI também seria viável, mas GitHub Actions simplificou comunicação com reviewers.

---
