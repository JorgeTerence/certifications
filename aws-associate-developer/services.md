# AWS Associate Developer

## Analítica

### Athena

Filtrar e transformar queries de objetos S3. Opera filtrando vários objetos do bucket. Mais complexo que S3 Select.

### Kinesis

Ideal para fluxos ou streams de dados.

Shard é a unidade de fluxo de dados; +shards = +fluxo

Shards precisam de um worker para processar os dados, idealmente 1/1

### OpenSearch Service

Permite a criação de dashboards com Kibana a partir de dados vindos do Kinesis

---

## Integração

### Step Functions

Visual workflows

State machines

Cria *Activities* como unidade básica de processamento, criada com `CreateActivity`

Atividades precisam de um *worker* como back-end, seja Lambda, ECS, EC2 etc.

Esperar tarefas assíncronas com `.wait ForTaskToken`

Existem APIs nativas para controlar o workflow:

- [`CreateActivity`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_CreateActivity.html)
- [`GetActivityTask`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_GetActivityTask.html)
- [`ListActivities`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_ListActivities.html)
- [`SendTaskFailure`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_SendTaskFailure.html)
- [`SendTaskHeartbeat`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_SendTaskHeartbeat.html)
- [`SendTaskSuccess`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_SendTaskSuccess.html)

### SQS (Simple Queue Service)

Fila de mensagens

Deletar fila → deleta itens permanentemente

Tempo de vida máximo = 14 dias

Tamanho máximo de item = 256kB

Nome < 80 caracteres, único na conta AWS

*Long pooling* = esperar até 20s para ler próximo item

*Short pooling* = itens são lidos imediatamente

### SNS (Simple Notification Service)

JSON `{ MessageId, unsubscribeURL, Subject, Message, … }`

Suporta HTTP e SMS

Novos tópicos criam um novo ARN (*Amazon Resource Name*), máximo de 3000

### AppSync

Connect to AWS services through GraphQL API

### Event Bridge

---

## Compute

### EC2 (Elastic Cloud Compute)

Máquinas virtuais

Dados persistentes com [EBS (Elastic Block Store)](https://www.notion.so/EBS-Elastic-Block-Store-157a20190a6180119755ce1bbc1dd62a?pvs=21)

Initialization script = `UserData` field

Metadados da instância contém informações do hardware, como acesso à GPU e endereço de IP. Acessível através de http://169.254.169.254/latest/meta-data

- **Spot instances:** serviços podem ser inicializados e interrompidos de forma flexível
- **Reserved host:** servidor 100% dedicado
- **Instance Saving Plan:** contrato de uso/hora
- **Reserved Instance:** similar, porém sem limite de uso por hora

### Lambda Functions

Invocar scripts através de API, SDK ou eventos dentro da AWS

Paga por segundos de execução

Máximo de 15 mintutos

Valores persistentes são fora do método `handler`

`${stageVariables.LambdaAlias}`

Código pode ser alocado em camadas, ou *layers*, que são compartilhadas entre funções

***Lambda@Edge*** functions encaixam com o serviço CloudFront para serem executadas em localidades específicas; também podem ler e modificar requisições da CloudFront. São mais limitadas do que Lambda functions convencionais.

Logs de console são preservados

Id da requisição fica dentro do objeto de contexto

Usando invocação assíncrona (`--invocation-type Event`), a infraestrutura do Lambda toma conta do calendário de execução com uma fila. Isso não significa que o startup será mais rápido. Apenas um código de sucesso é retornado, nada mais.

+Memória = +CPU

Header `X-Amz-Invocation-Type = ‘Event’` invoca uma função assíncrona, que apenas retorna um resultado sobre o sucesso ou falha, sem outro retorno.

### Elastic Beanstalk

Implantação e escalonamento de aplicações

Utiliza Amazon Linux AMI ou Amazon Linux 2

Permanece disponível ao longo de mudanças de provisão

**Estratégias de implantação**

- **All-at-once:** deixa tudo offline e sobe de volta com nova versão.
- **Rolling:** deixa n deploys offline (batch) de cada vez.
- **Rolling with additional batch:** pra cada deploy da batch, cria um novo deploy com a última versão. Quando a build termina, integra o novo ao antigo, termina o antigo. Zero downtime.
- **Blue-green:** criar nova URL para novo deploy. Só distribui nova versão quando o deploy tiver sucesso, daí é possível trocar o direcionamento dos recursos para a versão *green*.
- **Canary:** entregar para uma porção de usuários de cada vez, observando caso ocorra alguma falha. Limita escopo de falhas.
- **Immutable:** criar novas instâncias com a nova versão. Mais demorado. Zero downtime.
- **Traffic-splitting:** redirecionar tráfego para deploys recentes enquanto builds ocorrem.

Atualizar código das implantações `eb deploy`

Cobranças são contadas mesmo com a instância offline

[Sample config](https://www.notion.so/Sample-config-158a20190a61804b88c6e3219f401569?pvs=21)

### SAM (Serverless Application Model)

`sam build` e `sam deploy`

---

## Containers

### EKS (*Elastic Kubernetes Service*)

Run Docker containers with Kubernetes for orchestration

### ECS (*Elastic Container Service*)

Variáveis de ambiente da aplicação podem ser definidas no passo de ***Task Definition***

### ECR (*Elastic Container Registry)*

Evitar builds duplicados de imagens

### Copilot

CLI para configurar e implantar aplicações baseadas em containers.

Tem como destinatários workers como ECS, Fargate e App Runner

---

## Databases

| Título | Modelo de dados | Diferencial |
| --- | --- | --- |
| RDS | SQL |  |
| Aurora | Enterprise SQL | multi AZ por padrão, 6 clusters |
| DynamoDB | *key-value* + document | extremamente veloz |
| DocumentDB | document | compatível com MongoDB |
| ElastiCache | key-value (memory cache) | compatível com Redis + Valkey |
| Neptune | graphs |  |
| QLDB (*Quantum Ledger)* | blockchain | histórico de transações |
| Redshift | data lake | big data |

### DynamoDB

`Tabelas > key-value > propriedade`

Para que funcionem as *triggers*, serviço de *streams* deve estar habilitado

Não suporta criptografia *in-place*

Uma tabela é uma coleção de itens

Não há suporte para remoção em massa de itens

TTL (*Time To Live*) pode ser usado para deletar itens antigos automaticamente

***Transaction*** requests permitem a leitura ou edição de vários campos simultaneamente de uma forma atômica. Isso reduz o tempo de transmissão de diversas requisições HTTP e evita conflito de estados entre diferentes requisições.

***Scan Operations*** são eventualmente consistentes

Suporta inserções condicionais, baseado no estado atual da tabela.

***Capacidade***

***RCU*** (*Read Capacity Unit*) = $\lceil doc\rceil\;/\;8\text{kB} *\#reads$

Se I/O exceder RCU, requisições ficam engasgadas, resultando em HTTP 400

***DynamoDB Accelerator (DAX)*** é um serviço de otimização compatível com o SDK padrão da DynamoDB. Gerencia cache automaticamente, permite até 10x mais velocidade na leitura de dados e replicação de dados em clusters. Máximo de 10 nós por cluster. O cache é *eventualmente consistente*. 

***Chaves de tabela***

- ***primary-key:*** propriedade índice padrão da BD, criando identidade. pode ser multi-valorada
- ***partition-key:*** propriedade que determina como registros são agrupados fisicamente
- ***sort-key:*** determina a ordenação de registros dentro de uma partição

É impossível modificar índices primários

***Indexes***

- ***Local Secondary Indexes (LSI):*** apenas sort key diferente
    - Máximo 5 LSIs por tabela
    - Máximo 10GB por partição
    - Criados no mesmo instante da BD, imutáveis
- ***Global Secondary Indexes (GSI):*** partition key e sort key diferentes
    - Máximo 20 GSIs por tabela
    - Flexíveis: criados ou removidos a qualquer momento
    - Limites de RCU separados da BD primária

### ElastiCache

Sincronização de dados com *write-through*

Ideal para armazenar sessões de usuário ao longo de diferentes conexões.

- ***Lazy-loading:*** buscar primeiro no cache. Se resultado for nulo, buscar na BD.
- ***Write-through:*** sincronizar cache a todo post de dados. Sempre sincronizado.
- ***Adding TTL:*** dados no cache são excluídos com o passar do tempo.
- ***Write-behind:*** post é efetivado e cache é atualizado de forma assíncrona.
- ***Refresh-ahead:*** cache tenta predizer quais dados serão buscados e atualiza cache. Tenta ganhar eficiência com cache inteligente, mas arrisca ser fonte de latência.

### RDS

Tamanho máximo por instância = 3TB

*Read replicas* são atualizadas de forma assíncrona, ou seja, podem levar alguns minutos ou horas para serem atualizadas com os últimos registros.

---

## Development

### Amplify

Especializado na implantação de front-ends e apps full-stack

Construir apps mobile usando frameworks populares

Conecta automaticamente com repositórios Git

### CI-CD Familly

| CodeCommit | versionamento Git |
| --- | --- |
| CodeBuild | build |
| CodeDeploy | deploy |
| CodePipeline | linha de CI/CD |
| CodeArtifact | ??? |

***CodeDeploy*** apenas suporta deploys *in-place* e *blue/green. Procura `appspec.yml` na raiz do diretório da aplicação.*

### X-Ray

Container com daemon para monitoramento de performance das aplicações na rede

Acessado através da porta UDP 2000

### Code Guru

IA para verificar vulnerabilidades de código

### Cloud Shell

Terminal web com AWS CLI e autenticação já configurados.

Suporta Bash, PowerShell, ZSH etc.

Limite de 1GB de armazenamento

---

## Management and Governance

### AppConfig

Controla variáveis dentro da nuvem AWS, dando controle manual a configurações e interruptores da aplicação e implantações.

- Feature flags e toggles
- Bloqueio de usuários
- Ajuste de recursos da aplicação
- Centralização da configuração

### CDK (*Cloud Development Kit*)

### CloudFormation

IaC escrito em JSON ou YAML

Código *inline* com propriedade `zipFile`

Organiza recursos de várias contas com `Stack Set`

Mappings = pares de chave e valor

Transform = macros dentro do template

### CloudTrail

Monitorar invocação de serviços dentro da AWS

Ideal para monitoramento e audição de software

### CloudWatch

Registro de logs de sistemas

Serviços precisam de permissão IAM para registrar logs

### Systems Manager Parameter Store

key-value stores

Ideal para gerenciar variáveis de ambiente

Pode ser acessado programaticamente pela aplicação

Pode ser acessado por outros serviços como CodeBuild, CodePipeline etc.

### CLI

`aws get-session-token` com credenciais IAM para gerar token de conexão

Utilizar opção `--page-size` para limitar output e evitar timeout

---

## Networking and Content Delivery

### API Gateway

Conectar e documentar APIs com backends usando Lambda, EC2 e outros serviços de computação da AWS.

Função nativa para criação de chaves de API. Precisam ser registradas com `RegisterUsagePlanKey`

Stages

### Elastic Load Balancing

Distribui carga de trabalho e tráfego entre vários deploys

Instâncias com defeitos ou mal funcionamento são excluídas indefinidamente do tráfego

Sempre vai dar preferência ao Access Zone com menos instâncias

Mantém lista de redirecionamentos no header `X-Forwarded-For`

### CloudFront

CDN (*Content Delivery Network*)

Pode invocar Lambda@Edge

### Route 53

DNS (*Domain Name Service*)

### VPC (*Virtual Private Cloud*)

Conexão entre serviços privados da sua arquitetura

Um ***security group*** é responsável por controlar o tráfego que entra e sai da arquitetura

Cria uma conexão direta (***direct connect***) ou utilizar um VPN para acessar aparelhos na VPC

***Flow Log*** pode ser usado para ter registro dos IPs de requisições

---

## Identidade e Compliance

### IAM

Controle de acesso a recursos da nuvem

Autenticação através de ID e *access key*

Login: `https://{aws_account.id}.signin.aws.amazon.com/console/`

Não configura acesso a VPCs

Não é possível incluir usuários externos a *user groups*

Não há cobrança por número de contas de usuário IAM

Não gerencia coneção com BD, apenas acesso

### Cognito

Servidor de autenticação

Serviço de OAuth 2.0 e de credenciais AWS

***Identity Pool:*** gerencia credenciais para uso de serviços AWS

***User Pool:*** cria credenciais de usuários de aplicação do desenvolvedor

Integra de forma nativa com o ALB para gerenciar credenciais da requisição antes de entrar em contato com a aplicação.

Suporta criptografia *at-rest* e *in-transit* de forma nativa.

### KMS (Key Management System)

*Envelope encryption* = cifrar dado com chave local, cifrar chave local com chave global. Guardar dado cifrado e chave local cifrada no registro da BD.

***Simetria de chaves***

- **Chave simétrica:** mesma chave para cifrar e decifrar
- **Chave assimétrica:** cifra com chave 1, decifra com chave 2

***Governança de chaves***

- **Customer Owned:** desenvolvedor controla as chaves, podendo editá-las. Rotação personalizada. É possível chamar uma Lambda como worker para a rotação. Taxa mensal e por uso da chave além do plano gratuito.
- **AWS Managed:** AWS cria e gerencia chaves. Chaves são visíveis através do console do KMS. Cobrado pelo uso da API.
- **AWS Owned:** AWS cria e controla chaves. Impossível ver o valor da chave de forma pública. Útil em cenários de desenvolvimento, pois são gerenciadas automaticamente.

### Secrets Manager

Armazenar e rotacionar valores sigilosos, como chaves de API e senhas de BD

### WAF (*Web Application Firewall*)

Proteção contra XSS e injeção de SQL

### ACM (*AWS Certificate Manager*)

Domínio do certificado de SSL/TLS deve ser o mesmo do domínio CloudFront.

### Private Certificate Authority

Gerenciar certificados TLS para seu sistema, incluindo usuários, máquinas, aparelhos IoT e APIs

### STS (*Security Token Service*)

API+SDK para gerar credenciais temporárias para usuários da aplicação

---

## Storage

### S3 (Simple Storage Solution)

Diretório de arquivos

Mais bruto que EFS

Objetos são imutáveis

Não é especializado um fluxo constante de dados

É possível criar triggers para invocar Lambda Functions

Capaz de entregar websites estáticos. Porém, estes sites não tem acesso imediato aos serviços AWS, pois tudo acontece no cliente. É possível gerar credenciais temporárias para serviços AWS, potencialmente com Web Identity Federation ou Cognito, permitindo acesso a APIs de serviços.

Utiliza ***presigned URLs*** para compartilhar objetos de forma temporária

Podem ser configurados como host de websites estáticos. Isso gera endpoints para cada região.

- **Intelligent-Tiering:** classifica documentos de acordo com frequência de acesso → standard/IA.
- **Glacier Flexible Retrieval:** ideal para depósitos de arquivos, baixo custo, mais lento.
- **Standard-IA:** acesso infrequente mas com alta disponibilidade. IA = Infrequent Access.
- **One Zone-IA:** baixa frequência e baixa disponibilidade.

### EBS (Elastic Block Store)

Volumes persistentes que podem ser montados em uma máquina virtual

Escalonamento manual, pre-alocado

Cobrado de acordo com alocação

Suporta discos criptografados

### EFS (Elastic File Storage)

Permite volumes compartilhados entre várias instâncias virtuais

Pode ser escalonado de acordo com a demanda, atingindo petabytes

Sistema de arquivos exclusivo NFS

Cobrado de acordo com uso e taxas de transferência

## General programming stuff

- HTTP 502 = data in unexpected format
- Autenticação:
    - Header `Authorization`
    - Query string `X-Amz-Signature`

## Outros (não essenciais)

| Título | Função |
| --- | --- |
| CloudTrail | Registro de chamadas de API |
| CloudWatch | Registro de logs do sistema |
| Inspector | Revisão de source-code para garantir segurança e boas-práticas |
| Shield | Proteção contra DDoS |
| GuardDuty | Detecção de ameaças e pontos fracos na infraestrutura |
| CloudHSM | Conectar com hardware de segurança com chaves de acesso |
| *Snow* family | Importação de dados para AWS |
| Identity Federation | SSO e conexão com usuários de 3º (Google, Meta, Microsoft…)  |
| Fargate | Serviço de computação baseado em containers |
| Simple Workflow Service | nunca replica dados; máx exec = 1 ano; máx atv. abertas = 1000; lim. retenção = 90 dias |
| SageMaker | Criar e treinar modelos de IA |
| Bedrock | Consumir modelos de IA existentes |
| Lex | Criar chatbots |
| Textract | Extrair texto de imagens |
| Amazon Augmented AI) | Conectar workflow de treinamento de IA com revisão manual |
| NAT Gateway | permissão de entrada e saída de tráfego |
| Outposts | nuvem híbrida |
| Elastic Network Interface | pseudônimos de IP, grupos de segurança, redirecionamento de tráfego em caso de falhas |
| Route Table | Conectar várias *subnets* de um sistema |
| Direct Connect | ??? |
