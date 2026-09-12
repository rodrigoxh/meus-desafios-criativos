# Prompt de Implementação: Automação n8n para Captura e Qualificação de Leads

Você é um Arquiteto de Automação e Especialista em n8n. Sua tarefa é implementar um workflow robusto, de alta performance e à prova de falhas para captura, validação, persistência e notificação de leads recebidos via Google Forms.

---

## 1. Visão Geral do Projeto

- **Objetivo:** Capturar respostas de um formulário (Google Forms), validar a existência e sintaxe do e-mail, salvar os dados qualificados em uma planilha operacional de vendas (Google Sheets) e disparar notificações imediatas via Gmail (confirmação ao lead e alerta interno à equipe comercial).
- **Público-alvo:** Equipe comercial / time de vendas.
- **Stack / Ferramentas:** 
  - Google Forms (Captação de leads)
  - n8n (Orquestrador / Engine de automação)
  - Google Sheets (Base de dados / CRM de entrada)
  - Gmail (Canal transacional de e-mails)

---

## 2. Regras de Negócio e Requisitos

1. **Validação Estrita de E-mail:**
   - Registros sem endereço de e-mail ou com formato sintático inválido devem ser sumariamente descartados (ou desviados da esteira principal) antes de qualquer persistência ou disparo.
   - Utilizar validação por Regex padrão RFC 5322 simplificado: `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`.

2. **Higienização e Normalização de Dados:**
   - Fazer o trim de espaços em branco acidentais em todos os campos textuais (`nome`, `email`, `telefone`).
   - Forçar caixa baixa no e-mail para evitar duplicações e inconsistências.
   - Limpar formatações de telefone (remover parênteses, traços e espaços) mantendo apenas dígitos com DDI/DDD.
   - Gerar timestamp padronizado no fuso horário local (`America/Sao_Paulo`) no formato `dd/MM/yyyy HH:mm:ss`.

3. **Persistência Estruturada:**
   - Registrar na planilha `Controle Comercial` nas colunas: `Data`, `Nome`, `Email`, `Telefone` e `Status` (inicializado como `"Novo Lead"`).

4. **Comunicação Dupla via Gmail:**
   - **Lead:** E-mail transacional HTML de confirmação de recebimento com prazo de retorno.
   - **Time Comercial:** E-mail de alerta interno contendo todos os dados coletados e um link direto para abertura de conversa no WhatsApp (`https://wa.me/55...`).

---

## 3. Arquitetura do Workflow no n8n

```text
[1. Trigger: Google Sheets Trigger (ou Webhook)]
                    │
                    ▼
[2. Edit Fields / Set: Sanitização & Normalização]
                    │
                    ▼
[3. If Node: Validação Regex de E-mail]
         ├── [False] ──► [Encerrar execução / Log]
         └── [True]
                    │
                    ▼
[4. Google Sheets: Append Row (Planilha Comercial)]
                    │
                    ▼
[5. Gmail: Confirmação Transacional para o Lead]
                    │
                    ▼
[6. Gmail: Alerta Imediato para o Vendedor / Comercial]
```

---

## 4. Detalhamento dos Nós e Configurações Técnicas

### Nó 1: Gatilho (`Google Sheets Trigger`)
- **Tipo:** `n8n-nodes-base.googleSheetsTrigger` (versão 1)
- **Evento:** `Row Added`
- **Frequência de Checagem:** `Every Minute` (ou webhook instantâneo via Apps Script)
- **Planilha:** Aba vinculada ao Google Forms (ex: `Respostas ao formulário 1`).

### Nó 2: Sanitização de Dados (`Edit Fields / Set`)
- **Tipo:** `n8n-nodes-base.set` (versão 3.4)
- **Atribuições:**
  - `email`: `={{ $json['Endereço de e-mail'] ? $json['Endereço de e-mail'].trim().toLowerCase() : '' }}`
  - `nome`: `={{ $json['Nome'] ? $json['Nome'].trim() : 'Cliente' }}`
  - `telefone`: `={{ $json['Telefone'] ? $json['Telefone'].toString().replace(/\D/g, '') : '' }}`
  - `data_registro`: `={{ $now.setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy HH:mm:ss') }}`

### Nó 3: Validação Condicional (`If`)
- **Tipo:** `n8n-nodes-base.if` (versão 2)
- **Condição:** String Regex Match
- **Left Value:** `={{ $json.email }}`
- **Right Value:** `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
- **Saída True:** Segue para o Nó 4.
- **Saída False:** Fim do fluxo.

### Nó 4: Gravação na Planilha (`Google Sheets`)
- **Tipo:** `n8n-nodes-base.googleSheets` (versão 4.5)
- **Operação:** `Append Row`
- **Mapeamento de Colunas:**
  - `Data` -> `={{ $json.data_registro }}`
  - `Nome` -> `={{ $json.nome }}`
  - `Email` -> `={{ $json.email }}`
  - `Telefone` -> `={{ $json.telefone }}`
  - `Status` -> `"Novo Lead"`

### Nó 5: Disparo para o Lead (`Gmail - Confirmação`)
- **Tipo:** `n8n-nodes-base.gmail` (versão 2.1)
- **Destinatário:** `={{ $('Sanitizar Dados').item.json.email }}`
- **Assunto:** `Recebemos sua mensagem, {{ $('Sanitizar Dados').item.json.nome }}!`
- **Formato:** HTML
- **Template:**
  ```html
  <p>Olá, <strong>{{ $('Sanitizar Dados').item.json.nome }}</strong>!</p>
  <p>Agradecemos seu contato. Nossos consultores comerciais já receberam seus dados e entrarão em contato em breve.</p>
  <br>
  <p>Atenciosamente,<br><strong>Equipe Comercial</strong></p>
  ```

### Nó 6: Alerta Interno (`Gmail - Vendas`)
- **Tipo:** `n8n-nodes-base.gmail` (versão 2.1)
- **Destinatário:** `comercial@suaempresa.com.br`
- **Assunto:** `🚨 [Novo Lead Comercial] {{ $('Sanitizar Dados').item.json.nome }}`
- **Formato:** HTML
- **Template:**
  ```html
  <h3>Novo Lead Cadastrado no Formulário</h3>
  <ul>
    <li><strong>Nome:</strong> {{ $('Sanitizar Dados').item.json.nome }}</li>
    <li><strong>E-mail:</strong> {{ $('Sanitizar Dados').item.json.email }}</li>
    <li><strong>Telefone:</strong> {{ $('Sanitizar Dados').item.json.telefone }}</li>
    <li><strong>Data/Hora:</strong> {{ $('Sanitizar Dados').item.json.data_registro }}</li>
  </ul>
  <p><a href="https://wa.me/55{{ $('Sanitizar Dados').item.json.telefone }}" target="_blank">Clique aqui para chamar no WhatsApp</a></p>
  ```

---

## 5. Código JSON para Importação Direta no n8n

```json
{
  "name": "Captura e Qualificacao de Leads - Forms to Sheets & Gmail",
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyMinute"
            }
          ]
        },
        "documentId": {
          "__rl": true,
          "value": "SEU_SPREADSHEET_ID_AQUI",
          "mode": "id"
        },
        "sheetName": {
          "__rl": true,
          "value": "Respostas ao formulário 1",
          "mode": "name"
        },
        "event": "rowAdded"
      },
      "id": "1",
      "name": "Google Sheets Trigger",
      "type": "n8n-nodes-base.googleSheetsTrigger",
      "typeVersion": 1,
      "position": [240, 300]
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "email_norm",
              "name": "email",
              "value": "={{ $json['Endereço de e-mail'] ? $json['Endereço de e-mail'].trim().toLowerCase() : '' }}",
              "type": "string"
            },
            {
              "id": "nome_norm",
              "name": "nome",
              "value": "={{ $json['Nome'] ? $json['Nome'].trim() : 'Cliente' }}",
              "type": "string"
            },
            {
              "id": "tel_norm",
              "name": "telefone",
              "value": "={{ $json['Telefone'] ? $json['Telefone'].toString().replace(/\\D/g, '') : '' }}",
              "type": "string"
            },
            {
              "id": "data_norm",
              "name": "data_registro",
              "value": "={{ $now.setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy HH:mm:ss') }}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "id": "2",
      "name": "Sanitizar Dados",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [460, 300]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "valida_email",
              "leftValue": "={{ $json.email }}",
              "rightValue": "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
              "operator": {
                "type": "string",
                "operation": "regex"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "3",
      "name": "Possui E-mail Válido?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [680, 300]
    },
    {
      "parameters": {
        "operation": "append",
        "documentId": {
          "__rl": true,
          "value": "SEU_SPREADSHEET_ID_AQUI",
          "mode": "id"
        },
        "sheetName": {
          "__rl": true,
          "value": "Controle Comercial",
          "mode": "name"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Data": "={{ $json.data_registro }}",
            "Nome": "={{ $json.nome }}",
            "Email": "={{ $json.email }}",
            "Telefone": "={{ $json.telefone }}",
            "Status": "Novo Lead"
          },
          "matchingColumns": [],
          "schema": []
        },
        "options": {}
      },
      "id": "4",
      "name": "Registrar no Sheets",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.5,
      "position": [900, 280]
    },
    {
      "parameters": {
        "sendTo": "={{ $('Sanitizar Dados').item.json.email }}",
        "subject": "=Recebemos sua mensagem, {{ $('Sanitizar Dados').item.json.nome }}!",
        "emailType": "html",
        "message": "=<p>Olá, <strong>{{ $('Sanitizar Dados').item.json.nome }}</strong>!</p><p>Agradecemos seu contato. Nossos consultores comerciais já receberam seus dados e entrarão em contato em breve.</p><br><p>Atenciosamente,<br><strong>Equipe Comercial</strong></p>",
        "options": {}
      },
      "id": "5",
      "name": "Gmail - Confirmação Lead",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [1120, 280]
    },
    {
      "parameters": {
        "sendTo": "comercial@suaempresa.com.br",
        "subject": "=🚨 [Novo Lead Comercial] {{ $('Sanitizar Dados').item.json.nome }}",
        "emailType": "html",
        "message": "=<h3>Novo Lead Cadastrado no Formulário</h3><ul><li><strong>Nome:</strong> {{ $('Sanitizar Dados').item.json.nome }}</li><li><strong>E-mail:</strong> {{ $('Sanitizar Dados').item.json.email }}</li><li><strong>Telefone:</strong> {{ $('Sanitizar Dados').item.json.telefone }}</li><li><strong>Data/Hora:</strong> {{ $('Sanitizar Dados').item.json.data_registro }}</li></ul><p><a href=\"https://wa.me/55{{ $('Sanitizar Dados').item.json.telefone }}\" target=\"_blank\">Clique aqui para chamar no WhatsApp</a></p>",
        "options": {}
      },
      "id": "6",
      "name": "Gmail - Alerta Vendas",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [1340, 280]
    }
  ],
  "connections": {
    "Google Sheets Trigger": {
      "main": [
        [
          {
            "node": "Sanitizar Dados",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Sanitizar Dados": {
      "main": [
        [
          {
            "node": "Possui E-mail Válido?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Possui E-mail Válido?": {
      "main": [
        [
          {
            "node": "Registrar no Sheets",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Registrar no Sheets": {
      "main": [
        [
          {
            "node": "Gmail - Confirmação Lead",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Gmail - Confirmação Lead": {
      "main": [
        [
          {
            "node": "Gmail - Alerta Vendas",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```
