# Documentação Hackathon
Repositório com a documentação para o hackathon

<details>
  <summary>Visão Geral da Arquitetura</summary>

# Descrição da arquitetura

## Visão Geral da Arquitetura

A arquitetura do projeto foi desenhada para ser robusta e escalável, utilizando uma abordagem de microserviços e infraestrutura como código com Terraform. A solução é composta por diferentes repositórios que gerenciam desde a infraestrutura base até a lógica de cada serviço.

## Componentes da Arquitetura
A solução é dividida nos seguintes componentes principais:

## Repositórios Terraform:

Infraestrutura Base: Responsável por provisionar a infraestrutura essencial, como as VPCs (Virtual Private Clouds) e o cluster Kubernetes.

Infraestrutura do Banco de Dados: Encarregado de criar a instância do banco de dados MongoDB, que foi a tecnologia escolhida para a persistência dos dados. Como o banco de dados escolhido foi o MongoDB não há a necessidade de scripts relacionados ao banco de dados.

## Microserviços:

* MS Upload: Responsável por receber os arquivos enviados pelos usuários.


* MS Frame Extractor: Processa os arquivos de vídeo, extraindo os frames, de acordo com a frequência determinada no arquivo application.properties do microserviço (propriedade file.processing.frame.interval=7).  Atualmente, as extrações são realizadas a cada 7 segundos.


* MS Processamento: Orquestra o armazenamento do arquivo completo no Amazon S3 e persiste o usuário que fez o upload e a url do arquivo compactado e do arquivo completo armazenados na S3, no banco de dados MongoDB.


* MS Notificação: Envia notificações sobre o status do processamento (sucesso ou erro) para o usuário através de webhooks.


## Autenticação:

AWS Cognito: Utilizado para gerenciar a autenticação dos usuários, gerando um token JWT (JSON Web Token) após o login.

AWS API Gateway: Atua como um ponto de entrada para as requisições, validando o token JWT gerado pelo Cognito antes de autorizar o acesso aos serviços.

## Fluxo de Funcionamento

Autenticação: O usuário se autentica no AWS Cognito, que gera um token JWT.

Requisição e Validação: Toda requisição é enviada para o AWS API Gateway, que valida a autenticidade do token JWT. Se o token for inválido, o acesso é negado.

Roteamento: O API Gateway expõe endpoints para os microserviços de Upload e Processamento. Os demais serviços (Frame Extractor e Notificação) não são expostos publicamente, comunicando-se apenas internamente.


A estratégia adotada para a extração dos frames do vídeo, foi realizar a extração dos frames, compactação do arquivo e armazenamento na Amazon S3 sob demanda, em pedaços (chunks).

Com a abordagem de processamento sob demanda, não é necessário nenhum tipo de cache dos dados e, nem é preciso receber o arquivo inteiro para depois realizar o processamento. Dessa forma, é possível realizar o processamento simultâneo de mais arquivos com a menor quantidade de recursos possível.

Segue abaixo o diagrama do fluxo descrito acima.

<img width="1024" height="768" alt="fluxograma-hacka-novo" src="https://github.com/user-attachments/assets/ca263541-9260-4998-85a5-fbc9856b3d7c" />


## Comunicação entre Serviços:

A comunicação entre os serviços Upload, Frame Extractor e Processamento é assíncrona, realizada através de mensageria com RabbitMQ.

Já a comunicação com o MS Notificação é feita de forma síncrona, através de requisições REST (HTTP) utilizando OpenFeign, a partir dos serviços Frame Extractor e Processamento.


## Processo de Upload:

O usuário envia um arquivo de vídeo para o MS Upload.

Este serviço divide o arquivo em partes menores ("chunks") de 1 MB. A fragmentação do arquivo otimiza o processamento, permitindo o envio de múltiplos arquivos simultaneamente e eliminando a necessidade de um limite para o tamanho do upload.

Os chunks do arquivo são enviados, via mensageria (RabbitMQ), para o MS Frame Extractor, identificando a qual arquivo cada chunk pertence. Para controle de reprocessamento e conclusão do processo, esse serviço indica se o chunk é o primeiro ou o último do arquivo.


## Frame Extractor:

O MS Frame Extractor consome as mensagens da fila, junta as partes do arquivo e processa os frames do vídeo.  Ao receber os chunks, este serviço realiza a extração dos frames a partir dos chunks, depois inicia a compactação e o envio do arquivo compactado para o Amazon S3 (sob demanda), e depois posta este mesmo chunk na fila MS Processamento.

Em caso de falha, é enviada uma mensagem para o MS Notificação para informar o usuário de que o processamento falhou e, por último, posta o chunk processado no momento da falha, na fila do MS Processamento, informando que ocorreu uma falha. Os chunks posteriores recebidos, em caso de falha, serão ignorados. Em caso de reprocessamento, por parte do usuário, ao receber o chunk inicial, re-inicia todo o fluxo.


## Processamento:

O MS Processamento recebe o arquivo completo, envia para armazenamento no Amazon S3 e grava os dados do upload no MongoDB.

Ao receber os chunks, o MS Processamento inicia a compactação do arquivo original e, em seguida, o envio do arquivo compactado para a Amazon S3. Ao receber o chunk final do arquivo, ele finaliza a compactação e o armazenamento do arquivo na Amazon S3. Em caso de recebimento de um chunk informando que houve falha durante a extração dos frames, ele remove o que já foi armazenado do arquivo zip na S3 e ignora os demais chunks, identificando o reprocessamento quando é recebido o chunk inicial.

Se ocorrer uma falha durante a compactação ou envio deste arquivo para a Amazon S3, é enviada uma mensagem para o MS Notificação informar o usuário e ignora os demais arquivos recebidos e informa o MS Frame Extractor para remover o arquivo armazenado previamente.


## Notificação:

Em caso de sucesso ou erro no processamento, o MS Notificação é acionado para informar o usuário através de um serviço de webhook.

</details>

<details>
  <summary>Detalhamento execução do projeto</summary>

## 👟 Passos para o provisionamento
Este projeto possui um ecossistema composto por múltiplos repositórios que se comunicam entre si e também utilizam GitHub Actions para provisionamento ou deploy automatizado.

> Para completo funcionamento da plataforma, é necessário seguir o seguinte fluxo de provisionamento:
> 1. A provisão deste repositório; [infra-base](https://github.com/ns-fiap-tc/fiap_hackathon_infra_base)
> 2. A provisão do repositório do banco de dados: [infra-bd](https://github.com/ns-fiap-tc/fiap_hackathon_infra_bd);
> 3. A provisão do repositório do microsserviço de upload: [fiap_hackathon_ms_upload](https://github.com/ns-fiap-tc/fiap_hackathon_ms_upload);
> 4. A provisão do repositório do microsserviço de notificação: [fiap_hackathon_ms_notificacao](https://github.com/ns-fiap-tc/fiap_hackathon_ms_notificacao);
> 5. A provisão do repositório do microsserviço de processamento: [fiap_hackathon_ms_processamento](https://github.com/ns-fiap-tc/fiap_hackathon_ms_processamento);
> 6. A provisão do repositório do microsserviço de extração de frames: [fiap_hackathon_ms_frameextractor](https://github.com/ns-fiap-tc/fiap_hackathon_ms_frameextractor);
> 7. A provisão do repositório para autenticação com cognito e api gateway: [fiap_hackathon_autenticacao](https://github.com/ns-fiap-tc/fiap_hackathon_autenticacao);

## 🚀 Como rodar o projeto

### 💻 Localmente

<details>
  <summary>Passo a passo</summary>

#### Pré-requisitos

Antes de começar, certifique-se de ter os seguintes itens instalados e configurados em seu ambiente:

1. **Terraform**: A ferramenta que permite definir, visualizar e implantar a infraestrutura de nuvem.
2. **AWS CLI**: A interface de linha de comando da AWS.
3. **Credenciais AWS válidas**: Você precisará de uma chave de acesso e uma chave secreta para autenticar com a AWS (no momento, o repositório usa chaves e credenciais fornecidas pelo [AWS Academy](https://awsacademy.instructure.com/) e que divergem de contas padrão). Tais credenciais devem ser inseridas no arquivo `credentials` que fica dentro da pasta `.aws`
4. **Bucket S3 criado na AWS convencional (que não seja na aws academy)**: Você precisará de uma chave de acesso e uma chave secreta para autenticar com a AWS e conectar ao S3. Tal abordagem foi necessária pois a AWS academy não permite a criação de roles e isso inviabilizou a comunicação dos serviços rodando no eks com o S3 da AWS academy. Com isso a solução foi criar um bucket com uma role específica para ele em um conta convencional da AWS 

## Como usar(Seguir os passos abaixo para cada repositório mencionado anteriormente obedecendo a ordem correta)

1. **Clonar cada repositório mencionado acima, por exemplo**:

```bash
git clone https://github.com/ns-fiap-tc/fiap_hackathon_ms_upload
```

2. **Acesse o diretório do repositório clonado, por exemplo**:

```bash
cd fiap_hackathon_ms_upload
```

3. **Defina as variáveis necessárias ao nível de ambiente, criando um arquivo `.env` de acordo com o arquivo contido em cada repositório `.env.exemplo`. Exemplo:**:

```bash
DOCKERHUB_USERNAME="dockerhub_username"
DOCKERHUB_ACCESS_TOKEN="dokerhub_token"
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

7. **Para destruir a infraestrutura provisionada**:

```bash
./terraform.sh destroy -auto-approve
```
</details>
</details>

## ✨ Contribuidores

- Guilherme Fausto - RM 359909
- Nicolas Silva - RM 360621
- Rodrigo Medda Pereira - RM 360575

## Licença

[![Licence](https://img.shields.io/github/license/Ileriayo/markdown-badges?style=for-the-badge)](./LICENSE)
