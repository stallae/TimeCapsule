
## **Requisitos Detalhados da API**

### 1. **Armazenamento de Mensagens, Vídeos e Imagens**

Este endpoint é responsável por receber o conteúdo que o usuário deseja enviar no futuro (seja uma mensagem de texto, vídeo ou imagem). O objetivo é garantir que arquivos grandes sejam armazenados de forma segura e eficiente, e que os dados de envio sejam processados corretamente.

#### **Endpoint:** `POST /messages`

- **Descrição:** Armazena uma nova mensagem (texto, vídeo ou imagem) para ser enviada posteriormente em uma data específica.
  
- **Arquitetura**:
  - **Armazenamento de Arquivos (Bucket)**:
    - Caso o conteúdo seja um vídeo ou imagem, o arquivo não será armazenado diretamente no banco de dados, mas sim em um bucket , no **Amazon S3**. A URL pública ou privada para acessar o arquivo será armazenada no banco de dados.
    - O bucket deve ser configurado para permitir que o acesso aos arquivos seja seguro e controlado. Arquivos que ainda não foram enviados devem ter permissões restritas, enquanto os arquivos enviados podem ser temporariamente disponibilizados aos destinatários.
  - **Banco de Dados**:
    - O banco de dados armazenará informações básicas como o ID da mensagem, a data de envio agendada e os dados de quem a receberá (caso seja um familiar, por exemplo). Este banco pode ser um **PostgreSQL** ou **MongoDB**, dependendo da estrutura de dados necessária.

- **Parâmetros:**
  - `user_id` (string) — Identificador único do usuário que enviou a mensagem.
  - `content_type` (string) — Tipo de conteúdo enviado (texto, vídeo, imagem).
  - `content` (string | file) — Mensagem de texto ou upload de vídeo/imagem.
    - Se o `content` for um arquivo (vídeo/imagem), o serviço deve fazer o upload do arquivo para o bucket.
    - Após o upload, retornar a URL gerada pelo bucket e armazená-la no banco de dados.
  - `recipient_email` (string) — Email do destinatário (ou destinatários).
  - `send_date` (date) — Data em que a mensagem deve ser enviada.
  - `trigger_on_death` (boolean) — Se `true`, a mensagem será enviada automaticamente caso o usuário não passe na prova de vida.
  - `paid` (boolean) — Indica se a mensagem já foi paga ou ainda está aguardando confirmação de pagamento.

- **Fluxo de Processamento**:
  1. **Upload de Arquivo:** Se o conteúdo for um arquivo, ele será enviado diretamente para o bucket (Amazon S3, Google Cloud Storage). Retorna uma URL que será armazenada no banco de dados.
  2. **Armazenamento de Dados:** O restante das informações (ex: `user_id`, `recipient_email`, `send_date`, `trigger_on_death`, `paid`) será armazenado em um banco de dados.
  3. **Retorno:** O sistema retorna o ID da mensagem criada e a URL do arquivo (se aplicável).

- **Retorno:** Confirmação do armazenamento da mensagem com os detalhes da mensagem, incluindo o `message_id` e o status do upload do arquivo (se for o caso).

#### **Exemplo de Resposta:**
```json
{
  "message_id": "msg_123456",
  "content_url": "https://bucket.s3.amazonaws.com/uploads/video.mp4",
  "status": "pending_payment",
  "send_date": "2025-12-25",
  "recipient_email": "exemplo@email.com"
}
```

### 2. **Webhook de Confirmação de Pagamento (Stripe)**

Esse webhook receberá a confirmação de que o pagamento foi realizado com sucesso e, com base nisso, marcará a mensagem como "paga" no banco de dados, permitindo que ela seja agendada para envio.

#### **Endpoint:** `POST /payment/webhook`

- **Descrição:** Recebe as notificações de eventos da Stripe, especialmente a confirmação de que o pagamento foi realizado, e atualiza o status da mensagem para "paga".

- **Fluxo de Processamento**:
  1. **Recepção do Webhook:**
     - A Stripe enviará um evento contendo informações sobre a transação (ID da transação, status, e metadados).
     - No webhook, o sistema deve verificar a assinatura da Stripe para garantir a autenticidade do evento.
  
  2. **Atualização de Status no Banco de Dados:**
     - Se o evento de pagamento for de sucesso (ex: `payment_succeeded`), o sistema deve procurar a mensagem vinculada ao pagamento (provavelmente pelo `message_id` armazenado como metadata na Stripe) e atualizar o campo `paid` para `true`.
     - Se o pagamento falhar ou for cancelado, o status da mensagem deve permanecer como "aguardando pagamento" (`pending_payment`).

- **Parâmetros:** Dados do webhook da Stripe, incluindo:
  - `event_type` (string) — Tipo de evento (por exemplo, `payment_succeeded`, `payment_failed`).
  - `data.object.metadata.message_id` (string) — ID da mensagem para a qual o pagamento foi realizado.
  - `data.object.status` (string) — Status do pagamento.

- **Retorno:** Resposta de status `200 OK` para confirmar o recebimento do webhook.

#### **Exemplo de Webhook Payload da Stripe:**
```json
{
  "event_type": "payment_succeeded",
  "data": {
    "object": {
      "status": "succeeded",
      "metadata": {
        "message_id": "msg_123456"
      }
    }
  }
}
```

#### **Exemplo de Resposta da API:**
```json
{
  "message_id": "msg_123456",
  "status": "payment_confirmed"
}
```

### 3. **Prova de Vida e Envio de Mensagens**

Esse mecanismo é essencial para mensagens que são disparadas em caso de morte ou falha na prova de vida. O sistema vai operar de maneira regular (como um cron job ou serviço separado) que envia uma requisição para os usuários pedindo que confirmem sua "prova de vida" em intervalos configuráveis.

#### **Endpoint:** `POST /life-proof`

- **Descrição:** Gera e envia a prova de vida (via SMS ou email), solicitando uma ação do usuário para garantir que ele ainda está ativo.

- **Parâmetros:**
  - `user_id` (string) — ID do usuário a quem será enviada a prova de vida.
  - `method` (string) — Método de envio (SMS, email).
  
- **Fluxo de Processamento:**
  1. **Geração de Token de Prova de Vida:** Um token único será gerado e associado ao usuário no banco de dados. Este token será enviado para o método especificado (SMS ou email).
  2. **Envio de SMS/Email:** O sistema enviará o token, pedindo ao usuário para clicar no link ou responder ao SMS para confirmar sua prova de vida.

#### **Endpoint:** `POST /confirm-life-proof`

- **Descrição:** Confirma a prova de vida do usuário com base no token enviado.

- **Parâmetros:**
  - `token` (string) — Token enviado via email ou SMS para validação.
  
- **Fluxo de Processamento:**
  1. **Verificação de Token:** O sistema verifica se o token é válido e corresponde ao usuário.
  2. **Atualização de Status:** Se o token for válido, a prova de vida é confirmada e o cronograma de mensagens continua normalmente.

### 4. **Arquitetura Geral da API**

#### **Banco de Dados**:
- **PostgreSQL** ou MYSQL:
  - **Tabelas/coleções**:
    - **Users**: para gerenciar informações básicas dos usuários.
    - **Messages**: contém os dados das mensagens, como texto, URL dos arquivos armazenados no bucket, data de envio, destinatário, status de pagamento, e informações sobre a prova de vida.
    - **Payment**: informações sobre as transações e status do pagamento para cada mensagem.

#### **Armazenamento de Arquivos (Bucket)**:
- **Amazon S3**:
  - Utilizado para armazenar vídeos e imagens.

#### **Containers (Docker)**:
- O ambiente da aplicação será orquestrado com **Docker** e **Docker Compose**, gerenciando os seguintes serviços:
  - **API** (backend principal).
  - **Banco de Dados** (PostgreSQL ).
  - **Serviço de Armazenamento** (conexão com S3). DOCKERFILE
  - **Serviço de Background (Job de Prova de Vida)**: Um serviço separado que roda periodicamente para verificar e enviar a prova de vida. [ JOB ]
