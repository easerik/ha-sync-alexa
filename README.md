# Skill Alexa para Home Assistant com Sincronização em Tempo Real via AWS
### _Alexa Skill for Home Assistant with Real-Time Sync via AWS_
### 🇧🇷 Versão em Português
Este projeto implementa uma Skill de Smart Home para a Alexa que se integra com o Home Assistant, utilizando AWS Lambda e DynamoDB. Ele oferece uma alternativa robusta e sem custos de assinatura ao Home Assistant Cloud (Nabu Casa), com foco em sincronização em tempo real e controle total.
---
### 🇬🇧 English Version
This project implements an Alexa Smart Home Skill that integrates with Home Assistant, using AWS Lambda and DynamoDB. It offers a robust, subscription-free alternative to the official Home Assistant Cloud (Nabu Casa), with a focus on real-time synchronization and full control.
---
## Conteúdo / Contents
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
![Diagrama da Arquitetura](https://github.com/easerik/ha-sync-alexa/blob/main/ha-alexa.gif)
> **Aviso Importante:** A URL do GIF acima parece ser um link temporário e pode expirar. Para uma solução permanente, é altamente recomendado fazer o upload do arquivo GIF para o seu próprio repositório no GitHub e usar o link direto gerado por ele.
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
3.  Salve as configurações.

#### Parte 2: Configuração na AWS (IAM, DynamoDB, Lambda)
1.  **Tabela DynamoDB:**
    -   Vá para o serviço **DynamoDB**.
    -   **Nome da tabela:** `alexa-user-tokens`.
    -   **Partition key (Chave de partição):** `user_id` (do tipo `String`).
    -   Crie a tabela.
2.  **Role do IAM para a Lambda:**
    -   Vá para o serviço **IAM** e crie uma nova **Role** para o serviço **Lambda**.
    -   Anexe a política `AWSLambdaBasicExecutionRole`.
    -   Para dar permissão ao DynamoDB, escolha uma das opções abaixo:
        

        Opção 1 (Recomendada - Mais Segura)
        Crie uma *inline policy* com o JSON abaixo para permitir acesso apenas à tabela específica, substituindo os placeholders:
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
        
        

        Opção 2 (Mais Simples)
        Anexe uma política já existente da AWS. Esta abordagem é mais fácil, mas menos segura, pois concede permissão total a todas as suas tabelas DynamoDB.
        1. Na sua Role, clique em **Add permissions -> Attach policies**.
        2. Busque pela política `AmazonDynamoDBFullAccess`.
        3. Selecione-a e clique em **Add permissions**.
        
3.  **Função Lambda:**
    -   Vá para o serviço **Lambda** e crie uma nova função.
    -   **Runtime:** `Python 3.11` ou superior.
    -   **Role:** Use a Role criada no passo anterior.
    -   Cole o código-fonte completo do arquivo `lambda_function.py` deste repositório.
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
        homeassistant:
          customize:
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
    > **Substitua `SUA_URL_DA_LAMBDA_AQUI` e `SUA_CHAVE_SECRETA_AQUI` pelos seus valores.**
3.  **Criar Automação do Webhook (`automations.yaml`):**
    -   Esta automação envia as atualizações de estado para a Lambda. É recomendado usar um grupo (`groups.yaml`) para gerenciar a lista de entidades.
        ```yaml
        alias: WEBHOOK-SYNC-ALEXA
        description: Webhook Alexa - apenas logs de erro
        trigger:
          - platform: state
            entity_id:
              - light.pendente
              - light.spot_1
              - light.spot_2
              - light.spot_3
              - light.spot_4
              - light.spot_5
              - light.spot_6
              - light.spot_7
              - light.spot_8
              - light.cabeceira
              - light.cortineiro
              - light.luz_do_banheiro
              - light.sonoff_1001138f26
              - light.sonoff_100123e5f9
              - cover.cortina_1_invertida
              - cover.cortina_2_invertida
        condition: []
        action:
          - variables:
              alexa_payload:
                entities:
                  - entity_id: "{{ trigger.entity_id }}"
                    state: "{{ trigger.to_state.state }}"
                    brightness: "{{ trigger.to_state.attributes.brightness | default(null) }}"
                    current_position: "{{ trigger.to_state.attributes.current_position | default(null) }}"
                timestamp: "{{ now().isoformat() }}"
                source: home_assistant
                trigger_entity: "{{ trigger.entity_id }}"
                change_id: "{{ trigger.entity_id }}_{{ now().timestamp() | int }}"
          - choose:
              - conditions:
                  - condition: template
                    value_template: >-
                      {{ trigger.to_state is not none and trigger.from_state is not none
                      }}
                sequence:
                  - data:
                      payload: "{{ alexa_payload | tojson }}"
                    continue_on_error: true
                    service: rest_command.enviar_para_alexa_lambda
              default:
                - data:
                    message: >-
                      ALEXA ERROR: Estado inválido para {{ trigger.entity_id }}. 
                      to_state: {{ trigger.to_state }}, from_state: {{ trigger.from_state
                      }}
                    level: error
                  service: system_log.write
        mode: parallel
        max: 20
        ```

---

### **Permissões**

Para que a **Skill Alexa** funcione corretamente, são necessárias as seguintes permissões para acessar recursos e funcionalidades. Você as configurará no **Amazon Developer Console**, dentro da sua skill:

1.  No menu lateral da skill, vá para **"Account Linking"**.
    * **Marque a opção "Auth Code Grant"**.
    * **Authorization URI:** `https://www.amazon.com/ap/oa`
    * **Access Token URI:** `https://api.amazon.com/auth/o2/token`
    * Anote o **Client ID** e o **Client Secret**.
    * **Scope:** Adicione `alexa::skill_messaging`.
    * Salve as configurações.

2.  Habilite o envio de eventos da Alexa:
    * Na seção de **"Permissões"** ou **"Build"** (a localização exata pode variar ligeiramente na interface), você deve habilitar sua skill para **enviar mensagens para o endpoint de Eventos de Entrada da Alexa**. Isso autoriza sua skill a responder de forma assíncrona às diretivas da Alexa e a enviar eventos de mudança de estado.
    * Procure por uma seção relacionada a **"Alexa Skill Messaging"** ou **"Send Alexa Events"**. Ao habilitar, você obterá o:
        * **Alexa Client Id:** `amzn1.application-oa2-**************` (Este será gerado para sua skill).
        * **Alexa Client Secret:** `**************************************************************************************` (Este também será gerado para sua skill; clique em **SHOW** para visualizá-lo).

---

### **Atenção: Para que as alterações nas permissões tenham efeito, é crucial desabilitar a skill, aguardar 30 segundos e, em seguida, habilitá-la novamente.**

---

### 💡 Considerações Importantes
-   **Cloudflare:** Esta arquitetura foi testada e é recomendada para uso com o **Cloudflare** atuando como um proxy reverso para o seu Home Assistant. Isso adiciona uma camada extra de segurança (WAF, proteção contra DDoS) e pode simplificar a exposição segura da sua instância.
-   **Custos (Nível Gratuito da AWS):** Os serviços da AWS utilizados (Lambda, DynamoDB) possuem um **Nível Gratuito (Free Tier)** generoso. Para um uso residencial típico, os custos de operação desta skill devem ser nulos ou muito próximos de zero, desde que os limites do Free Tier não sejam excedidos.
### 📄 Licença
Este projeto é distribuído sob a Licença MIT.
