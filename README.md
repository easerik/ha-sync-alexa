# Skill Alexa para Home Assistant com Sincronização em Tempo Real via AWS

Este projeto implementa uma Skill de Smart Home para a Alexa que se integra com o Home Assistant, utilizando AWS Lambda e DynamoDB. Ele oferece uma alternativa robusta e sem custos de assinatura ao Home Assistant Cloud (Nabu Casa), com foco em sincronização em tempo real e controle total.

---

## Conteúdo
* [Versão em Português](#-versão-em-português)
* [English Version](#-english-version)

---

## 🇧🇷 Versão em Português

### ✨ Funcionalidades Principais

-   **Sincronização em Tempo Real (HA → Alexa):** Utiliza webhooks do Home Assistant para enviar `ChangeReports` proativos para a Alexa, atualizando o status dos dispositivos instantaneamente.
-   **Controle Completo por Voz (Alexa → HA):** Suporte para uma vasta gama de comandos:
    -   Ligar/Desligar (`PowerController`)
    -   Controle de Brilho (`BrightnessController`)
    -   Controle de Cor / RGB (`ColorController`)
    -   Controle de Temperatura de Cor (`ColorTemperatureController`)
    -   Controle de Persianas e Cortinas (`RangeController`, `ModeController`)
    -   Ativação de Cenas e Scripts (`SceneController`)
-   **Persistência de Tokens:** Utiliza o Amazon DynamoDB para armazenar de forma segura e permanente os tokens de autenticação dos usuários.
-   **Segurança em Camadas:** Endpoint protegido por `secret` compartilhado, verificação de headers e `rate limiting`.
-   **Descoberta Configurável:** Controle exato de quais dispositivos são expostos através de uma tag personalizada.
-   **Arquitetura Serverless:** Custo-benefício extremamente alto, operando na maioria dos casos dentro do nível gratuito da AWS.

### 📐 Arquitetura

O sistema opera em dois fluxos principais:

1.  **Controle por Voz (Alexa → HA):**
    `Comando de Voz → Alexa Service → AWS Lambda → API do Home Assistant → Dispositivo Físico`
2.  **Atualização de Status (HA → Alexa):**
    `Dispositivo Físico → Automação HA → Webhook (Lambda URL) → AWS Lambda → Alexa Service → App Alexa`

### ⚙️ Pré-requisitos

1.  Uma conta na **AWS (Amazon Web Services)**.
2.  Uma instância do **Home Assistant** acessível publicamente via HTTPS.
3.  Uma conta de **Desenvolvedor Amazon** (a mesma da sua conta Alexa).

### 🚀 Guia de Instalação e Configuração

#### Parte 1: Amazon Developer Console (Criar a Skill)

1.  Faça login no [Amazon Developer Console](https://developer.amazon.com/alexa/console/ask).
2.  Clique em **"Create Skill"**.
    -   **Nome da Skill:** Escolha um nome (ex: "Minha Casa HA").
    -   **Modelo:** Selecione **"Smart Home"**.
    -   **Método de hospedagem:** Selecione **"Provision your own"**.
3.  No menu lateral da skill, vá para **"Account Linking"**.
    -   Marque a opção **"Auth Code Grant"**.
    -   **Authorization URI:** `https://www.amazon.com/ap/oa`
    -   **Access Token URI:** `https://api.amazon.com/auth/o2/token`
    -   Anote o **Client ID** e o **Client Secret**.
    -   **Scope:** Adicione `alexa::skill_messaging`.
4.  Salve as configurações.

#### Parte 2: Configuração na AWS (IAM, DynamoDB, Lambda)

1.  **Tabela DynamoDB:**
    -   Vá para o serviço **DynamoDB**.
    -   **Nome da tabela:** `alexa-user-tokens`.
    -   **Partition key (Chave de partição):** `user_id` (do tipo `String`).
    -   Crie a tabela.

2.  **Role do IAM para a Lambda:**
    -   Vá para o serviço **IAM** e crie uma nova **Role** para o serviço **Lambda**.
    -   Anexe a política `AWSLambdaBasicExecutionRole`.
    -   Crie uma *inline policy* com o JSON abaixo para permitir o acesso ao DynamoDB, substituindo os placeholders:
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowDynamoDBAccess",
                    "Effect": "Allow",
                    "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:UpdateItem", "dynamodb:Scan"],
                    "Resource": "arn:aws:dynamodb:SUA_REGIAO:SEU_ACCOUNT_ID:table/alexa-user-tokens"
                }
            ]
        }
        ```

3.  **Função Lambda:**
    -   Vá para o serviço **Lambda** e crie uma nova função.
    -   **Runtime:** `Python 3.11` ou superior.
    -   **Role:** Use a Role criada no passo anterior.
    -   Cole o código do arquivo `lambda_function.py` deste repositório.
    -   **Timeout:** Aumente o timeout para **15 segundos** (em Configuration > General configuration).
    -   **Variáveis de Ambiente:** Adicione as seguintes variáveis:
        -   `HA_URL`: URL pública do seu Home Assistant.
        -   `HA_TOKEN`: Token de Acesso de Longa Duração do HA.
        -   `ALEXA_CLIENT_ID`: Client ID da sua skill.
        -   `ALEXA_CLIENT_SECRET`: Client Secret da sua skill.
        -   `WEBHOOK_SECRET`: Uma senha forte e longa criada por você.
        -   `DYNAMODB_TABLE`: `alexa-user-tokens`.
        -   `HA_DISCOVERY_TAG`: A tag para descobrir dispositivos (ex: `alexa_erik`). Deixe em branco para descobrir todos.
    -   **URL da Função:** Crie uma **Function URL** na aba correspondente, com tipo de autenticação `NONE` e CORS habilitado para `POST`. Anote a URL gerada.

4.  **Conectar Skill e Lambda:**
    -   Volte ao **Amazon Developer Console**, na sua skill, vá em **"Endpoint"** e cole o **ARN** da sua função Lambda na região correspondente.

#### Parte 3: Configuração no Home Assistant

1.  **Personalizar Entidades (`customize.yaml`):**
    -   Marque os dispositivos que você quer que a Alexa descubra com a tag definida em `HA_DISCOVERY_TAG`.
        ```yaml
        # em customize.yaml
        light.luz_da_sala:
          friendly_name: "Luz da Sala"
          alexa_erik: true
        ```

2.  **Definir Comando REST (`configuration.yaml`):**
    -   Adicione o serviço que fará a chamada para a sua Lambda.
        ```yaml
        # em configuration.yaml
        rest_command:
          enviar_para_alexa_lambda:
            url: "SUA_URL_DA_LAMBDA_AQUI"
            method: POST
            headers:
              Content-Type: "application/json"
              x-webhook-secret: "SUA_CHAVE_SECRETA_AQUI"
            payload: "{{ payload }}"
            timeout: 30
        ```

3.  **Criar Automação do Webhook (`automations.yaml`):**
    -   Esta automação envia as atualizações de estado para a Lambda. É recomendado usar um grupo (`groups.yaml`) para gerenciar a lista de entidades.
        ```yaml
        - alias: 'Webhook para Sincronização com Alexa'
          description: 'Envia atualizações de estado para a Lambda da Alexa'
          mode: parallel
          max: 20
          trigger:
            - platform: state
              entity_id: group.entidades_para_sincronizar_com_alexa
          action:
            - choose:
                - conditions:
                    - condition: template
                      value_template: "{{ trigger.to_state is not none and trigger.from_state is not none }}"
                  sequence:
                    - service: rest_command.enviar_para_alexa_lambda
                      data:
                        payload: >
                          {
                            "entities": [
                              {
                                "entity_id": "{{ trigger.entity_id }}",
                                "state": "{{ trigger.to_state.state }}",
                                "attributes": {{ trigger.to_state.attributes | tojson }}
                              }
                            ]
                          }
              default:
                - service: system_log.write
                  data:
                    message: "ALEXA SYNC ERROR: Gatilho com estado inválido para {{ trigger.entity_id }}."
                    level: error
        ```

### 💡 Considerações Importantes

-   **Cloudflare:** Esta arquitetura foi testada e é recomendada para uso com o **Cloudflare** atuando como um proxy reverso para o seu Home Assistant. Isso adiciona uma camada extra de segurança (WAF, proteção contra DDoS) e pode simplificar a exposição segura da sua instância.
-   **Custos (Nível Gratuito da AWS):** Os serviços da AWS utilizados (Lambda, DynamoDB) possuem um **Nível Gratuito (Free Tier)** generoso. Para um uso residencial típico, os custos de operação desta skill devem ser nulos ou muito próximos de zero, desde que os limites do Free Tier não sejam excedidos.

### 📄 Licença

Este projeto é distribuído sob a Licença MIT.

---

## 🇬🇧 English Version

### ✨ Key Features

-   **Real-Time Sync (HA → Alexa):** Utilizes Home Assistant webhooks to send proactive `ChangeReports` to Alexa, instantly updating device status.
-   **Full Voice Control (Alexa → HA):** Supports a wide range of commands:
    -   On/Off (`PowerController`)
    -   Brightness (`BrightnessController`)
    -   Color / RGB (`ColorController`)
    -   Color Temperature (Cool White, Warm White, etc.) (`ColorTemperatureController`)
    -   Blinds & Covers (`RangeController`, `ModeController`)
    -   Scenes & Scripts Activation (`SceneController`)
-   **Token Persistence:** Uses Amazon DynamoDB to securely and permanently store user authentication tokens.
-   **Layered Security:** The webhook endpoint is protected by a shared secret, header verification, and rate limiting.
-   **Configurable Discovery:** Provides precise control over which devices are exposed to Alexa via a custom tag.
-   **Serverless Architecture:** Extremely cost-effective, operating almost entirely within the AWS Free Tier for most residential use cases.

### 📐 Architecture

The system operates in two main flows:

1.  **Voice Control (Alexa → HA):**
    `Voice Command → Alexa Service → AWS Lambda (Directive Handler) → Home Assistant API → Physical Device`
2.  **Status Update (HA → Alexa):**
    `Physical Device → HA Automation → Webhook (Lambda URL) → AWS Lambda (ChangeReport Handler) → Alexa Service → Alexa App`

### ⚙️ Prerequisites

1.  An **AWS (Amazon Web Services)** account.
2.  A **Home Assistant** instance publicly accessible via HTTPS.
3.  An **Amazon Developer** account (same as your Alexa account).

### 🚀 Installation and Setup Guide

#### Part 1: Amazon Developer Console (Create the Skill)

1.  Log in to the [Amazon Developer Console](https://developer.amazon.com/alexa/console/ask).
2.  Click **"Create Skill"**.
    -   **Skill Name:** Choose a name (e.g., "My HA Home").
    -   **Model:** Select **"Smart Home"**.
    -   **Hosting Method:** Select **"Provision your own"**.
3.  In the skill's side menu, go to **"Account Linking"**.
    -   Select **"Auth Code Grant"**.
    -   **Authorization URI:** `https://www.amazon.com/ap/oa`
    -   **Access Token URI:** `https://api.amazon.com/auth/o2/token`
    -   Note down the **Client ID** and **Client Secret**.
    -   **Scope:** Add `alexa::skill_messaging`.
4.  Save the configuration.

#### Part 2: AWS Setup (IAM, DynamoDB, & Lambda)

1.  **DynamoDB Table:**
    -   Go to the **DynamoDB** service.
    -   **Table name:** `alexa-user-tokens`.
    -   **Partition key:** `user_id` (Type: `String`).
    -   Create the table.

2.  **IAM Role for Lambda:**
    -   Go to the **IAM** service and create a new **Role** for the **Lambda** service.
    -   Attach the `AWSLambdaBasicExecutionRole` policy.
    -   Create an *inline policy* with the JSON below to allow access to the DynamoDB table, replacing the placeholders:
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowDynamoDBAccess",
                    "Effect": "Allow",
                    "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:UpdateItem", "dynamodb:Scan"],
                    "Resource": "arn:aws:dynamodb:YOUR_REGION:YOUR_ACCOUNT_ID:table/alexa-user-tokens"
                }
            ]
        }
        ```

3.  **Lambda Function:**
    -   Go to the **Lambda** service and create a new function.
    -   **Runtime:** `Python 3.11` or newer.
    -   **Role:** Use the IAM Role created in the previous step.
    -   Paste the full source code from the `lambda_function.py` file in this repository.
    -   **Timeout:** Increase the timeout to **15 seconds** (in Configuration > General configuration).
    -   **Environment Variables:** Add the following variables:
        -   `HA_URL`: Your public Home Assistant URL.
        -   `HA_TOKEN`: A Long-Lived Access Token from your HA profile.
        -   `ALEXA_CLIENT_ID`: Your skill's Client ID.
        -   `ALEXA_CLIENT_SECRET`: Your skill's Client Secret.
        -   `WEBHOOK_SECRET`: A strong, long password you create.
        -   `DYNAMODB_TABLE`: `alexa-user-tokens`.
        -   `HA_DISCOVERY_TAG`: The tag for filtering devices (e.g., `alexa_erik`). Leave blank to discover all.
    -   **Function URL:** Create a **Function URL** with Auth type `NONE` and CORS enabled for `POST` requests. Note the generated URL.

4.  **Connect Skill and Lambda:**
    -   Go back to the **Amazon Developer Console**, navigate to your skill's **"Endpoint"** section, and paste the **ARN** of your Lambda function.

#### Part 3: Home Assistant Configuration

1.  **Customize Entities (`customize.yaml`):**
    -   Mark the devices you want Alexa to discover with the tag defined in `HA_DISCOVERY_TAG`.
        ```yaml
        # in customize.yaml
        light.living_room_light:
          friendly_name: "Living Room Light"
          alexa_erik: true
        ```

2.  **Define REST Command (`configuration.yaml`):**
    -   Add the service that will call your Lambda URL.
        ```yaml
        # in configuration.yaml
        rest_command:
          enviar_para_alexa_lambda:
            url: "YOUR_LAMBDA_FUNCTION_URL_HERE"
            method: POST
            headers:
              Content-Type: "application/json"
              x-webhook-secret: "YOUR_SECRET_KEY_HERE"
            payload: "{{ payload }}"
            timeout: 30
        ```
    > **Replace `YOUR_LAMBDA_FUNCTION_URL_HERE` and `YOUR_SECRET_KEY_HERE` with your values.**

3.  **Create Webhook Automation (`automations.yaml`):**
    -   This automation sends state updates to Lambda. Using a group is recommended to manage the entity list.
        ```yaml
        - alias: 'Webhook for Alexa Sync'
          description: 'Sends state updates to the Alexa Lambda'
          mode: parallel
          max: 20
          trigger:
            - platform: state
              # RECOMMENDED: Use a group to easily manage entities.
              entity_id: group.alexa_sync_entities
          action:
            - choose:
                - conditions:
                    - condition: template
                      value_template: "{{ trigger.to_state is not none and trigger.from_state is not none }}"
                  sequence:
                    - service: rest_command.enviar_para_alexa_lambda
                      data:
                        payload: >
                          {
                            "entities": [
                              {
                                "entity_id": "{{ trigger.entity_id }}",
                                "state": "{{ trigger.to_state.state }}",
                                "attributes": {{ trigger.to_state.attributes | tojson }}
                              }
                            ]
                          }
              default:
                - service: system_log.write
                  data:
                    message: "ALEXA SYNC ERROR: Trigger has an invalid state for {{ trigger.entity_id }}."
                    level: error
        ```

### 💡 Important Considerations

-   **Cloudflare:** This architecture was tested and is recommended for use with **Cloudflare** acting as a reverse proxy for your Home Assistant instance. This adds an extra layer of security (WAF, DDoS protection) and can simplify secure exposure of your instance.
-   **Costs (AWS Free Tier):** The AWS services used (Lambda, DynamoDB) have a generous **Free Tier**. For typical residential use, the operational costs for this skill should be zero or very close to zero, provided the Free Tier limits are not exceeded.

### 📄 License

This project is distributed under the MIT License.
