# Modelagem e Persistência de Dados

## 1. Tecnologia Escolhida

- **Banco de Dados:** PostgreSQL 
 - Com `JSONB` + índice GIN (`data` > 1 GB ainda roda bem nos meus testes) 
 - Transações ACID garantem integridade em cenários de múltiplas operações (por ex., update de estoque + gravar pedido). 
 - Índices GIN aceleram buscas em campos JSON.

> Com base em projetos anteriores, eu acredito que JSONB, quando bem indexado, atende bem a volume alto de leitura sem prejudicar muito o desempenho.


## 2. Estrutura de Tabelas

### Tabela `data`

- **Path do arquivo de modelo:** `app/db/models/data_model.py` 
- Colunas principais: 
 - `id`: serial (chave primária). 
 - `user_id`: UUID (identifica usuário). 
 - `payload`: JSONB com detalhes do pedido (itens, valores, endereço). 
 - `created_at`: timestamp com fuso. 
 - `updated_at`: timestamp com fuso.

- **Índices:** 
 - B-Tree em `user_id` para consultas por usuário. 
 - GIN em `payload` para buscas específicas (status do pedido, itens). 
 - B-Tree em `created_at` para consultas por período.

### Tabela `users`

- **Path do arquivo de modelo:** `app/db/models/user_model.py` 
- Colunas principais: 
 - `id`: UUID (pk). 
 - `nome`: text. 
 - `email`: text (unique) — campo para login. 
 - `cpf_cnpj`: varchar(14) — usado para LGPD. 
 - `telefone`: varchar(11). 
 - `endereco`: JSONB (rua, número, bairro, cidade, uf, cep).

- **Índices:** 
 - B-Tree em `email` (unique). 
 - B-Tree em `cpf_cnpj` (unique, para evitar duplicidade de titular). 

### Tabela de Auditoria `audit_logs`

- **Path do arquivo de modelo:** `app/db/models/audit_model.py` 
- Colunas principais: 
 - `id`: serial (pk). 
 - `user_id`: UUID do usuário que executou ação. 
 - `endpoint`: text. 
 - `action`: text (ex.: “DELETE”, “ANONYMIZE”). 
 - `timestamp`: timestamptz. 
 - `metadata`: JSONB (extras, como status code ou detalhes do erro).

## 3. Políticas LGPD

### 3.1 Minimização e Anonimização

- **Masking no Service Layer:** 
 - Função em `app/services/masking_service.py`. 
 - Substitui campos como CPF por “***.***.***-XX”. 
 - Endereço no JSON também tem CEP mascarado.

> Já identifiquei vazamento de CEP em um projeto qem que trabalhei e precisei reaplicar masking — aqui já deixei de antemão no fluxo.

### 3.2 Exclusão (Hard Delete) e Anonimização (Soft Delete)

- **Hard Delete:** 
 - Endpoint em `app/api/routes/user_routes.py`. 
 - Ao chamar, o Service Layer faz: 
   1. `DELETE FROM data WHERE user_id = :user_id` 
   2. `DELETE FROM users WHERE id = :user_id` 
   3. Insere entrada em `audit_logs` com ação “HARD_DELETE”.

- **Soft Delete / Anonimização:** 
 - Outro endpoint em `app/api/routes/user_routes.py`. 
 - Altera colunas sensíveis para null ou placeholders: 
   1. `UPDATE users SET nome = 'Anonymized', email = NULL, cpf_cnpj = NULL WHERE id = :user_id` 
   2. `UPDATE data SET payload = jsonb_set(payload, '{endereco}', 'null') WHERE user_id = :user_id` 
   3. Grava em `audit_logs` ação “ANONYMIZE”.

### 3.3 Retenção de Dados

- **Script de limpeza periódica:** 
 - `scripts/cleanup_old_data.py` (invocado por Cron ou Kubernetes CronJob). 
 - Exclui dados em `data` com `created_at < now() - interval '365 days'`.

- **Estratégia de Backup:** 
 - Backups automáticos diários em RDS apenas se usar banco gerenciado. 
 - Retenção de 7 dias mas pode ser reconfigurado, criptografados em AES-256.

## 4. Alternativas Consideradas

- **MongoDB** 
 - Vantagem: sharding nativo, JSON nativo mas Postgres ainda seria uma melhor opção. 
 - Desvantagem: transações menos maduras, ACID limitado. 

- **Redis** 
 - Vantagem: latência extremamente baixa. 
 - Desvantagem: não é ideal para persistência de grandes volumes (memória). 

- **MySQL** 
 - Vantagem: Widespread, fácil de encontrar DBA. 
 - Desvantagem: JSONB no MySQL não tão otimizado quanto no PostgreSQL.

## 5. Particionamento e Sharding Futuro

- Se o volume de `data` crescer muito, seria uma boa escolha particionar por faixa de data em tabelas filhas. 
- Futuramente, considerar migração para Aurora ou cluster gerenciado com shards.
---
