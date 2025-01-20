# Documentação Técnica – Sistema de Concessão de Crédito

## Sumário

1. [Introdução](#introdução)  
2. [Contexto do Negócio](#contexto-do-negócio)  
3. [Visão Arquitetural](#visão-arquitetural)  
   1. [Descrição Geral](#descrição-geral)  
   2. [Diagrama de Containers](#diagrama-de-containers)  
   3. [Principais Componentes](#principais-componentes)  
   4. [Endpoints e Contratos de API](#endpoints-e-contratos-de-api)  
   5. [Integração e Resiliência](#integração-e-resiliência)  
4. [Atributos de Qualidade](#atributos-de-qualidade)  
   1. [Performance (Desempenho)](#performance-desempenho)  
   2. [Escalabilidade](#escalabilidade)  
   3. [Disponibilidade](#disponibilidade)  
   4. [Segurança](#segurança)  
   5. [Observabilidade e Monitoramento](#observabilidade-e-monitoramento)  
5. [Decisões de Implementação](#decisões-de-implementação)  
   1. [Linguagem e Frameworks](#linguagem-e-frameworks)  
   2. [Infraestrutura como Código (IaC)](#infraestrutura-como-código-iac)  
   3. [Padrões de Arquitetura](#padrões-de-arquitetura)  
   4. [Pipelines de CI/CD](#pipelines-de-cicd)  
   5. [Estratégia de Logs e Relatório de Erros](#estratégia-de-logs-e-relatório-de-erros)  
   6. [Feature Toggles e Deploy Canary/Blue-Green](#feature-toggles-e-deploy-canaryblue-green)  
6. [Custos e Pontos de Atenção](#custos-e-pontos-de-atenção)  
7. [Conclusão e Próximos Passos](#conclusão-e-próximos-passos)  
8. [Referências](#referências)

---

## 1. Introdução

Este documento descreve a **arquitetura de software** de um **Sistema de Concessão de Crédito**, contemplando:

- **Objetivos de negócio**: Aprovar limites de crédito para clientes de forma rápida e confiável.  
- **Design de microserviços** e APIs, com foco em **resiliência** e **alta disponibilidade**.  
- **Integrações** com serviços externos de histórico/score de crédito.  
- **Decisões de implementação**: Linguagem (Go), uso de AWS (ECS Fargate, RDS, API Gateway, etc.), infraestrutura como código (Terraform) e práticas de segurança (JWT, KMS, etc.).  
- **Estratégias** de desempenho (caching, paralelismo), segurança (JWT, criptografia), observabilidade (logs, métricas, tracing) e escalabilidade (auto scaling, event-driven).

---

## 2. Contexto do Negócio

A instituição financeira necessita de um serviço para **aprovar crédito** em **tempo real**, considerando:

- **Latência**: Resposta em ~200 ms.  
- **Disponibilidade**: SLA de 99,9%.  
- **Recalcular limite** a cada novo investimento (gatilho assíncrono/evento).  
- **Segurança**: Compliance com LGPD, PCI, etc., protegendo dados sensíveis do cliente.  
- **Escalabilidade**: 50 TPS em média, chegando a 600 TPS em picos.  

Além disso, há dependências de **APIs externas** (histórico/score) que podem falhar ou ficar indisponíveis. A arquitetura deve ser **resiliente** e **observável** para permitir rápida detecção e mitigação de problemas.

---

## 3. Visão Arquitetural

### 3.1 Descrição Geral

- **API Gateway**: Ponto de entrada único (REST) para o consumo do serviço de concessão de crédito.  
- **Microserviços**: Cada subdomínio (Concessão, Motor de Regras, Histórico, Score, Investimento) é um serviço independente, escalável e isolado.  
- **Comunicação**:  
  - **Síncrona** (REST/HTTPS) para a maioria das solicitações de análise.  
  - **Event-driven** (via tópico/fila) para disparar o recálculo de crédito após cada adesão de investimento.  
- **Armazenamento**: Cada serviço tem seu **banco de dados** ou cache próprio (ex.: RDS PostgreSQL, Redis).  
- **Resiliência**: Implementada por meio de timeouts, retries, backoff exponencial e circuit breakers nas integrações externas.

### 3.2 Diagrama de Containers
![legenda](./legenda.png)
![legenda](./diagrama.png)

### 3.3 Principais Componentes

1. **API Gateway** (AWS API Gateway)  
   - Expõe rotas HTTP/HTTPS para clientes internos e externos.  
   - Autenticação via JWT, rate limiting, logging centralizado.  

2. **MS - Concessao de Cartao**  
   - Orquestra todo o fluxo de aprovação, agregando dados de Score, Histórico e Motor de Regras.  
   - Após cada adesão de investimento (recebida via evento), recalcula possíveis novos limites.  

3. **MS - Motor de Regras**  
   - Encapsula as políticas de crédito (limites, perfis de risco, etc.).  
   - Atualizações frequentes de regras não afetam outros serviços.  

4. **MS - Historico de Crédito**  
   - Integra com APIs externas para verificar inadimplência e contratos ativos do cliente.  
   - Pode manter cache (Redis) para consultas repetidas ou fallback.  

5. **MS - Score de Crédito**  
   - Integra com bureaus de crédito (ex.: Serasa), armazena/atualiza pontuações em cache local ou Redis.  

6. **MS - Investimento**  
   - Gerencia adesões de investimento. A cada novo registro, publica evento de “investimento-adicionado” que aciona o MS - Concessao para reanálise.  

---

### 3.4 Endpoints e Contratos de API

Abaixo, **exemplos** de endpoints para cada serviço. As rotas são expostas pelo **API Gateway** e roteadas ao microserviço correspondente.

#### 3.4.1 MS - Concessao de Cartao

1. **POST** `/v1/cartao/concessao`  
   - **Descrição**: Inicia o processo de concessão de cartão.  
   - **Request Body (JSON)**:
     ```json
     {
       "idCliente": "12345",
       "tipoCartao": "GOLD"
     }
     ```
   - **Response Body (JSON)**:
     ```json
     {
       "limiteAprovado": 5000,
       "codigoDecisao": "APROVADO",
       "descricao": "Cartão aprovado com limite de R$ 5.000"
     }
     ```
   - **HTTP Status Codes**:  
     - `200 OK` em caso de sucesso  
     - `400 Bad Request` se dados inválidos  
     - `500 Internal Server Error` se falha inesperada

2. **GET** `/v1/cartao/status/{idCliente}`  
   - **Descrição**: Retorna status de concessão para o cliente.  
   - **Response (JSON)**:
     ```json
     {
       "idCliente": "12345",
       "statusCartao": "APROVADO",
       "limiteAtual": 5000
     }
     ```

#### 3.4.2 MS - Investimento

1. **POST** `/v1/investimentos`  
   - **Descrição**: Registra novo investimento.  
   - **Request (JSON)**:
     ```json
     {
       "idCliente": "12345",
       "tipoInvestimento": "CDB",
       "valor": 10000
     }
     ```
   - **Resposta**:
     ```json
     {
       "mensagem": "Investimento registrado com sucesso",
       "idInvestimento": 98765
     }
     ```
   - Emite um **evento** (`investimento-criado`) para o MS - Concessao de Cartao recalcular limite.

#### 3.4.3 MS - Historico de Crédito

1. **GET** `/v1/historico/contratos/{idCliente}`  
   - **Descrição**: Obtém lista de contratos ativos do cliente.  
   - **Response (JSON)**:
     ```json
     [
       {
         "idContrato": "CT-1001",
         "tipoProduto": "Empréstimo",
         "valorContratado": 5000,
         "saldoAberto": 2500
       },
       {
         "idContrato": "CT-1002",
         "tipoProduto": "Financiamento",
         "valorContratado": 20000,
         "saldoAberto": 15000
       }
     ]
     ```

#### 3.4.4 MS - Score de Crédito

1. **GET** `/v1/score/{idCliente}`  
   - **Descrição**: Retorna o score atual do cliente.  
   - **Response (JSON)**:
     ```json
     {
       "idCliente": "12345",
       "score": 750,
       "ultimaAtualizacao": "2025-01-01T10:00:00Z"
     }
     ```

#### 3.4.5 MS - Motor de Regras

1. **POST** `/v1/regras/avaliar`  
   - **Descrição**: Aplica regras de crédito com base em dados agregados (score, histórico, investimentos).  
   - **Request (JSON)**:
     ```json
     {
       "idCliente": "12345",
       "listaContratos": [
         { "idContrato": "CT-1001", "saldoAberto": 2500 }
       ],
       "listaInvestimentos": [
         { "tipoInvestimento": "CDB", "valor": 10000 }
       ],
       "score": 750
     }
     ```
   - **Response (JSON)**:
     ```json
     {
       "codigoDecisao": "APROVADO",
       "descricao": "Limite aprovado de R$ 5.000 baseado em score e histórico"
     }
     ```

> **Autenticação**: Todos os endpoints requerem cabeçalho `Authorization: Bearer <jwt_token>` validado pelo API Gateway.

---

### 3.5 Integração e Resiliência

Para **tolerar falhas** dos serviços externos (APIs de histórico e score), adotamos:

- **Timeouts**: Cada chamada externa é configurada com tempo de espera (ex.: 2s). Se exceder, interrompe e retorna erro controlado.  
- **Retries com Backoff Exponencial**: Em caso de falhas transitórias (HTTP 5xx ou timeout), tenta novamente até 3 vezes, escalonando o intervalo (ex.: 100ms, 200ms, 400ms).  
- **Circuit Breaker**: Se a taxa de erro ultrapassar um limiar (ex.: 50% das requisições em 30s), o circuito “abre” e as chamadas passam a falhar imediatamente com fallback (“não é possível obter histórico no momento”), evitando sobrecarregar o serviço ou ficar em loop de tentativas.  
- **Fallback**: Caso o score não seja consultado a tempo, podemos usar um “score padrão” ou classificar o cliente como “risco indefinido” (dependendo da política).  

A comunicação segura entre microserviços (dentro de uma VPC) é feita com **mTLS ou TLS**, e a **autorização** é controlada via **JWT** ou, internamente, **IAM Roles**/**Service Accounts** no ambiente AWS.

---

## 4. Atributos de Qualidade

### 4.1 Performance (Desempenho)

- **Metas**:  
  - Latência média: ~200 ms.  
  - Capacidade de picos de 600 TPS.  
- **Paralelismo**: O MS - Concessao de Cartao faz chamadas em **goroutines** simultâneas para Historico/Score, reduzindo tempo de espera total.  
- **Caching**: Redis (ElastiCache) para armazenar dados de score/histórico com tempo de expiração curto (ex.: 1 hora).  
- **Medição de Performance**:  
  - Ferramentas como K6 ou Locust para testes de carga.  
  - Monitoramento contínuo de p95 e p99 de latência para cada endpoint.

### 4.2 Escalabilidade

- **Auto Scaling**:  
  - AWS ECS Fargate scale-out baseado em métricas (CPU, memória, latência).  
- **Event-Driven**:  
  - Novos investimentos são processados de forma assíncrona, garantindo que picos de adesão não bloqueiem ou sobrecarreguem a análise de crédito em tempo real.  

### 4.3 Disponibilidade

- **SLA 99,9%**:  
  - Múltiplas AZs na AWS, réplicas de contêineres, balanceamento automático.  
- **Circuit Breakers** e **Fallbacks**:  
  - Evitam falhas em cascata caso um serviço crítico fique indisponível.  
- **Roteamento**:  
  - Se um contêiner estiver indisponível, o ECS retira-o do balanceador, garantindo mínima interrupção ao cliente.

### 4.4 Segurança

- **JWT** no cabeçalho `Authorization: Bearer <token>` para requests externos.  
- **OAuth2** (opcional) ou **AWS IAM** para comunicações internas.  
- **Criptografia** em repouso (AWS KMS) e em trânsito (TLS 1.2+).  
- **Políticas de Segurança**: Fine-grained IAM roles para cada microserviço, minimizando privilégios.

### 4.5 Observabilidade e Monitoramento

- **Logs Estruturados** em JSON via CloudWatch Logs.  
- **Métricas** (latência, TPS, CPU, memória, contagem de erros) visualizadas em Grafana/CloudWatch.  
- **Tracing Distribuído**: AWS X-Ray para depurar gargalos entre microserviços.  
- **Alertas**: CloudWatch Alarms enviando para Slack/E-mail em caso de anomalias (erros 5xx > 5%, latência p95 > 300ms, etc.).

---

## 5. Decisões de Implementação

### 5.1 Linguagem e Frameworks

- **Golang**:  
  - Execução rápida, binários leves, excelente para serviços de baixa latência.  
- **Framework**: Pode ser o nativo `net/http` ou **Gin** (que oferece roteamento e middlewares).  
- **Bibliotecas de Resiliência**: [sony/gobreaker](https://github.com/sony/gobreaker) para circuit breaker, estratégias de retry/backoff customizadas.

### 5.2 Infraestrutura como Código (IaC)

- **Terraform**:  
  - Declaração de todos os recursos AWS (VPC, ECS Fargate, RDS, ElastiCache, API Gateway).  
  - Pipelines executam `terraform plan` e `terraform apply` após aprovação.

### 5.3 Padrões de Arquitetura

- **Microserviços**:  
  - Reduz impacto de falhas e permite evolução independente de cada domínio.  
- **Event-Driven**:  
  - SNS/SQS ou Amazon EventBridge para publicar “investimento-criado”, desacoplando processos de recálculo.  
- **CQRS** (opcional/conforme necessidade):  
  - Separar responsabilidades de escrita (investimentos) e leitura (consultas ao histórico/score).

### 5.4 Pipelines de CI/CD

- **GitHub Actions** (ou Jenkins, CircleCI):  
  - Build, testes unitários, análise de segurança (GoSec), testes de integração.  
  - Geração de imagem Docker e push para ECR.  
- **Deploy Contínuo**:  
  - ECS Fargate com **Rolling Update** ou **Blue/Green** (AWS CodeDeploy), minimizando downtime.  
  - Validação do novo serviço antes de rotear tráfego.

### 5.5 Estratégia de Logs e Relatório de Erros

1. **Logs Estruturados**  
   - JSON com campos de correlação (`trace_id`, `service_name`), facilitando rastreamento.

2. **Coleta de Logs**  
   - CloudWatch Logs ou FireLens (Fluent Bit) para enviar logs a ELK/Splunk (dependendo da necessidade).

3. **Tracing Distribuído**  
   - AWS X-Ray com instrumentação de cada chamada HTTP.

4. **Relatório de Erros**  
   - **Sentry** ou Bugsnag para agrupar exceptions e gerar insights de falhas.

### 5.6 Feature Toggles e Deploy Canary/Blue-Green

- **Feature Toggles**:  
  - Implementadas via biblioteca em Go (ex.: GoFeatureFlag) ou sistemas de terceiro (LaunchDarkly).  
  - Permite ativar/desativar funcionalidades de forma gradativa sem reimplementar o serviço.  
- **Canary Release/Blue-Green**:  
  - AWS CodeDeploy ou Istio (se Kubernetes) para liberar tráfego gradualmente a uma nova versão.  
  - Se não houver erros, aumenta o tráfego até atingir 100%.

---

## 6. Custos e Pontos de Atenção

1. **Custos AWS**:  
   - ECS Fargate pode ser mais caro para workloads estáveis. Monitorar e avaliar Reserved Instances ou Savings Plans.  
2. **Complexidade de Microserviços**:  
   - Demanda CI/CD robusto, estratégias de logs e tracing distribuído para troubleshooting efetivo.  
3. **Conformidade**:  
   - LGPD/PCI exigem encriptação (KMS), segregação de dados e logs sem informações sensíveis.  
4. **Dependência de APIs Externas**:  
   - Mecanismos de fallback, caching e circuit breaker são essenciais para evitar indisponibilidade total.

---

## 7. Resumo das decisões 

| **Etapa**                        | **Pontos Solicitados**                                                                                      | **Escolhas Realizadas**                                                                                                                                                        | **Justificativa**                                                                                                                                                           |
|----------------------------------|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Etapa 1 - Design de API e Microserviços** | - Definição dos domínios das APIs, métodos, entradas e saídas. <br> - Arquitetura dividida em microserviços e comunicação. <br> - Garantir escalabilidade (CQRS/Event-Driven). | - **Domínios Definidos**: APIs documentadas para cada serviço (`/cartao/concessao`, `/historico`, `/score`). <br> - **Arquitetura Event-Driven**: Serviços desacoplados (ex.: evento "investimento-criado"). <br> - **Escalabilidade**: Uso de autoscaling (ECS Fargate) e padrões event-driven. | - APIs foram detalhadas com entradas, saídas e status HTTP, garantindo clareza. <br> - Microserviços permitem evolução independente e simplificam manutenção. <br> - Event-driven suporta picos de carga e desacoplamento. |
| **Etapa 2 - Integração dos Serviços** | - Estratégia resiliente para integrar serviços. <br> - Timeouts, retries/backoff exponencial, circuit breakers. <br> - Segurança para autenticação/autorização.                | - **Resiliência**: Timeouts configurados (2s), retries com backoff exponencial, circuit breakers (limitação de tentativas). <br> - **Segurança**: Autenticação via JWT e comunicação segura com TLS/mTLS.            | - Resiliência reduz impacto de falhas externas, garantindo continuidade do serviço. <br> - JWT e TLS protegem dados sensíveis e seguem padrões modernos de autenticação e segurança. |
| **Etapa 3 - Otimização de Desempenho** | - Técnicas para reduzir latência e aumentar capacidade (caching, lazy loading, paralelismo). <br> - Medir e monitorar performance.                                              | - **Caching**: Redis para dados frequentemente consultados (score, histórico). <br> - **Paralelismo**: Uso de goroutines no Go para chamadas simultâneas. <br> - **Monitoramento**: AWS X-Ray (tracing), CloudWatch (métricas). | - Caching reduz latência e dependência de APIs externas. <br> - Paralelismo no Go diminui tempo de resposta ao processar chamadas simultâneas. <br> - Ferramentas de monitoramento garantem visibilidade e identificação de gargalos. |
| **Etapa 4 - Qualidade de Código e Estratégia de Deploy** | - Garantia de qualidade com testes automatizados. <br> - Deploy contínuo com feature toggles e blue/green deployments.                                                         | - **Testes Automatizados**: Unitários (Golang), testes de integração e segurança (GoSec). <br> - **Deploy CI/CD**: GitHub Actions e AWS CodeDeploy com blue/green. <br> - **Feature Toggles**: Controle de lançamentos incrementais. | - Testes automatizados garantem confiabilidade do código. <br> - Deploy blue/green minimiza riscos, validando novas versões antes de liberar para todos. <br> - Feature toggles permitem ativar/desativar recursos sem downtime. |


---

## 7. Referências

- **Microservices Patterns (Chris Richardson)**:  
  [https://microservices.io/patterns](https://microservices.io/patterns)
- **AWS Well-Architected Framework**:  
  [https://aws.amazon.com/architecture/well-architected/](https://aws.amazon.com/architecture/well-architected/)
- **GoSec** (Análise de Segurança em Go):  
  [https://github.com/securego/gosec](https://github.com/securego/gosec)
- **Resilience in Go** (sony/gobreaker):  
  [https://github.com/sony/gobreaker](https://github.com/sony/gobreaker)
- **Terraform**:  
  [https://www.terraform.io/](https://www.terraform.io/)
- **Sentry**:  
  [https://sentry.io/](https://sentry.io/)
- **Grafana**:  
  [https://grafana.com/](https://grafana.com/)
- **Structurizr**:  
  [https://structurizr.com/](https://structurizr.com/)
