# fiap_hackathon_doc
Repositório com a documentação para o hackathon

## Descrição da arquitetura gerada pela IA para ser revisada e alterada!!

Descrição Detalhada da Arquitetura da Aplicação
A arquitetura do sistema é fundamentada em uma série de repositórios distintos que gerenciam diferentes aspectos da aplicação. A infraestrutura base é estabelecida por um repositório chamado infra-base, responsável por provisionar os recursos essenciais na AWS, como VPCs e clusters. O banco de dados, que foi escolhido ser o MongoDB, é criado por outro repositório, o infra-bd.

O núcleo da aplicação é composto por quatro microsserviços principais:

ms-upload: Responsável por receber o arquivo enviado pelo usuário.

ms-processamento: Realiza o processamento do arquivo recebido.

frame-extractor: Extrai os frames do vídeo processado.

ms-notificacao: Envia uma notificação ao usuário via webhook ao final do processo.

Além destes, há um serviço dedicado à autenticação, que gerencia a configuração do Amazon Cognito e do Amazon API Gateway.

Fluxo de Autenticação e Acesso
O processo de interação do usuário com a API é iniciado pela autenticação:

O usuário se autentica no Amazon Cognito, que por sua vez gera um token JWT (JSON Web Token) de acesso.

Toda requisição do usuário é direcionada para o API Gateway. Este atua como a porta de entrada para os serviços, validando o token JWT antes de permitir o acesso.

Uma vez autenticado, o usuário pode interagir com os serviços expostos, como o ms-upload para enviar novos arquivos ou o ms-processamento para consultar o status de um processamento.

Fluxo de Processamento do Vídeo
O fluxo de trabalho, desde o upload do arquivo até a notificação final, ocorre da seguinte maneira:

O microsserviço ms-upload recebe o arquivo e o envia de forma fragmentada (em partes) para o ms-processamento.

O ms-processamento remonta o arquivo e faz o upload da sua versão completa para um bucket no Amazon S3.

Após o upload para o S3, a tarefa é encaminhada para o serviço frame-extractor.

O frame-extractor gera todos os frames do vídeo, faz o upload desses frames para o S3 e, em seguida, cria um arquivo zipado contendo todos eles.

Finalmente, o link para download do arquivo zipado é disponibilizado para o serviço de notificação.

O ms-notificacao envia o link de download para o usuário através de um webhook, concluindo o fluxo.

Toda a comunicação entre os microsserviços, como a passagem de tarefas do ms-upload para o ms-processamento, é realizada de forma assíncrona através de um sistema de mensageria, utilizando o RabbitMQ. Essa abordagem desacoplada representa a arquitetura escolhida para a aplicação.

<img width="1024" height="768" alt="fluxograma-hacka" src="https://github.com/user-attachments/assets/2b209a44-3702-40e9-8a91-0ab37752d821" />

