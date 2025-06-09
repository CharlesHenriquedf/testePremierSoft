# Estratégia de Testes

## 1. Visão Geral

Os testes estão divididos em:

- **Unitários**  
  - Servem para funções isoladas (validação de payload, lógica de serviço, masking LGPD).  
  - Não acessam banco real, usam mocks/fixtures.

- **Integração**  
  - Validam fluxo ponta-a-ponta: API, Service Layer e banco de teste (PostgreSQL em Docker).  
  - Fixtures devem limpar tabelas entre testes para garantir isolamento.

- **Performance**  
  - Simulam carga usando Locust (arquivo `tests/performance/locustfile.py`).  
  - Testes devem confirmar que latência média de POST/GET < 500 ms sob 1 000 usuários concorrentes.

- **Contract/Schema**  
  - Usam Schemathesis para validar que respostas da API batem com schema OpenAPI.  
  - Garante que, se o FastAPI mudar o modelo, testes falhem e alertem.

## 2. Estrutura de Testes

```

tests/
├── unit/
│   ├── test\_validation.py
│   ├── test\_service\_logic.py
│   └── test\_masking\_lgpd.py
├── integration/
│   ├── test\_post\_data\_integration.py
│   ├── test\_get\_data\_integration.py
│   └── test\_error\_flows\_integration.py
├── performance/
│   └── locustfile.py
└── conftest.py

```

- **`conftest.py`**: define fixtures de sessão de teste (banco em memória ou Docker), cliente FastAPI e override de `get_db`.

## 3. Metas e Ferramentas

- **Cobertura**  
  - Mínimo 90 % nos caminhos críticos (validação, persistência, leitura, erro).  
  - Ferramenta: `coverage.py`, relatório enviado para Codecov.

- **Ferramentas**  
  - **Unit/Integração:** Pytest + FastAPI TestClient  
  - **Performance:** Locust (script em `tests/performance/locustfile.py`)  
  - **Contract/Schema:** Schemathesis (configuração em `tests/integration/test_contract.py`)

## 4. Exemplos de Testes (Descrição Abrangente)

- **Validação de Payload (Unitário)**  
  - Verifique se payload sem campo obrigatório gera erro de validação HTTP 422.  
  - Verifica se valor total incorreto lança erro customizado no Service Layer.

- **Masking LGPD (Unitário)**  
  - Verifica que função de masking transforma CPF “12345678901” em “***.***.***-01”.  
  - Testa se endereço JSON tem “cep” substituído por “***-***”.

- **Endpoint POST /data (Integração)**  
  - Envia payload válido e verifica status 201 e registro no banco de teste.  
  - Envia payload inválido (campo faltando) e verifica status 422.

- **Endpoint GET /data (Integração)**  
  - Com banco vazio: retorna total = 0 e lista vazia.  
  - Com registros: retorna lista paginada e campos mascarados, status 200.

- **Contract/Schema**  
  - Carrega esquema OpenAPI e gera casos de teste que validam todos endpoints.  
  - Garante que tipos de dados, formas de erro e códigos HTTP correspondem ao schema.

- **Performance (Locust)**  
  - Roda script para simular 1 000 usuários fazendo requisições POST/GET.  
  - Mede latência média e percentis 95/99, confirmando < 500 ms.

## 5. Futuro

- Integrar testes de carga contínuos no pipeline CI/CD.  
- Adicionar testes de fuzzing ou API security scanning (ex.: OWASP ZAP).  
- Expandir cobertura para cenários de falhas externas (timeout de DB, erro de rede).
---
```
