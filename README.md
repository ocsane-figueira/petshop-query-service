# 🔍 Petshop Query Service (`petshop-query-service`)

Este repositório contém o **Microsserviço de Consulta** da arquitetura distribuída do Petshop. Ele representa o lado de **Leitura (Query)** no padrão CQRS (Command Query Responsibility Segregation).

---

## 🏗️ Papel e Funcionalidade no Ecossistema

O `query-service` é responsável pela consolidação e entrega de dados prontos e desnormalizados para o frontend/consumidores:
1. **Domínio de Leitura Ultra Rápida**: Em vez de fazer junções (`JOINs`) pesadas em tempo de execução entre tabelas relacionais de bancos diferentes, este serviço entrega dados de leitura instantânea estruturados como documentos JSON.
2. **Persistência de Alto Desempenho**: Utiliza **MongoDB** (NoSQL orientado a documentos) para salvar views consolidadas.
   * **Ambiente DEV**: Conecta-se ao cluster de desenvolvimento MongoDB.
   * **Ambiente HOMOL/PROD**: Conecta-se ao cluster de homologação/produção MongoDB.
3. **Consumo de Eventos Reativos (Sincronização CQRS)**: O serviço escuta constantemente as filas declaradas no **RabbitMQ**. Conforme eventos ocorrem nos microsserviços de escrita (Commands), o `query-service` os consome e atualiza sua base NoSQL:
   * **`ClientCreatedEvent`**: Adiciona ou atualiza dados desnormalizados do cliente.
   * **`AnimalCreatedEvent`**: Adiciona ou atualiza dados do pet mapeados.
   * **`AppointmentScheduledEvent`**: Captura o evento de agendamento, busca as referências salvas de cliente e animal e monta um documento unificado e rico chamado `AppointmentViewDocument` (contendo dados do agendamento + nome do cliente + nome do animal) gravando-o no MongoDB.
   * **`AppointmentCancelledEvent`**: Remove a view correspondente do MongoDB de forma automática.

---

## 🛠️ Tecnologias Principais

* **Java 21** e **Quarkus Framework**
* **MongoDB Client** (Integração otimizada para banco NoSQL)
* **SmallRye Reactive Messaging - RabbitMQ Connector** (Consumo reativo de filas assíncronas)
* **Jackson** (Processamento e serialização de documentos JSON de alta velocidade)
* **Quarkus Micrometer & Prometheus Registry** (Telemetria)

---

## 💻 Como Rodar o Serviço Localmente

### Pré-requisitos
* Java 21 JDK instalado localmente
* Maven instalado localmente (ou use o `./mvnw` incluso)
* Docker ativo para executar as bases de dados de suporte locais (consulte o repositório `petshop-infra`)

### Executando em Modo de Desenvolvimento (Live Coding)

Para iniciar o Quarkus localmente, conectado ao MongoDB e RabbitMQ rodando no Docker local:

```bash
./mvnw compile quarkus:dev
```

* **Porta local padrão**: `8084`
* **Painel Dev UI do Quarkus**: `http://localhost:8084/q/dev/`

---

## 🧪 Testes Automatizados e Ajustes de Configuração

O projeto possui suíte de testes unitários e de integração utilizando **JUnit 5**, **Mockito** e **RestAssured**:

### Executar Testes Locais
```bash
./mvnw clean verify
```

O Jacoco gerará o relatório visual em `target/jacoco-report/index.html` para auditar a cobertura de código (exigido mínimo de **50%** de cobertura).

---

## 🎛️ Observabilidade

O serviço expõe telemetria rica em tempo real para monitoramento corporativo:
* **Endpoint de Métricas**: `GET http://localhost:8084/q/metrics`
* Expõe estatísticas de consumo de mensagens por segundo do RabbitMQ, latências de escrita/leitura no MongoDB, tempos de resposta HTTP e métricas de sistema JVM.
* **Integração**: Coletado pelo Prometheus e encaminhado ao Grafana Cloud via `remote_write`.

---

## 🚀 Pipeline de CI/CD (GitHub Actions)

Este repositório possui fluxos totalmente automatizados integrando as melhores práticas DevOps:

1. **Continuous Integration (`ci.yml`)**:
   * Executado a cada push/pull request para as branches `main` e `develop`.
   * Realiza a compilação e validação do código com Java 21.
   * Envia análises de qualidade estática para o **SonarCloud** (Project Key: `ocsane-figueira_petshop-query-service`).
   * Para pushes aprovados em `main`, constrói a imagem Docker oficial multi-stage e envia para o Docker Hub com tags SHA e `main` (`ocsane/petshop-query-service`).

2. **Automatic Release (`release.yml`)**:
   * Executado na branch `main` pós-CI bem-sucedido.
   * Utiliza **Semantic Release** para analisar commits convencionais e atualizar o SemVer no GitHub automaticamente.

3. **Continuous Deployment (`cd.yml`)**:
   * O fluxo monitora a conclusão do CI. Caso a validação de testes finalize com sucesso:
     * Branch `develop`: Invoca o webhook do Render para atualizar o ambiente de desenvolvimento (`petshop-query-service-dev`).
     * Branch `main`: Invoca o webhook do Render para atualizar o ambiente de produção (`petshop-query-service`).
