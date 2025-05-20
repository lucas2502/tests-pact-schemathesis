# 🧪 Documentação: Testes de Contrato com Pact + Schemathesis

## ✅ Objetivo

Garantir que os serviços (frontend ↔ backend ou microserviços) respeitem os contratos de API definidos, tanto do ponto de vista funcional (mensagens esperadas) quanto estrutural (validação de schema, tipos, limites, etc.).

---

## 🏗️ Arquitetura Sugerida (CI/CD)

```text
                    ┌──────────────────────────────┐
                    │          Developer           │
                    └──────────────────────────────┘
                               │ Push (feature branch)
                               ▼
                   ┌───────────────────────────────┐
                   │        CI Pipeline (ex: GitHub Actions, GitLab CI, Jenkins)
                   └───────────────────────────────┘
                               │
      ┌────────────────────────┼─────────────────────────────┐
      ▼                        ▼                             ▼
┌───────────────┐     ┌────────────────┐              ┌────────────────────┐
│ Pact Consumer │     │ Pact Provider  │              │ OpenAPI Contract   │
│   Tests       │     │ Verification   │              │   (`openapi.yaml`) │
└───────────────┘     └────────────────┘              └────────────────────┘
        │                    │                                 │
        │ pact publish       │ pact verify                    ▼
        ▼                    ▼                        ┌──────────────────────┐
 ┌──────────────────┐  ┌────────────────────┐        │ Run Schemathesis CLI │
 │ Pact Broker/Flow │  │ Provider App Build │        │ → `schemathesis run` │
 └──────────────────┘  └────────────────────┘        └──────────────────────┘
        │
        ▼
   Pact Verification Matrix ✅
```

## ⚙️ Etapas no CI/CD

1. Testes Pact (Consumer)
   Executa os testes do consumidor (ex: frontend) para gerar o contrato pact:
   > npm run test:contract
2. Publicação no Pact Broker (ou Pactflow)
   > pact-broker publish ./pacts --consumer-app-version $GIT_COMMIT
3. Verificação no Provedor
   Valida que o backend implementa corretamente o contrato pact:
   > npm run verify:contract
4. Validação do Schema com Schemathesis
   Testa a conformidade da API com openapi.yaml, incluindo tipos, limites e campos obrigatórios:
   > schemathesis run openapi.yaml --base-url http://localhost:3000

## 📦 Exemplo de GitHub Actions

```text
name: Contract Tests

on: [push]

jobs:
  contract:
    runs-on: ubuntu-latest

    services:
      api:
        image: my-org/api:latest
        ports:
          - 3000:3000

    steps:
      - uses: actions/checkout@v3

      - name: Install deps
        run: npm ci

      - name: Run Pact (consumer tests)
        run: npm run test:contract

      - name: Publish Pact to Broker
        run: pact-broker publish ./pacts --consumer-app-version ${{ github.sha }}

      - name: Pact Verification (provider)
        run: npm run verify:contract

      - name: Run Schemathesis against OpenAPI
        run: |
          pip install schemathesis
          schemathesis run openapi.yaml --base-url http://localhost:3000


```

## 📄 Exemplo de OpenAPI Schema com Validações

```text
openapi: 3.0.3
info:
  title: API de Usuários
  version: 1.0.0
paths:
  /users:
    post:
      summary: Cria um novo usuário
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserCreate'
      responses:
        '201':
          description: Usuário criado
        '400':
          description: Dados inválidos

components:
  schemas:
    UserCreate:
      type: object
      required:
        - name
        - age
        - role
      properties:
        name:
          type: string
          maxLength: 50
        age:
          type: integer
          minimum: 18
          maximum: 99
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, user, guest]

```

## 🚀 Executando o Schemathesis

Com o servidor rodando localmente em http://localhost:3000, execute:

> schemathesis run openapi.yaml --base-url=http://localhost:3000 --hypothesis-seed=42

## 🧪 O que será testado?

Schemathesis irá automaticamente gerar:

✅ Casos válidos com todos os campos no formato correto

❌ Casos inválidos, como:

- name com mais de 50 caracteres

- age fora do intervalo 18–99

- email com formato incorreto

- role com valor fora do enum

- Campos ausentes (required)

## 🎯 Testar apenas o endpoint /users

> schemathesis run --endpoint "/users" openapi.yaml --base-url=http://localhost:3000

## 📊 Conclusão

Ferramenta Finalidade
Pact Valida o contrato funcional entre consumidor e provedor (mensagens trocadas)
Schemathesis Valida a conformidade com o OpenAPI schema (tipos, limites, fuzzing, etc)

🔒 Benefícios da Combinação
✅ Contrato funcional validado

✅ Conformidade de schema testada

✅ Detecção de regressões no CI/CD

✅ Adoção de práticas modernas de QA

## 🎁 Extras

Exportar relatório HTML

> schemathesis run openapi.yaml --base-url=http://localhost:3000 --report=relatorio.html
