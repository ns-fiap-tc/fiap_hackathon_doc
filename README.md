# fiap_hackathon_doc
Repositório com a documentação para o hackathon

<details>
  <summary>Visão Geral da Arquitetura !!! PARA SER ALTERADA !!!</summary>

## Descrição da arquitetura gerada pela IA para ser revisada e alterada!!

# Visão Geral da Arquitetura

A arquitetura do projeto foi desenhada para ser robusta e escalável, utilizando uma abordagem de microserviços e infraestrutura como código com Terraform. A solução é composta por diferentes repositórios que gerenciam desde a infraestrutura base até a lógica de cada serviço.

# Componentes da Arquitetura
A solução é dividida nos seguintes componentes principais:

# Repositórios Terraform:

Infraestrutura Base: Responsável por provisionar a infraestrutura essencial, como as VPCs (Virtual Private Clouds) e o cluster Kubernetes.

Infraestrutura do Banco de Dados: Encarregado de criar a instância do banco de dados MongoDB, que foi a tecnologia escolhida para a persistência dos dados.

# Microserviços:

MS Upload: Responsável por receber os arquivos enviados pelos usuários.

MS Frame Extractor: Processa os arquivos de vídeo, extraindo os frames.

MS Processamento: Orquestra o armazenamento do arquivo completo no Amazon S3 e persiste os metadados do upload e dos frames no banco de dados.

MS Notificação: Envia notificações sobre o status do processamento (sucesso ou erro) para o usuário através de webhooks.

# Autenticação:

AWS Cognito: Utilizado para gerenciar a autenticação dos usuários, gerando um token JWT (JSON Web Token) após o login.

AWS API Gateway: Atua como um ponto de entrada para as requisições, validando o token JWT gerado pelo Cognito antes de autorizar o acesso aos serviços.

# Fluxo de Funcionamento
Autenticação: O usuário se autentica no AWS Cognito, que gera um token JWT.

# Requisição e Validação: Toda requisição é enviada para o AWS API Gateway, que valida a autenticidade do token JWT. Se o token for inválido, o acesso é negado.

# Roteamento: O API Gateway expõe endpoints para os microserviços de Upload e Processamento. Os demais serviços (Frame Extractor e Notificação) não são expostos publicamente, comunicando-se apenas internamente.

# Processo de Upload:

O usuário envia um arquivo de vídeo para o MS Upload.

Este serviço divide o arquivo em partes menores ("chunks") de 1 MB. A fragmentação do arquivo otimiza o processamento, permitindo o envio de múltiplos arquivos simultaneamente e eliminando a necessidade de um limite para o tamanho do upload.

As partes do arquivo são enviadas via mensageria (RabbitMQ) para o MS Frame Extractor.

# Extração e Processamento:

O MS Frame Extractor consome as mensagens da fila, junta as partes do arquivo e processa os frames do vídeo.

O MS Processamento recebe o arquivo completo, envia para armazenamento no Amazon S3 e grava os dados do upload no MongoDB.

# Comunicação entre Serviços:

A comunicação entre os serviços Upload, Frame Extractor e Processamento é assíncrona, realizada através de mensageria com RabbitMQ.

Já a comunicação com o MS Notificação é feita de forma síncrona, através de requisições REST (HTTP) utilizando OpenFeign, a partir dos serviços Frame Extractor e Processamento.

# Notificação: Em caso de sucesso ou erro no processamento, o MS Notificação é acionado para informar o usuário através de um serviço de webhook.

<img width="1024" height="768" alt="fluxograma-hacka-novo" src="https://github.com/user-attachments/assets/ca263541-9260-4998-85a5-fbc9856b3d7c" />
</details>

<details>
  <summary>Como rodar o projeto</summary>

## 🚀 Como rodar o projeto

### 🤖 Via GitHub Actions
<details>
  <summary>Passo a passo</summary>

#### 📖 Resumo
Este repositório possui uma pipeline automatizada chamada `Terraform Deploy` que permite **provisionar a aplicação do microsserviço de categoria** sempre que houver um push na branch `main`.

A branch é protegida e só aceita alterações que venham de PRs previamente aprovadas.

> ⚠️ Apenas usuários com acesso ao repositório e às **GitHub Secrets** corretas conseguem utilizar esse fluxo.

#### 🔐 Pré-requisitos
Certifique-se de que as seguintes **secrets** estejam configuradas no repositório do GitHub (`Settings > Secrets and variables > Actions`):
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN` *(se estiver usando AWS Academy)*
- `TF_VAR_DB_USERNAME`
- `TF_VAR_DB_PASSWORD`

Essas variáveis são utilizadas pelo Terraform para autenticação e execução dos planos na AWS.

#### ⚙️ Etapas da pipeline `Terraform Deploy`
1. 🧾 **Checkout do código**: A action clona este repositório.
2. ⚒️ **Setup do Terraform**: Instala a ferramenta na máquina runner.
3. 📂 **Acesso ao diretório atual**: Todos os arquivos `.tf` são lidos da raiz do repositório.
4. 🔐 **Carregamento das variáveis sensíveis** via secrets.
5. 🧪 **Execução do `terraform init`**: Inicializa o backend e os providers.
6. 🚀 **Execução do `terraform apply`**: Cria ou atualiza a instância de banco de dados no Amazon RDS.

#### 🧭 Diagrama do fluxo

```mermaid
flowchart TD
    G[Push na branch main] --> A[Workflow: Terraform Deploy]

    subgraph Pipeline
        A1[Checkout do código]
        A2[Setup do Terraform]
        A3[Carrega Secrets da AWS]
        A4[terraform init]
        A5[terraform plan]
        A6[terraform apply]
    end

    A --> A1 --> A2 --> A3 --> A4 --> A5 --> A6 --> Container EKS[Instância EKS no AWS rodando a aplicação]
```

#### Benefícios desse fluxo
- 🤖 Automatização do deploy do banco de dados
- ✅ Redução de erros manuais
- 🔐 Segurança no uso de credenciais via GitHub Secrets
- 🔁 Reprodutibilidade garantida
- 💬 Transparência nos logs via GitHub Actions

</details>

### 💻 Localmente

<details>
  <summary>Passo a passo</summary>

#### Pré-requisitos

Antes de começar, certifique-se de ter os seguintes itens instalados e configurados em seu ambiente:

1. **Terraform**: A ferramenta que permite definir, visualizar e implantar a infraestrutura de nuvem.
2. **AWS CLI**: A interface de linha de comando da AWS.
3. **Credenciais AWS válidas**: Você precisará de uma chave de acesso e uma chave secreta para autenticar com a AWS (no momento, o repositório usa chaves e credenciais fornecidas pelo [AWS Academy](https://awsacademy.instructure.com/) e que divergem de contas padrão). Tais credenciais devem ser inseridas no arquivo `credentials` que fica dentro da pasta `.aws`

## Como usar

1. **Clone este repositório**:

```bash
git clone https://github.com/ns-fiap-tc/tech_challenge_fiap_ms_categoria
```

2. **Acesse o diretório do repositório**:

```bash
cd tech_challenge_fiap_ms_categoria
```

3. **Defina as variáveis necessárias ao nível de ambiente, criando um arquivo `.env` de acordo com o arquivo `.env.exemplo`. Exemplo:**:

```bash
DB_CATEGORIA_NAME="lanch_cat_db"
DB_CATEGORIA_USERNAME="lanch_cat_user"
DB_CATEGORIA_PASSWORD="lanchcatpass"
DB_CATEGORIA_PORT="5432"
DB_CATEGORIA_IDENTIFIER="lanchonete-categoria-db"
DOCKERHUB_USERNAME="meuusuariodockerhub"
DOCKERHUB_ACCESS_TOKEN=
```

4. **Inicialize o diretório Terraform**:

```bash
terraform init
```

5. **Visualize as mudanças que serão feitas**:

```bash
./terraform.sh plan
```

6. **Provisione a infraestrutura**:

```bash
./terraform.sh apply -auto-approve
```

7. **Para destruir a aplicação provisionada**:

```bash
./terraform.sh destroy -auto-approve
```

</details>

## ✨ Contribuidores

- Guilherme Fausto - RM 359909
- Nicolas Silva - RM 360621
- Rodrigo Medda Pereira - RM 360575


