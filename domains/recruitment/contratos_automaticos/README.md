# 📝 Contratos BMídia — Geração e Envio Automático de Contratos (n8n)

Sistema de geração automática de contratos operado via **Schedule Trigger**, construído em n8n. A automação monitora uma planilha no **Google Sheets** em busca de novos contratos pendentes, identifica o modelo correto entre os três tipos disponíveis, preenche automaticamente os dados do cliente no template via **Google Docs API**, converte o documento para PDF e o envia diretamente à plataforma de assinatura digital **Autentique** — sem nenhuma intervenção manual após o cadastro na planilha.

---

## 📁 Arquivo

| Arquivo | Descrição |
|---|---|
| [`Contratos_Bmidia_sanitized.json`](./Contratos_Bmidia_sanitized.json) | Workflow único — monitora a planilha, gera o contrato e envia para assinatura |

> ⚠️ **Todos os dados sensíveis foram removidos.** Substitua os placeholders `{{...}}` pelos valores reais antes de importar no n8n. Veja a seção [Configuração](#%EF%B8%8F-configuração) abaixo.

---

## 🏗️ Arquitetura

```
Schedule Trigger (a cada 5 min)
         │
         ▼
┌─────────────────────────────────────────┐
│   Leitura e Filtragem (Google Sheets)   │
│                                         │
│  Info Inicial (URL)1                    │  ← Centraliza URLs das abas + token
│         │                               │
│  Dados Doc1 (aba Documentos)            │  ← Lê todos os registros
│         │                               │
│  Filter Docs1                           │  ← Filtra: Id Doc preenchido
│         │                                     Nome Empresa preenchido
│         │                                     STATUS vazio (não processado)
│         │                                     Tipo de Doc preenchido
│  Limit1 (1 por vez)                     │  ← Processa um contrato por ciclo
│         │                               │
│  Dados Usuarios1 (aba Usuários)         │  ← Lê todos os signatários
│         │                               │
│  Filter Usuarios1                       │  ← Filtra: mesmo Id Doc
│         │                                     Nome preenchido
│         │                                     Status vazio
│         │                                     Id Usuario preenchido
│  Aggregate All1                         │  ← Agrupa todos os signatários
│         │                               │
│  Map Info Usuarios1 (Code)              │  ← Formata lista de signers
│         │                                     [{name, email, action: "SIGN"}]
│  Map Info Doc1                          │  ← Mapeia campos do contrato
└─────────┼───────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────┐
│   Switch por Tipo de Contrato           │
│                                         │
│  ├── "IMPLEMENTAÇÃO" → Edit Fields      │  ← label: "IMPLEMENTAÇÃO DE TRÁFEGO"
│  ├── "START"         → Edit Fields2     │  ← label: "CONTRATO DE TRÁFEGO PAGO"
│  └── "ESSENCIAL"     → Edit Fields3     │  ← label: "CONTRATO DE TRÁFEGO PAGO"
└─────────┬───────────────────────────────┘
          │
          ▼ (por branch, lógica idêntica)
┌─────────────────────────────────────────┐
│   Geração do Documento                  │
│                                         │
│  Copy file (Google Drive)               │  ← Copia template correto
│         │                               │
│  Update a document (Google Docs API)    │  ← Substituição de placeholders:
│         │                                     {{$nome.empresa}}
│         │                                     {{$cnpj}}
│         │                                     {{$endereco.empresa}}
│         │                                     {{$nome.representante}}
│         │                                     {{$cpf.representante}}
│  Download Doc (PDF) (Google Drive)      │  ← Converte e baixa como PDF
└─────────┼───────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────┐
│   Envio para Assinatura (Autentique)    │
│                                         │
│  Criar Doc (Autentique)                 │  ← POST /v2/graphql (GraphQL)
│  mutation CreateDocumentMutation        │     Envia PDF + signatários
│         │                               │     Configurações: lembrete diário,
│         │                               │     ordenação, recusa, vencimento,
│         │                               │     notificações, locale pt-BR
│         │                               │
│  UPDATE STATUS DOC1 (Sheets)            │  ← Marca STATUS = "CONCLUÍDO" na aba Docs
│         │                               │
│  Edit Fields1 → Split Out1              │  ← Expande lista de signatários
│         │                               │
│  UPDATE STATUS USUARIOS1 (Sheets)       │  ← Marca Status = "CONCLUIDO" para cada
└─────────────────────────────────────────┘     signatário processado
```

---

## 📋 Detalhamento do Fluxo

### Etapa 1 — Leitura e Filtragem da Planilha

O workflow dispara a cada **5 minutos** via Schedule Trigger. O nó `Info Inicial (URL)1` centraliza as URLs das duas abas da planilha e o token da Autentique — ponto único de configuração para quem for fazer manutenção.

O filtro `Filter Docs1` garante que apenas linhas válidas e ainda não processadas avancem, verificando quatro condições simultaneamente: `Id Doc` preenchido, `Nome Empresa` preenchido, `STATUS` **vazio** (anti-reprocessamento) e `Tipo de Doc` preenchido.

O nó `Limit1` processa **um contrato por execução** — decisão intencional para garantir que cada contrato seja gerado e enviado de forma íntegra antes do próximo ciclo.

### Etapa 2 — Coleta e Formatação dos Signatários

A aba de Usuários pode ter **múltiplos signatários** para o mesmo contrato. O filtro `Filter Usuarios1` cruza o `Id Doc` da aba de Documentos com o `ID Doc` da aba de Usuários para capturar todos os responsáveis pela assinatura do contrato em questão.

O nó `Map Info Usuarios1` (Code JavaScript) converte os dados brutos em uma lista de signatários no formato exigido pela API da Autentique:

```javascript
signers = [{ name, email, action: "SIGN" }]
```

### Etapa 3 — Switch por Tipo de Contrato

O campo `Tipo de Doc` da planilha define qual template será usado. O nó `Switch` roteia para a branch correta:

| Valor na Planilha | Template usado | Label do Documento |
|---|---|---|
| `IMPLEMENTAÇÃO` | Template Implementação de Tráfego | `IMPLEMENTAÇÃO DE TRÁFEGO -` |
| `START` | Template Tráfego Pago Plano Start | `CONTRATO DE TRÁFEGO PAGO -` |
| `ESSENCIAL` | Template Tráfego Pago Essencial | `CONTRATO DE TRÁFEGO PAGO -` |

### Etapa 4 — Preenchimento e Conversão do Documento

Para cada branch, a lógica é idêntica:

1. **Copy file** — copia o template correspondente para a pasta "Contratos ENVIADOS" no Drive, renomeando com o nome da empresa
2. **Update a document** — usa a Google Docs API para substituir os placeholders do template com os dados reais do cliente. Os placeholders nos templates são:

| Placeholder no Template | Campo da Planilha |
|---|---|
| `{{$nome.empresa}}` | Nome Empresa |
| `{{$cnpj}}` | CNPJ |
| `{{$endereco.empresa}}` | Endereço Empresa |
| `{{$nome.representante}}` | Nome (aba Usuários) |
| `{{$cpf.representante}}` | CPF |

3. **Download Doc (PDF)** — baixa o documento preenchido convertido para PDF via Google Drive

### Etapa 5 — Envio para a Autentique

O nó `Criar Doc (Autentique)` realiza uma chamada GraphQL (`mutation CreateDocumentMutation`) à API da Autentique enviando o PDF como upload binário (`multipart/form-data`) junto com as configurações do documento:

- Lista de signatários com nome, e-mail e ação (`SIGN`)
- Lembrete automático diário (`reminder: "DAILY"`)
- Ordenação de assinaturas configurável por linha da planilha
- Permissão de recusa do documento configurável
- Data de vencimento da assinatura (opcional, lida da planilha)
- Notificação ao concluir e ao receber cada assinatura
- Locale `pt-BR` / timezone `America/Sao_Paulo`

Após a confirmação da Autentique, o workflow atualiza a planilha: marca `STATUS = "CONCLUÍDO"` na aba de Documentos e `Status = "CONCLUIDO"` para cada signatário na aba de Usuários — impedindo reprocessamento nos ciclos seguintes.

---

## 📋 Estrutura Esperada da Planilha

O workflow lê de **duas abas** na mesma planilha Google Sheets.

**Aba 1 — Documentos** (referenciada por `URL Info Doc`):

| Coluna | Descrição |
|---|---|
| `Id Doc` | Identificador único do contrato |
| `Nome Empresa` | Razão social do cliente |
| `CNPJ` | CNPJ da empresa |
| `Endereço Empresa` | Endereço completo da empresa |
| `Tipo de Doc` | `IMPLEMENTAÇÃO`, `START` ou `ESSENCIAL` |
| `Mensagem Quando Enviado` | Mensagem personalizada no e-mail da Autentique |
| `Assinatura em Ordem` | `TRUE` / `FALSE` |
| `Permitir Recusa Documento` | `TRUE` / `FALSE` |
| `Ler Todo Documento` | `TRUE` / `FALSE` |
| `Vencimento Assinatura` | Data de expiração (opcional) |
| `Enviar Para Assinatura` | `TRUE` / `FALSE` |
| `Receber Notificação Quando Assinado` | `TRUE` / `FALSE` |
| `STATUS` | Deixar **vazio** para processar. Preenchido automaticamente com `CONCLUÍDO` |

**Aba 2 — Usuários** (referenciada por `URL Info Usuarios`):

| Coluna | Descrição |
|---|---|
| `Id Usuario` | Identificador único do signatário |
| `ID Doc` | Deve ser igual ao `Id Doc` do contrato correspondente |
| `Nome` | Nome completo do signatário |
| `Email` | E-mail do signatário (usado pela Autentique) |
| `CPF` | CPF do signatário |
| `Status` | Deixar **vazio** para processar. Preenchido automaticamente com `CONCLUIDO` |

---

## 🔌 Integrações

| Serviço | Uso | Autenticação |
|---|---|---|
| Google Sheets | Leitura dos dados de contratos e signatários + atualização de status | OAuth2 via credencial n8n |
| Google Docs | Substituição de placeholders no template do contrato | OAuth2 via credencial n8n |
| Google Drive | Cópia do template e download do PDF gerado | OAuth2 via credencial n8n |
| Autentique | Envio do contrato PDF para assinatura digital com configurações completas | Bearer Token (API Key) |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Credenciais OAuth2 Google (Sheets, Docs, Drive) configuradas no n8n
- Conta na [Autentique](https://autentique.com.br) com API Token gerado
- Planilha Google Sheets criada com as duas abas estruturadas conforme a seção acima
- Três templates de contrato criados no Google Docs, cada um com os placeholders `{{$nome.empresa}}`, `{{$cnpj}}`, `{{$endereco.empresa}}`, `{{$nome.representante}}` e `{{$cpf.representante}}` no corpo do texto
- Pasta "Contratos ENVIADOS" criada no Google Drive

### Instalação

1. **Importe o workflow** `Contratos_Bmidia_sanitized.json` no n8n.

2. **Configure as credenciais** no n8n:
   - `googleSheetsOAuth2Api` — OAuth2 Google Sheets
   - `googleDocsOAuth2Api` — OAuth2 Google Docs
   - `googleDriveOAuth2Api` — OAuth2 Google Drive

3. **Preencha os placeholders** no nó `Info Inicial (URL)1`:

| Placeholder | Descrição |
|---|---|
| `{{GOOGLE_SHEETS_CONTRATOS_URL}}` | URL da aba "Documentos" da planilha |
| `{{AUTENTIQUE_API_TOKEN}}` | Token de API da sua conta Autentique |

4. **Preencha os IDs do Google Drive** nos nós `Copy file`, `Copy file1` e `Copy file2`:

| Placeholder | Descrição |
|---|---|
| `{{GOOGLE_DRIVE_TEMPLATE_TRAFEGO_ESSENCIAL_ID}}` | ID do documento template — Tráfego Pago Essencial |
| `{{GOOGLE_DRIVE_TEMPLATE_TRAFEGO_START_ID}}` | ID do documento template — Tráfego Pago Start |
| `{{GOOGLE_DRIVE_TEMPLATE_IMPLEMENTACAO_ID}}` | ID do documento template — Implementação de Tráfego |
| `{{GOOGLE_DRIVE_FOLDER_CONTRATOS_ENVIADOS_ID}}` | ID da pasta "Contratos ENVIADOS" no Drive |

5. **Ajuste o intervalo do Schedule Trigger** se necessário (padrão: 5 minutos).

6. **Ative o workflow.**

---

## 🔒 Segurança

- Nunca versione o arquivo JSON com o `{{AUTENTIQUE_API_TOKEN}}` real — ele concede acesso total à criação e envio de documentos jurídicos da conta
- A planilha contém **dados pessoais e empresariais sensíveis** (CNPJ, CPF, endereço, e-mail) — restrinja o acesso ao mínimo de colaboradores necessários e use permissões OAuth2 com escopo restrito
- Os templates de contrato no Google Drive não devem ter permissão pública de leitura — mantenha o acesso restrito ao Service Account / OAuth2 utilizado pelo n8n
- A pasta "Contratos ENVIADOS" contém cópias dos contratos preenchidos com dados reais — restrinja o compartilhamento dessa pasta
- O mecanismo anti-reprocessamento (coluna `STATUS`) é crítico: nunca limpe o status de um contrato já enviado sem verificar se ele foi devidamente cancelado na Autentique primeiro
