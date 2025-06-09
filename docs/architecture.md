# Arquitetura do Microsserviço

## 1. Visão Geral

Este microsserviço segue uma estrutura em camadas que facilita manutenção e escalabilidade. A arquitetura foi projetada para suportar picos de tráfico com HPA e Kubernetes com o uso opcional de cache para garantir o acesso de milhoes de usuários de muita degradação. A cada projeto novo que participei, notei que separação em camadas ajuda bastante nos testes e deploys sem downtime. As camadas são:

1. **API Layer** 
  - Recebe requisições HTTP (FastAPI). 
  - Valida payloads usando Pydantic, dispara erros HTTP 422 se algo não bater. 
  - Autentica/tokeniza via JWT e aplica rate limiting (Redis).

2. **Service Layer** 
  - Contém as regras de negócio (cálculo de valor total, verificação de estoque, masking LGPD). 
  - Orquestra chamadas a Persistence Layer e a APIs externas (pagamento, notificações). 
  - Emite métricas para Prometheus (latência, contagens de requisições).

3. **Persistence Layer** 
  - Responsável por CRUD no PostgreSQL via SQLAlchemy. 
  - Gerencia conexões, transações e aplica índices (JSONB, B-Tree). 
  - Scripts de migration (Alembic) em `alembic/versions/` (opcional).

> Nota do autor: já vi cenários onde colocar lógica de negócio na camada de API gerava código inchado, então evitei esse erro aqui.

## 2. Diagrama de Arquitetura

- **Arquivo-fonte:** `docs/diagrams/architecture.drawio` 
- **Imagem exportada:** `docs/diagrams/architecture.png`

Este diagrama mostra:
- Fluxo de uma requisição `POST /data` desde o cliente até escrita no banco. 
- Como métricas são coletadas e expostas. 
- Conexão ao Sentry para registro de erros.

## 3. Fluxo de Requisição

### 3.1 POST /data

1. Cliente envia JSON para `/data`. 
2. API Layer valida o JSON (Pydantic) e verifica autenticação. 
3. Service Layer processa regras (soma de itens, valida valores, verifica estoque). 
4. Service Layer chama Persistence Layer: grava no PostgreSQL usando JSONB. 
5. Persistence retorna resultado ao Service, que emite métricas e logs estruturados. 
6. API Layer devolve HTTP 201 com ID e timestamp.

### 3.2 GET /data

1. Cliente faz GET em `/data?limit=...&offset=...`. 
2. API Layer extrai parâmetros e checa autorização. 
3. Service Layer executa consulta paginada no banco (limit/offset ou cursor-based). 
4. Persistence retorna lista de registros; Service aplica masking de campos sensíveis. 
5. API Layer devolve HTTP 200 com lista e metadados (total, limit, offset).

## 4. Justificativas e Trade-offs

- **Separação de Camadas** 
 - Permite testar Service Layer isoladamente (unit tests), sem subir servidor HTTP. 
 - Facilita atualização de uma camada sem impactar as outras. 
 - Trade-off: aumento de chamadas internas (overhead mínimo), mas ganho em manutenção.

- **JSONB vs Tabelas Normalizadas** 
 - JSONB: permite payloads flexíveis e evolução do esquema sem alterar migrations. 
 - Tabelas normalizadas: melhor performance em JOINs e integridade referencial. 
 - Escolha: JSONB, pois cenários de e-commerce podem ter campos variando conforme negócio.

- **Observabilidade Embutida** 
 - Inserir métricas e tracing no código aumenta o tempo inicial de desenvolvimento. 
 - Mas o retorno é maior visibilidade em produção, reduzo MTTR.

- **Monolito vs Microserviço** 
 - Monolito: menos overhead de comunicação, deploy mais rápido em protótipos. 
 - Microserviço: isola falhas, escalabilidade independente. 
 - Optamos por microserviço para poder escalar apenas a API quando necessário sem afetar outros componentes.

---
