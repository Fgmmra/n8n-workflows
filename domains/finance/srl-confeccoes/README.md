# 📊 CFO Digital — Agente Financeiro IA (n8n)

Sistema de CFO virtual operado via **WhatsApp**, construído em n8n. O agente acessa dados financeiros reais em **Google Sheets**, executa análises de DRE, Fluxo de Caixa e centros de custo com **Google Gemini**, e entrega relatórios em texto ou **PDF** diretamente ao usuário. Conta com whitelist de acesso, buffer de mensagens com **Guardrails** anti-jailbreak e suporte a múltiplos tipos de mídia.

---

## 📁 Arquivos

| Arquivo | Descrição |
|---|---|
| [`CFO_Digital_-_Agente_conversacional_sanitized.json`](./CFO_Digital_-_Agente_conversacional_sanitized.json) | Workflow principal — recebe mensagens e orquestra o agente CFO |
| [`CFO_Digital_-_buffer_sanitized.json`](./CFO_Digital_-_buffer_sanitized.json) | Buffer de mensagens com Guardrails anti-jailbreak/NSFW |
| [`CFO_Digital_-_Agente_de_calculos_sanitized.json`](./CFO_Digital_-_Agente_de_calculos_sanitized.json) | Sub-agente de análise financeira com acesso ao Google Sheets |
| [`CFO_Digital_-_Envio_de_pdf_sanitized.json`](./CFO_Digital_-_Envio_de_pdf_sanitized.json) | Geração e envio de relatório em PDF via Google Docs + Drive |
| [`CFO_digital_-_envio_de_texto_sanitized.json`](./CFO_digital_-_envio_de_texto_sanitized.json) | Sub-workflow de envio de mensagens de texto via WhatsApp |
| [`CFO_Digital_-_Envio_de_audio_sanitized.json`](./CFO_Digital_-_Envio_de_audio_sanitized.json) | Sub-workflow de envio de mensagens de áudio via WhatsApp |

> ⚠️ **Todos os dados sensíveis foram removidos.** Substitua os placeholders `{{...}}` pelos valores reais antes de importar no n8n. Veja a seção [Configuração](#%EF%B8%8F-configuração) abaixo.

---

## 🏗️ Arquitetura

```
WhatsApp (UazAPI Webhook)
         │
         ▼
┌─────────────────────────────────────────┐
│   Agente Conversacional                 │  ← Workflow principal
│                                         │
│  Webhook → variaveis → #sair?           │
│                │                        │
│         verificaContato                 │  ← Whitelist de telefones
│                │                        │
│    verificaTipoMensagem (switch)         │
│    audio/texto/image/video/file         │
│                │                        │
│    Gemini (transcrição/análise mídia)   │
│                │                        │
│         unificaMessage                  │
│                │                        │
└────────────────┼────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   Buffer                                │  ← Agrupa msgs + Guardrails
│                                         │
│   Redis (push) → Wait 8s               │
│   → Verificar nova msg                  │
│         ├── Guardrails (Gemini)         │  ← Detecta jailbreak/NSFW
│         │     ├── OK → passa adiante   │
│         │     └── Bloqueado → recusa   │
│         └── Captura lista de msgs      │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   AI Agent (CFO Virtual)                │  ← Gemini + Redis Memory
│                                         │
│   Tool: Agente de Cálculos             │  ← Lê Google Sheets
│   Tool: Envio de PDF                   │  ← Gera PDF via Google Docs
└────────┬────────────────────────────────┘
         │
    ┌────┴──────────────────┐
    ▼                       ▼
Envio de texto          Envio de áudio
(UazAPI → WhatsApp)     (UazAPI → WhatsApp)
```

---

## 📋 Workflows

### 1. `CFO Digital - Agente Conversacional`

Workflow principal. Recebe todas as mensagens do WhatsApp via webhook e orquestra o fluxo completo.

**Fluxo:**

```
Webhook (POST)
  → Extrai variáveis (phoneNumber, messageType, messageId, userMessage)
  → Verifica comando #sair
      → Sim: limpa Redis + envia confirmação
      → Não: verificaContato (whitelist)
            → Fora da lista: encerra silenciosamente
            → Na lista: identificaTipoMensagem (switch)
                  ├── audio    → Download → Gemini (transcrição)
                  ├── texto    → mensagem direta
                  ├── image    → Download → Gemini (análise)
                  ├── video    → Download → Gemini (análise)
                  └── file     → Download → Gemini (análise)
  → Unifica mensagem
  → Buffer (140s de espera)
  → AI Agent CFO (Gemini + Redis Memory)
      ├── Tool: Agente de Cálculos
      └── Tool: Envio de PDF
  → Envio de texto ao usuário
```

**Diferenciais em relação a outros agentes:**
- **Whitelist de acesso:** apenas telefones autorizados interagem com o CFO
- **Suporte a mídia completo:** o agente consegue analisar documentos financeiros enviados como imagem, PDF ou vídeo

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{WEBHOOK_PATH}}` | Path do webhook (ex: `CFOdigitall`) |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n |
| `{{CREDENTIAL_ID}}` | ID da credencial Google Gemini / UazAPI / Redis |
| `{{ALLOWED_PHONE_1}}` … `{{ALLOWED_PHONE_5}}` | Telefones autorizados a usar o CFO Digital |
| `{{GOOGLE_DOC_TEMPLATE_ID}}` | ID do documento modelo no Google Drive para geração de PDF |
| `{{CFO_DIGITAL_BUFFER_WORKFLOW_ID}}` | ID do workflow CFO Digital - buffer |
| `{{CFO_DIGITAL_ENVIO_DE_TEXTO_WORKFLOW_ID}}` | ID do workflow CFO digital - envio de texto |
| `{{CFO_DIGITAL_AGENTE_DE_CALCULOS_WORKFLOW_ID}}` | ID do workflow CFO Digital - Agente de calculos |
| `{{CFO_DIGITAL_ENVIO_DE_PDF_WORKFLOW_ID}}` | ID do workflow CFO Digital - Envio de pdf |
| `{{ERROR_WORKFLOW_ID}}` | ID do workflow de tratamento de erros |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 2. `CFO Digital - Buffer`

Buffer de mensagens com camada de segurança via **Guardrails**. Diferente do buffer padrão, este valida o conteúdo da mensagem antes de passá-la ao agente.

**Fluxo:**

```
Recebe (message + phoneNumber + delayTimeMs)
  → Salva mensagem em lista Redis (key por telefone)
  → Aguarda 8 segundos
  → Busca última mensagem
  → Compara com a mensagem atual
      → Diferente (nova msg chegou): encerra
      → Igual: passa pelo Guardrails (Gemini)
            ├── Aprovado: agrega lista → deleta Redis → retorna
            └── Bloqueado (jailbreak/NSFW): envia mensagem de recusa
                → deleta Redis → encerra
```

**Guardrails configurados:**
- Detecção de **jailbreak** (threshold 0.7)
- Detecção de conteúdo **NSFW** (threshold 0.7)

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID da credencial Redis / Gemini |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n (nó Wait) |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 3. `CFO Digital - Agente de Cálculos`

Sub-agente especializado em análise financeira. É acionado como tool pelo agente principal sempre que a pergunta envolve dados financeiros.

**Fluxo:**

```
Recebe (userMessage + userNumber)
  → Envia aviso ao usuário ("Um minuto, estou fazendo a consulta...")
  → Gemini Flash com acesso tool ao Google Sheets (Big Data Financeiro)
  → Retorna análise estruturada ao agente principal
```

**Capacidades analíticas:**
- DRE por período e unidade de negócio
- Análise de variância (Budget vs Actual)
- Eficiência operacional e margem de contribuição
- Ciclo financeiro e liquidez (Cash Runway)
- Comparações MoM e YoY com rigor matemático
- Projeções 2025 identificadas como "Estimativa baseada em tendência"

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID da credencial UazAPI / Google Sheets / Gemini |
| `{{GOOGLE_SHEETS_URL}}` | URL completa da planilha Big Data Financeiro |
| `{{SHEET_ID}}` | ID numérico da aba da planilha |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 4. `CFO Digital - Envio de PDF`

Sub-workflow que transforma a análise financeira em um documento PDF formatado e o envia ao usuário via WhatsApp.

**Fluxo:**

```
Recebe (conteudo + docDriveId + formatacao + userNumber)
  → Copia documento modelo no Google Drive
  → Compartilha a cópia (permissão pública de escrita)
  → Gemini Flash: limpa e estrutura o texto financeiro (remove markdown, emojis, etc.)
  → Substitui placeholder {{texto}} no documento com o conteúdo limpo
  → Lê índices do documento via Google Docs API
  → Gemini Pro: gera JSON de formatação para Google Docs API (batchUpdate)
  → Aplica formatação (fontes, tamanhos, negrito, alinhamento)
  → Baixa o documento como .xlsx via Google Drive
  → Converte para base64
  → Envia o PDF via UazAPI ao usuário
```

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID das credenciais UazAPI / Google Drive / Google Docs / Gemini |
| `{{UAZAPI_BASE_URL}}` | URL base da instância UazAPI (ex: `https://sua-instancia.uazapi.com`) |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 5. `CFO digital - Envio de texto`

Sub-workflow de envio de respostas em texto ao usuário, dividindo mensagens longas em blocos.

**Fluxo:**
```
Recebe (phoneNumber + message)
  → Divide message por \n\n em array de blocos
  → Loop: envia bloco → aguarda 1.5s → próximo bloco
```

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID da credencial UazAPI |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n (nó Wait) |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 6. `CFO Digital - Envio de áudio`

Sub-workflow de envio de mensagens via WhatsApp com a mesma lógica de blocos do envio de texto, utilizando a credencial UazAPI dedicada ao projeto CFO.

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID da credencial UazAPI |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n (nó Wait) |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

## 🔌 Integrações

| Serviço | Uso | Autenticação |
|---|---|---|
| UazAPI (WhatsApp) | Recebimento e envio de mensagens e documentos | API Key via credencial n8n |
| Google Gemini 2.0 Flash | Transcrição de áudio, análise de mídia, limpeza de texto | API Key (googlePalmApi) |
| Google Gemini 3 Flash Preview | Agente de cálculos financeiros | API Key (googlePalmApi) |
| Google Gemini 3 Pro Preview | Geração de JSON de formatação para Google Docs API | API Key (googlePalmApi) |
| Google Sheets | Fonte dos dados financeiros (Big Data Financeiro) | OAuth2 via credencial n8n |
| Google Docs | Edição e formatação do documento PDF | OAuth2 via credencial n8n |
| Google Drive | Cópia, compartilhamento e download do documento | OAuth2 via credencial n8n |
| Redis | Buffer de mensagens e memória do agente | Credencial Redis no n8n |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Instância UazAPI configurada e conectada a um número WhatsApp
- API Key do Google Gemini (Google AI Studio)
- Credenciais OAuth2 Google (Sheets, Docs, Drive) configuradas no n8n
- Instância Redis acessível pelo n8n
- Planilha **Big Data Financeiro** no Google Sheets com as colunas esperadas pelo agente
- Documento modelo no Google Drive com o placeholder `{{texto}}` no corpo

### Instalação

1. **Importe os workflows** no n8n nesta ordem:
   1. `CFO_Digital_-_buffer_sanitized.json`
   2. `CFO_Digital_-_Agente_de_calculos_sanitized.json`
   3. `CFO_Digital_-_Envio_de_pdf_sanitized.json`
   4. `CFO_digital_-_envio_de_texto_sanitized.json`
   5. `CFO_Digital_-_Envio_de_audio_sanitized.json`
   6. `CFO_Digital_-_Agente_conversacional_sanitized.json`

2. **Configure as credenciais** no n8n:
   - `googlePalmApi` — API Key do Google Gemini
   - `googleSheetsOAuth2Api` — OAuth2 Google Sheets
   - `googleDocsOAuth2Api` — OAuth2 Google Docs
   - `googleDriveOAuth2Api` — OAuth2 Google Drive
   - `redis` — host, porta e senha do Redis
   - `uazApiApi` — URL base e token da instância UazAPI

3. **Substitua todos os placeholders** `{{...}}` pelos valores reais em cada workflow conforme as tabelas acima.

4. **Configure a whitelist** no nó `verificaContato` do Agente Conversacional com os telefones autorizados no formato `5511999999999`.

5. **Vincule os sub-workflows** no Agente Conversacional: atualize os `workflowId` dos nós `Execute Workflow` e `toolWorkflow` com os IDs gerados na importação.

6. **Configure o webhook na UazAPI:** aponte o webhook de mensagens recebidas para a URL do nó Webhook do Agente Conversacional.

7. **Ative os 6 workflows.**

### Comando especial

Enviar `#sair` pelo WhatsApp limpa a memória Redis do usuário e encerra a sessão — útil durante testes.

---

## 🔒 Segurança

- Nunca versione os arquivos JSON com credenciais reais
- A **whitelist de telefones** no nó `verificaContato` é a primeira barreira de acesso — mantenha-a restrita apenas aos usuários autorizados
- A planilha **Big Data Financeiro** contém dados financeiros sensíveis da empresa — restrinja as permissões OAuth2 ao mínimo necessário (somente leitura)
- O **Google Drive Doc ID** do template é referenciado em múltiplos workflows — não o exponha publicamente
- A URL base da UazAPI (`{{UAZAPI_BASE_URL}}`) identifica sua instância — mantenha-a privada
- O buffer possui **Guardrails** com Gemini para bloquear tentativas de jailbreak e conteúdo NSFW antes de atingir o agente
- O prompt do agente possui proteção anti-prompt-injection embutida nos blocos `<GeneratedPrompt>` — não os remova
