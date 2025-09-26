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


