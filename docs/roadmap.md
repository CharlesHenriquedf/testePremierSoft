# Roadmap e Otimizações Futuras

Fiz esse roadmap considerando alguns projetos anteriores, inclusive cenários de alta escalabilidade.

## 1. Curto Prazo (1–3 meses)

1. **Cache Redis para GET /data**  
   - Implementar camada de cache em Service Layer para resultados de consultas frequentes.  
   - TTL configurável para atualizar cache periodicamente.  

2. **Melhorar Dashboards de Observabilidade**  
   - Adicionar painéis em Grafana para latência, saturação de recursos.  
   - Configurar alertas para picos de erro ou latência > 500 ms.

3. **Documentação de Monitoramento**  
   - Incluir instruções de configuração de Prometheus em `docs/deployment.md`.  
   - Criar exemplos de alertas em Grafana no repositório `docs/alerts/`.

## 2. Médio Prazo (4–6 meses)

1. **Chaos Engineering**  
   - Desenvolver experimentos de análise (LitmusChaos ou Chaos Mesh) para validar.  
   - Exemplos: injetar falha na conexão com PostgreSQL, matar pods aleatórios.

2. **Canary Releases / Blue-Green Deployments**  
   - Configurar pipeline CI/CD para rota canário (10 % do tráfego) antes do rollout completo.  
   - Utilizar Istio ou Ingress controllers que suportem weight-based routing.

3. **Particionamento de Banco**  
   - Usar particionamento por faixa de data em tabela `data` se volume ultrapassar limites de performance.  

4. **Otimização de Custos**  
   - Migrar para instâncias reservadas em AWS para EC2 e RDS.  
   - Usar nós spot em EKS para workloads não críticos (batch, limpeza de dados antigos).

## 3. Longo Prazo (7–12 meses)

1. **Auto-Archiving**  
   - Criar job que move dados com > 12 meses para data warehouse (Redshift, BigQuery).  
   - Permite manter a performance transacional em tabelas primárias enxutas.

2. **Multi-Tenant e Escopo de Cliente**  
   - Adaptar modelo de dados para incluir `tenant_id` em todas tabelas.  
   - Isolar dados de diferentes clientes no mesmo banco ou usar schemas separados.

3. **Suporte a Múltiplos Formatos**  
   - Adicionar endpoints para upload de CSV e XML (`POST /data/csv`, `/data/xml`).  
   - Converter para JSON internamente e validar via Pydantic.

4. **Integração de Ferramentas de Observabilidade Avançada**  
   - Adicionar OpenTelemetry Collector no pipeline de logging.  
   - Consolidar logs em Loki ou Elasticsearch + Kibana para buscas avançadas.

---

# docker/Dockerfile

* **Descrição:**
  Multi-stage build que instala dependências de compilação e runtime, cria usuário sem privilégios e expõe porta 8000.
* **Local do arquivo:** `docker/Dockerfile`

# docker/docker-compose.yml

* **Descrição:**
  Cria contêineres para `db` (PostgreSQL) com volume persistente e `app` (microsserviço).
* **Local do arquivo:** `docker/docker-compose.yml`


# k8s/deployment.yaml

* **Descrição:**
  Deployment Kubernetes com 3 réplicas, readiness/liveness probes (`/health`), requests/limits de recursos e variável `DATABASE_URL` de Secret.
* **Local do arquivo:** `k8s/deployment.yaml`

# k8s/service.yaml

* **Descrição:**
  Service do tipo ClusterIP mapeia porta 80 para 8000 nos pods.
* **Local do arquivo:** `k8s/service.yaml`

# k8s/configmap.yaml

* **Descrição:**
  ConfigMap com variáveis de ambiente não sensíveis, como `APP_ENV` e `LOG_LEVEL`.
* **Local do arquivo:** `k8s/configmap.yaml`

# k8s/secret.yaml

* **Descrição:**
  Secret que armazena `DATABASE_URL` codificado em base64.
* **Local do arquivo:** `k8s/secret.yaml`

# k8s/hpa.yaml

* **Descrição:**
  Horizontal Pod Autoscaler que escala de 2 a 10 réplicas com base em uso médio de CPU ≥ 50 %.
* **Local do arquivo:** `k8s/hpa.yaml`

# k8s/pvc.yaml

* **Descrição:**
  PersistentVolumeClaim para banco de dados rodando internamente, solicita 10 Gi de armazenamento com modo ReadWriteOnce.
* **Local do arquivo:** `k8s/pvc.yaml`

---
```
