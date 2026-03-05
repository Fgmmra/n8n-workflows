# 🍋 Lemon Consultoria — Automação Financeira (n8n)

Projeto de automação financeira integrado com **ContaAzul**, **Google Drive/Sheets**, **SharePoint** e **Gmail**, construído em n8n. Busca dados financeiros diários e mensais, consolida em planilhas e distribui automaticamente por e-mail e SharePoint.

---

## 📁 Arquivos

| Arquivo | Descrição |
|---|---|
| [`Lemon_Consultoria_-_AccessToken_sanitized.json`](./Lemon_Consultoria_-_AccessToken_sanitized.json) | Gerenciamento de tokens OAuth2 do ContaAzul |
| [`Lemon_Consultoria_-_Busca_Dados_sanitized.json`](./Lemon_Consultoria_-_Busca_Dados_sanitized.json) | Sub-workflow de coleta de dados financeiros |
| [`Lemon_Consultoria_-_Envio_diario_Mensal_sanitized.json`](./Lemon_Consultoria_-_Envio_diario_Mensal_sanitized.json) | Workflow principal — processamento e distribuição |

> ⚠️ **Todos os dados sensíveis foram removidos.** Substitua os placeholders `{{...}}` pelos valores reais antes de importar no n8n. Veja a seção [Configuração](#%EF%B8%8F-configuração) abaixo.

---

## 🏗️ Arquitetura

```
┌─────────────────────────────┐
│   AccessToken               │  ← Roda a cada 1h + webhook OAuth2
│   Gerencia tokens ContaAzul │
└────────────┬────────────────┘
             │ refresh_token → DataTable
             ▼
┌─────────────────────────────┐
│   Busca Dados               │  ← Sub-workflow chamado pelo principal
│   Coleta dados da API       │
└────────────┬────────────────┘
             │ array agregado
             ▼
┌─────────────────────────────┐
│   Envio Diário & Mensal     │  ← Roda todo dia às 06:30
│   Processa e distribui      │
└─────────────────────────────┘
```

---

## 📋 Workflows

### 1. `Lemon Consultoria - AccessToken`

Gerencia o ciclo de vida dos tokens OAuth2 do ContaAzul. Possui dois fluxos independentes:

**Autorização inicial (Webhook)**
- Recebe o `code` OAuth2 via webhook
- Troca o `code` por `access_token` + `refresh_token`
- Salva ambos na DataTable do n8n

**Renovação automática (Schedule — a cada 1h)**
- Lê o `refresh_token` da DataTable
- Solicita um novo `access_token` à API
- Atualiza o token na DataTable

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CONTAAZUL_BASIC_AUTH_BASE64}}` | `client_id:client_secret` da app ContaAzul em Base64 |
| `{{WEBHOOK_PATH}}` | Path do webhook para receber o callback OAuth2 |
| `{{N8N_WEBHOOK_BASE_URL}}` | URL base da instância n8n (ex: `https://sua-instancia.n8n.io`) |
| `{{DATATABLE_ID}}` | ID da DataTable onde os tokens são armazenados |
| `{{PROJECT_ID}}` | ID do projeto n8n |

---

### 2. `Lemon Consultoria - Busca Dados`

Sub-workflow de coleta paginada de dados financeiros da API ContaAzul. Realiza 5 buscas paralelas com paginação automática (50 itens/página, loop até `temMaisPaginas = false`):

- **Contas a Pagar** — intervalo mensal
- **Contas a Receber** — intervalo mensal
- **Vendas** — intervalo diário
- **NFS-e Parte 1** — notas fiscais (1ª quinzena do mês anterior)
- **NFS-e Parte 2** — notas fiscais (2ª quinzena até hoje)

> **Lógica de datas:** se for dia 5 do mês, usa intervalo mensal completo; caso contrário, usa apenas o dia anterior (modo diário).

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{DATATABLE_ID}}` | ID da DataTable com os tokens ContaAzul |
| `{{PROJECT_ID}}` | ID do projeto n8n |

---

### 3. `Lemon Consultoria - Envio Diário & Mensal`

Workflow principal. Orquestra a coleta, processamento e distribuição dos dados. Disparado todo dia às **06:30 AM**.

**Fluxo:**

```
Schedule (06:30)
  → Chama sub-workflow Busca Dados
  → Copia planilha modelo no Google Drive
  → Compartilha a cópia (permissão pública)
  → Normaliza os dados via nó JavaScript
  → Preenche a planilha via Google Sheets
  → Obtém token Azure AD para o SharePoint
  → Baixa a planilha como .xlsx
  → Envia por e-mail via Gmail
  → Faz upload do .xlsx no SharePoint
  → Aguarda 10s
  → Deleta a cópia temporária do Google Drive
```

**Schema unificado gerado pelo nó JavaScript:**

`origem` · `id` · `descricao` · `status` · `total` · `data_vencimento` · `data_competencia` · `data_criacao` · `data_alteracao` · `pago` · `nao_pago` · `cliente_nome` · `cliente_email` · `numero_venda` · `numero_nfse` · `numero_rps` · `codigo_cnae` · `documento_cliente` · `cidade_emissao` · `tipo` · `id_contrato`

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{GOOGLE_FILE_ID}}` | ID do arquivo modelo no Google Drive |
| `{{GOOGLE_FILE_URL}}` | URL do arquivo modelo no Google Drive |
| `{{GOOGLE_SHEET_URL}}` | URL da aba da planilha modelo |
| `{{CREDENTIAL_ID}}` | ID da credencial OAuth2 (Google Drive, Sheets ou Gmail) |
| `{{RECIPIENT_EMAIL}}` | E-mail destinatário do relatório |
| `{{AZURE_TENANT_ID}}` | Tenant ID do Azure AD |
| `{{AZURE_CLIENT_ID}}` | Client ID da aplicação registrada no Azure |
| `{{AZURE_CLIENT_SECRET}}` | Client Secret da aplicação Azure ⚠️ |
| `{{SHAREPOINT_DRIVE_ID}}` | ID do drive do SharePoint destino |
| `{{SUBFLOW_WORKFLOW_ID}}` | ID do workflow Busca Dados no n8n |
| `{{ERROR_WORKFLOW_ID}}` | ID do workflow de tratamento de erros |

---

## 🔌 Integrações

| Serviço | Uso | Autenticação |
|---|---|---|
| ContaAzul API v2 | Fonte dos dados financeiros | OAuth2 — gerenciado pelo workflow AccessToken |
| Google Drive | Cópia e download da planilha modelo | OAuth2 via credencial n8n |
| Google Sheets | Escrita dos dados na planilha | OAuth2 via credencial n8n |
| Gmail | Envio do relatório por e-mail | OAuth2 via credencial n8n |
| Microsoft SharePoint | Armazenamento do arquivo `.xlsx` | Client Credentials (Azure AD) |
| n8n DataTable | Armazenamento dos tokens ContaAzul | Interno |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Aplicação OAuth2 registrada no ContaAzul
- Credenciais Google (Drive, Sheets, Gmail) configuradas no n8n
- Aplicação registrada no Azure AD com permissão `Files.ReadWrite` no SharePoint
- Planilha modelo criada no Google Drive

### Instalação

1. **Importe os workflows** no n8n nesta ordem:
   1. `Lemon_Consultoria_-_AccessToken_sanitized.json`
   2. `Lemon_Consultoria_-_Busca_Dados_sanitized.json`
   3. `Lemon_Consultoria_-_Envio_diario_Mensal_sanitized.json`

2. **Crie a DataTable** no n8n com as colunas `AccessToken` e `RefreshToken`. Anote o ID gerado.

3. **Substitua todos os placeholders** `{{...}}` pelos valores reais nos três workflows conforme as tabelas acima.

4. **Vincule o sub-workflow:** no nó `subflowCall` do workflow principal, atualize o `workflowId` com o ID gerado para o workflow Busca Dados.

5. **Autorize o ContaAzul:** acesse a URL de autorização OAuth2 do ContaAzul apontando o redirect para o webhook configurado. Isso populará a DataTable com os tokens iniciais.

6. **Ative os 3 workflows.**

---

## 🔒 Segurança

- Nunca versione os arquivos JSON com credenciais reais
- Prefira usar o sistema de **Credentials** do n8n em vez de valores fixos nos nós HTTP Request
- Renove o **Azure Client Secret** periodicamente e imediatamente em caso de exposição
- Restrinja as permissões do SharePoint ao mínimo necessário (escrita apenas na pasta destino)
- Considere usar **variáveis de ambiente** do n8n para os valores sensíveis recorrentes
