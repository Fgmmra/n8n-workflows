# 🍵 Aroma Café — Agente de Gestão de Estoque via WhatsApp (n8n)

Sistema de gestão de estoque operado via **WhatsApp**, construído em n8n. O agente acessa dados reais em **Google Sheets**, permite consultar e atualizar produtos, preços e quantidades por linguagem natural usando **Google Gemini**, e entrega respostas em texto diretamente ao gestor. Conta com whitelist de acesso, buffer de mensagens e suporte a múltiplos tipos de mídia.

---

## 📁 Arquivos

| Arquivo | Descrição |
|---|---|
| [`Chat-Gestao-Aroma-cafe_sanitized.json`](./Chat-Gestao-Aroma-cafe_sanitized.json) | Workflow principal — recebe mensagens e orquestra o agente de estoque |
| [`Aroma-Cafe-buffer_sanitized.json`](./Aroma-Cafe-buffer_sanitized.json) | Buffer de mensagens — agrupa envios consecutivos antes de processar |
| [`Aroma-Cafe-text_sanitized.json`](./Aroma-Cafe-text_sanitized.json) | Sub-workflow de envio de respostas em texto via WhatsApp |
| [`Aroma-Cafe-Audio_sanitized.json`](./Aroma-Cafe-Audio_sanitized.json) | Sub-workflow de envio alternativo via instância WhatsApp secundária |

> ⚠️ **Todos os dados sensíveis foram removidos.** Substitua os placeholders `{{...}}` pelos valores reais antes de importar no n8n. Veja a seção [Configuração](#%EF%B8%8F-configuração) abaixo.

---

## 🏗️ Arquitetura

```
WhatsApp (UazAPI Webhook)
         │
         ▼
┌─────────────────────────────────────────┐
│   Chat-Gestao-Aroma-café                │  ← Workflow principal
│                                         │
│  Webhook → variaveisWebhook             │
│                │                        │
│         verificaContato (If)            │  ← Whitelist de telefones
│                │                        │
│         verificaGrupo (If)              │  ← Ignora mensagens de grupos
│                │                        │
│    verificaTipoMensagem (switch)         │
│    audio/texto/image/video/document     │
│                │                        │
│    Gemini (transcrição/análise mídia)   │
│                │                        │
│         unificaMessage                  │
└────────────────┼────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   Aroma-Café-buffer                     │  ← Agrupa mensagens consecutivas
│                                         │
│   Redis (push) → Wait 12s              │
│   → Verificar nova msg                  │
│         ├── Nova msg chegou: encerra   │
│         └── Nenhuma nova: agrega lista │
│               → Deleta Redis → retorna │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   AI Agent (Gemini + Redis Memory)      │  ← Janela de 10 mensagens
│                                         │
│   Tool: Consultar estoque              │  ← Lê Google Sheets
│   Tool: Alterar estoque                │  ← Escreve Google Sheets
└────────────────┬────────────────────────┘
                 │
                 ▼
        Envio de texto
   (Aroma-Café-text / Audio)
```

---

## 📋 Workflows

### 1. `Chat-Gestao-Aroma-café`

Workflow principal. Recebe todas as mensagens do WhatsApp via webhook e orquestra o fluxo completo.

**Fluxo:**

```
Webhook (POST)
  → Extrai variáveis (Telefone, Nome, tipo de mensagem, conteúdo)
  → verificaContato (whitelist — até 3 números autorizados)
        → Fora da lista: encerra silenciosamente
        → Na lista: verificaGrupo
              → Mensagem de grupo: encerra
              → Mensagem direta: verificaTipoMensagem (switch)
                    ├── audio    → Download → Gemini (transcrição)
                    ├── texto    → mensagem direta
                    ├── image    → Download → Gemini (análise)
                    ├── video    → Download → Gemini (análise)
                    └── document → Download → Gemini (análise)
  → Unifica mensagem
  → Chama Aroma-Café-buffer
  → AI Agent (Gemini + Redis Memory)
        ├── Tool: Consultar estoque (Google Sheets)
        └── Tool: Alterar estoque (Google Sheets)
  → Chama Aroma-Café-text
```

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{WEBHOOK_PATH}}` | Path do webhook no n8n |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n |
| `{{CREDENTIAL_ID}}` | ID das credenciais Google Gemini / UazAPI / Redis / Google Sheets |
| `{{ALLOWED_PHONE_1}}` … `{{ALLOWED_PHONE_3}}` | Telefones autorizados a usar o sistema |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 2. `Aroma-Café-buffer`

Buffer de mensagens que evita processar envios consecutivos em separado. Agrupa tudo dentro de uma janela de tempo e entrega o conjunto completo ao agente.

**Fluxo:**

```
Recebe (message + phoneNumber + delayTimeMs)
  → Monta chave Redis: services:message-buffering:{phoneNumber}
  → Empurra mensagem na lista Redis
  → Aguarda 12 segundos
  → Lê lista acumulada do Redis
  → Compara última mensagem com a mensagem original
        → Diferentes (nova msg chegou): encerra sem processar
        → Iguais: agrega lista em ordem cronológica
              → Deleta chave Redis → retorna conteúdo
```

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID da credencial Redis |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n (nó Wait) |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 3. `Aroma-Café-text`

Sub-workflow de envio de respostas em texto. Divide mensagens longas em blocos para uma experiência mais natural na conversa.

**Fluxo:**

```
Recebe (phoneNumber + message)
  → Divide message por \n\n em array de blocos
  → Remove marcações [NIN] e colchetes residuais
  → Loop: envia bloco → aguarda 1.5s → próximo bloco
  → Retry automático: 5 tentativas, intervalo de 5s
```

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID da credencial UazAPI (`leonardo`) |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n (nó Wait) |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

### 4. `Aroma-Café-Audio`

Sub-workflow de envio alternativo, estruturalmente idêntico ao `Aroma-Café-text`, porém utilizando uma instância UazAPI diferente. Pode ser usado como fallback ou canal secundário.

**Diferenças em relação ao `Aroma-Café-text`:**
- Usa a credencial `UazAPI Felipe` (em vez de `leonardo`)
- Não possui retry automático no nó de envio

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_ID}}` | ID da credencial UazAPI (`UazAPI Felipe`) |
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n (nó Wait) |
| `{{INSTANCE_ID}}` / `{{WORKFLOW_ID}}` | IDs internos do n8n |

---

## 🤖 Comportamento do Agente de IA

O agente responde **apenas em português**, sem emojis e sem formatações especiais. Suas regras principais são:

- **Consulta:** chama automaticamente a ferramenta de estoque ao receber perguntas sobre produtos, preços, marcas ou quantidades.
- **Atualização:** executa imediatamente quando produto, campo e novo valor estão claros — sem pedir confirmação desnecessária.
- **Esclarecimento:** solicita mais informações apenas quando o produto é ambíguo, inexistente ou o valor não foi informado.
- **Confirmação:** após alterar o estoque, confirma de forma objetiva o que foi feito (ex: *"O preço do Feijão 1kg foi atualizado para R$ 10,20."*).

A planilha de estoque possui as colunas: `Produto`, `Marca`, `Preço`, `Quantidade`.

---

## 🔌 Integrações

| Serviço | Uso | Autenticação |
|---|---|---|
| UazAPI (WhatsApp) | Recebimento e envio de mensagens | API Key via credencial n8n |
| Google Gemini | Transcrição de áudio, análise de mídia, agente conversacional | API Key (googlePalmApi) |
| Google Sheets | Fonte e destino dos dados de estoque | OAuth2 via credencial n8n |
| Redis | Buffer de mensagens e memória de conversa (janela de 10 msgs) | Credencial Redis no n8n |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Instância UazAPI configurada e conectada a um número WhatsApp
- API Key do Google Gemini (Google AI Studio)
- Credenciais OAuth2 Google (Sheets) configuradas no n8n
- Instância Redis acessível pelo n8n
- Planilha no Google Sheets com as colunas: `Produto`, `Marca`, `Preço`, `Quantidade`

### Instalação

1. **Importe os workflows** no n8n nesta ordem:
   1. `Aroma-Cafe-buffer_sanitized.json`
   2. `Aroma-Cafe-text_sanitized.json`
   3. `Aroma-Cafe-Audio_sanitized.json`
   4. `Chat-Gestao-Aroma-cafe_sanitized.json`

2. **Configure as credenciais** no n8n:
   - `googlePalmApi` — API Key do Google Gemini
   - `googleSheetsOAuth2Api` — OAuth2 Google Sheets
   - `redis` — host, porta e senha do Redis
   - `uazApiApi` — URL base e token da instância UazAPI

3. **Substitua todos os placeholders** `{{...}}` pelos valores reais em cada workflow conforme as tabelas acima.

4. **Configure a whitelist** no nó `If` do workflow principal com os telefones autorizados no formato `5511999999999`.

5. **Vincule os sub-workflows** no workflow principal: atualize os `workflowId` dos nós `Execute Workflow` com os IDs gerados na importação.

6. **Configure o webhook na UazAPI:** aponte o webhook de mensagens recebidas para a URL do nó Webhook do workflow principal.

7. **Ative os 4 workflows.**

---

## 🔒 Segurança

- Nunca versione os arquivos JSON com credenciais reais
- A **whitelist de telefones** no nó `If` é a primeira barreira de acesso — mantenha-a restrita apenas aos gestores autorizados
- Mensagens de **grupos do WhatsApp** são ignoradas automaticamente pelo nó `verificaGrupo`
- A planilha de estoque contém dados sensíveis do negócio — restrinja as permissões OAuth2 ao mínimo necessário
- O buffer descarta silenciosamente execuções duplicadas quando uma nova mensagem do mesmo usuário chega durante a janela de espera, evitando processamento redundante
