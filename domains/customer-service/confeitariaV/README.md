# 🎂 Bot de Atendimento — Doceria (n8n)

Sistema de atendimento comercial automatizado via **WhatsApp**, construído em n8n. O bot recebe clientes, apresenta o cardápio (padrão e especial de Natal), envia o PDF do cardápio de Natal, coleta pedidos com itens e quantidades, calcula valores estimados via tool de calculadora, registra novos clientes no **NocoDB**, e ao finalizar o atendimento dispara uma **notificação automática para o grupo interno** da equipe com todos os dados do pedido. O projeto conta com buffer de mensagens via Redis, suporte a áudio e imagem, e dois agentes Gemini com memórias separadas.

---

## 📁 Arquivos

| Arquivo | Descrição |
|---|---|
| [`mvp-v_sanitized.json`](./mvp-v_sanitized.json) | Workflow principal — recebe mensagens, orquestra o agente e notifica a equipe |
| [`envia_imagem_sanitized.json`](./envia_imagem_sanitized.json) | Sub-workflow — envia imagem de referência de tamanhos ao cliente |
| [`envia_cardapio_sanitized.json`](./envia_cardapio_sanitized.json) | Sub-workflow — busca e envia o PDF do cardápio de Natal via Supabase |

> ⚠️ **Todos os dados sensíveis foram removidos.** Substitua os placeholders `{{...}}` pelos valores reais antes de importar no n8n. Veja a seção [Configuração](#%EF%B8%8F-configuração) abaixo.

---

## 🏗️ Arquitetura

```
WhatsApp (Evolution API Webhook)
         │
         ▼
┌─────────────────────────────────────────┐
│   Workflow Principal (mvp-v)            │
│                                         │
│  Webhook → variaveisWebhook             │
│         │                               │
│       #sair?                            │  ← Limpa Redis + confirma saída
│         │                               │
│      filtro (whitelist)                 │  ← Bloqueia números não autorizados
│         │                               │
│   clientes (NocoDB)                     │  ← Busca cliente pelo telefone
│         │                               │
│   Switch: cliente novo?                 │
│       ├── Novo  → registro (NocoDB)     │  ← Cria registro do cliente
│       ├── Existe → segue               │
│       └── Bloqueado → No Operation     │
│         │                               │
│  verificaTipoMensagem (switch)          │
│  ├── audio → get base64 → Convert      │
│  │           → Analyze audio (Gemini)  │  ← Transcrição de áudio
│  ├── image → get base  → Convert       │
│  │           → Analyze image (Gemini)  │  ← Análise de imagem
│  └── texto → messageText              │
│         │                               │
│     unificaMessage                      │
└─────────┼───────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────┐
│   Buffer (sub-workflow genérico)        │  ← Agrupa msgs via Redis (140s)
└─────────┬───────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────┐
│   AI Agent (Gemini + Redis Memory)      │
│                                         │
│   Tool: cardapio       (NocoDB)         │  ← Produtos padrão
│   Tool: cardapio_natal (NocoDB)         │  ← Produtos especiais de Natal
│   Tool: envia_cardapio (sub-workflow)   │  ← Envia PDF do cardápio de Natal
│   Tool: calc           (Calculator)    │  ← Soma dos itens do pedido
│         │                               │
│   Mensagem em texto (sub-workflow)      │  ← Envia resposta ao cliente
│         │                               │
│   encaminhaGrupo (switch)               │  ← Detecta frase de encerramento
│         │                               │
│   Relatório (Agente Gemini 2)          │  ← Extrai JSON do pedido
│         │                               │
│   Enviar para grupo (Evolution API)     │  ← Notifica equipe no WhatsApp
└─────────────────────────────────────────┘
```

---

## 📋 Workflows

### 1. `mvp-v` — Workflow Principal

Recebe todas as mensagens via webhook, gerencia o ciclo de vida do cliente, orquestra o agente de atendimento e aciona a notificação interna ao final de cada pedido concluído.

**Fluxo completo:**

```
Webhook (POST /atendimentos_vanessa)
  → variaveisWebhook
      (Telefone, instanceId, message, nome, messageType)
  → #sair?
      → Sim: Delete Redis → Enviar confirmação de histórico apagado
      → Não: filtro (whitelist de telefones autorizados)
            → Bloqueado: encerra silenciosamente
            → Autorizado: clientes (NocoDB — busca por telefone)
                  → Switch
                        ├── status = false (cliente novo)
                        │     → registro (NocoDB — cria cliente)
                        │     → verificaTipoMensagem
                        ├── status = true (cliente existente)
                        │     → verificaTipoMensagem
                        └── Não encontrado → No Operation
  → verificaTipoMensagem (switch)
        ├── audio → get base64 → Convert to File → Analyze audio (Gemini)
        ├── image → get base  → Convert to File1 → Analyze image (Gemini)
        └── texto → messageText
  → unificaMessage
  → Buffering (sub-workflow, delay 140s)
  → AI Agent (Gemini + Redis Memory)
      → Mensagem em texto (sub-workflow)
      → encaminhaGrupo
            → output contém frase de encerramento?
                  → Sim: Relatório (Agente Gemini 2 — extrai JSON do pedido)
                         → relatorio (formata var_assistant)
                         → Enviar para grupo (notificação interna)
                  → Não: encerra ciclo
```

**Diferenciais:**
- **Registro automático de clientes:** ao detectar um número novo, o bot cria o registro no NocoDB antes de iniciar o atendimento — sem intervenção manual
- **Dois agentes Gemini com memórias independentes:** o `AI Agent` conduz o atendimento com memória conversacional própria (Redis); o `Relatório` extrai os dados do pedido em JSON sem contaminar o histórico do atendimento
- **Detecção automática de encerramento:** o nó `encaminhaGrupo` monitora o output do agente procurando pela frase exata de encerramento para acionar a notificação — sem dependência de botão ou confirmação manual
- **Suporte a mídia:** áudios são transcritos e imagens analisadas pelo Gemini antes de chegarem ao agente

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{WEBHOOK_ID}}` | ID interno do webhook n8n |
| `{{CREDENTIAL_GOOGLE_GEMINI_ID}}` | ID da credencial Google Gemini |
| `{{CREDENTIAL_REDIS_ID}}` | ID da credencial Redis |
| `{{CREDENTIAL_EVOLUTION_API_ID}}` | ID da credencial Evolution API |
| `{{CREDENTIAL_NOCODB_ID}}` | ID da credencial NocoDB |
| `{{ALLOWED_PHONE_1}}` / `{{ALLOWED_PHONE_2}}` | Telefones autorizados na whitelist |
| `{{WHATSAPP_GROUP_JID}}` | JID do grupo WhatsApp de notificação interna |
| `{{NOCODB_PROJECT_ID}}` | ID do projeto no NocoDB |
| `{{NOCODB_TABLE_CLIENTES_ID}}` | ID da tabela de clientes no NocoDB |
| `{{NOCODB_TABLE_CARDAPIO_ID}}` | ID da tabela de cardápio padrão no NocoDB |
| `{{NOCODB_TABLE_CARDAPIO_NATAL_ID}}` | ID da tabela de cardápio de Natal no NocoDB |
| `{{WORKFLOW_BUFFER_ID}}` | ID do workflow de buffer genérico |
| `{{WORKFLOW_GENERIC_TEXT_ID}}` | ID do sub-workflow de envio de texto |
| `{{WORKFLOW_ENVIA_CARDAPIO_ID}}` | ID do sub-workflow envia_cardapio |

---

### 2. `envia_imagem`

Sub-workflow acionado como tool pelo agente principal para enviar ao cliente uma imagem de referência dos tamanhos de produtos disponíveis. Baixa o arquivo diretamente do Google Drive, converte para base64 e envia via Evolution API.

**Fluxo:**

```
Recebe (instanceId + phoneNumber)
  → Formalizar (normaliza telefoneCliente, nomeCliente, InstanceId)
  → Download file (Google Drive — tamanhos_bolo.jpg)
  → Extract from File (binaryToProperty)
  → Evolution API (send-image → cliente)
  → Resposta para o Agente
      ("A imagem com os tamanhos foi encaminhada, prossiga com o atendimento ao cliente")
```

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_EVOLUTION_API_ID}}` | ID da credencial Evolution API |
| `{{CREDENTIAL_GOOGLE_DRIVE_ID}}` | ID da credencial Google Drive OAuth2 |
| `{{GOOGLE_DRIVE_FILE_TAMANHOS_BOLO_ID}}` | ID do arquivo de imagem de tamanhos no Google Drive |

---

### 3. `envia_cardapio`

Sub-workflow acionado como tool pelo agente principal quando o cliente solicita o cardápio de Natal. Busca a URL do PDF no **Supabase** e envia o documento diretamente no chat do cliente via Evolution API. O agente possui regra de envio único por conversa — reenvio só sob insistência explícita do cliente.

**Fluxo:**

```
Recebe (instance + telefone)
  → Formalizar (normaliza telefoneCliente, InstanceId)
  → GetCardapio (Supabase — tabela envia-cardapio → retorna URL do PDF)
  → Evolution API1 (send-document → envia PDF ao cliente)
      caption: "Aqui está nosso cardápio de natal"
      fileName: "CardapioDeNatal.pdf"
  → Resposta para o Agente
      ("O cardápio de natal já foi enviado! Pergunte agora, somente o pedido que o cliente deseja fazer.")
```

**Placeholders:**

| Placeholder | Descrição |
|---|---|
| `{{CREDENTIAL_EVOLUTION_API_ID}}` | ID da credencial Evolution API |
| `{{CREDENTIAL_SUPABASE_ID}}` | ID da credencial Supabase |
| `{{SUPABASE_TABLE_ENVIA_CARDAPIO}}` | Nome da tabela no Supabase que armazena a URL do PDF |

---

## 🤖 Agente de Atendimento

O agente segue um fluxo de atendimento estruturado com **4 etapas obrigatórias e sequenciais**:

**1. Saudação e oferta do cardápio de Natal**
Na primeira mensagem, o agente obrigatoriamente informa sobre as encomendas de Natal abertas e pergunta se o cliente prefere o cardápio de Natal (PDF) ou o cardápio padrão (lista no chat).

**2. Montagem do pedido**
Dependendo da escolha:
- **Natal** → aciona `envia_cardapio` (uma única vez) e usa `cardapio_natal` apenas para dúvidas pontuais
- **Padrão** → usa `cardapio` e apresenta os itens em formato de lista no chat

O valor total é sempre calculado pela tool `calc` e informado como **"a partir de R$X"** — nunca como valor fechado.

**3. Coleta da data de preferência**
Etapa separada e obrigatória. Para pedidos de Natal, a data fixa é 23/12. Para pedidos padrões, valida que seja uma data futura e informa as regras de entrega (segunda a sábado, sem domingos).

**4. Coleta do endereço de entrega**
Somente após a data confirmada. Coleta rua, número, bairro, complemento e cidade. Informa opções de retirada no Ateliê (Penha) ou Uber Flash por conta do cliente.

**Encerramento:** ao finalizar todas as etapas, o agente encerra com a frase exata *"entraremos em contato o quanto antes para finalização"* — frase que o nó `encaminhaGrupo` monitora para disparar a notificação interna.

**Notificação interna** inclui: nome do cliente, telefone, data de preferência, itens pedidos, valor calculado e endereço de entrega.

---

## 🔌 Integrações

| Serviço | Uso | Autenticação |
|---|---|---|
| Evolution API (WhatsApp) | Recebimento e envio de mensagens, imagens e documentos | API Key via credencial n8n |
| Google Gemini | Agente de atendimento, transcrição de áudio, análise de imagem, extração de relatório | API Key (googlePalmApi) |
| Redis | Buffer de mensagens e memória conversacional dos dois agentes | Credencial Redis no n8n |
| NocoDB | Base de dados de clientes, cardápio padrão e cardápio de Natal | API Token via credencial n8n |
| Google Drive | Armazenamento e download da imagem de referência de tamanhos | OAuth2 via credencial n8n |
| Supabase | Armazenamento da URL do PDF do cardápio de Natal | API Key via credencial n8n |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Instância **Evolution API** conectada ao número WhatsApp da doceria
- API Key do Google Gemini (Google AI Studio)
- Instância Redis acessível pelo n8n
- Projeto **NocoDB** com três tabelas criadas: clientes, cardápio padrão e cardápio de Natal
- Conta **Supabase** com tabela contendo a URL pública do PDF do cardápio de Natal
- Arquivo `tamanhos_bolo.jpg` no Google Drive com imagem de referência dos tamanhos
- Grupo WhatsApp interno criado e JID identificado
- Sub-workflow genérico de buffer (`generic-buffer`) e de envio de texto (`generic-text`) importados e ativos

### Estrutura esperada do NocoDB

**Tabela clientes** (`{{NOCODB_TABLE_CLIENTES_ID}}`):

| Campo | Tipo | Descrição |
|---|---|---|
| `nome` | Texto | Nome do cliente |
| `telefone` | Texto | Número no formato `5521999999999` |
| `status` | Texto | `true` = cliente já registrado |

**Tabela cardápio** (`{{NOCODB_TABLE_CARDAPIO_ID}}`):

| Campo | Tipo | Descrição |
|---|---|---|
| `nome` | Texto | Nome do produto |
| `preco` | Número | Preço unitário (opcional — itens sem preço são apresentados sem valor) |

**Tabela cardápio_natal** (`{{NOCODB_TABLE_CARDAPIO_NATAL_ID}}`):

| Campo | Tipo | Descrição |
|---|---|---|
| `nome` | Texto | Nome do produto especial de Natal |
| `preco` | Número | Preço unitário |
| `descricao` | Texto | Descrição / ingredientes do item |

**Tabela Supabase** (`{{SUPABASE_TABLE_ENVIA_CARDAPIO}}`):

| Campo | Tipo | Descrição |
|---|---|---|
| `url` | Texto | URL pública do PDF do cardápio de Natal |

### Instalação

1. **Importe os workflows** no n8n nesta ordem:
   1. `envia_imagem_sanitized.json`
   2. `envia_cardapio_sanitized.json`
   3. `mvp-v_sanitized.json`

2. **Configure as credenciais** no n8n:
   - `googlePalmApi` — API Key do Google Gemini
   - `googleDriveOAuth2Api` — OAuth2 Google Drive
   - `evolutionApi` — URL base e token da instância Evolution API
   - `redis` — host, porta e senha do Redis
   - `nocoDbApiToken` — URL e token da instância NocoDB
   - `supabaseApi` — URL e chave da instância Supabase

3. **Substitua todos os placeholders** `{{...}}` pelos valores reais em cada workflow conforme as tabelas acima.

4. **Atualize os IDs dos sub-workflows** nos nós `Buffering`, `Mensagem em texto` e `envia_cardapio` com os IDs gerados após a importação.

5. **Configure o webhook na Evolution API:** aponte o webhook de mensagens recebidas para a URL do nó Webhook do workflow principal (path: `/atendimentos_vanessa`).

6. **Ative os 3 workflows.**

### Comando especial

Enviar `#sair` pelo WhatsApp limpa a memória Redis da conversa e encerra a sessão — útil para reiniciar o atendimento durante testes sem precisar criar um novo número.

---

## 🔒 Segurança

- Nunca versione os arquivos JSON com credenciais reais
- A **whitelist de telefones** no nó `filtro` é a primeira barreira de acesso — mantenha-a com apenas os números autorizados para testes internos
- A tabela de **clientes no NocoDB** contém dados pessoais (nome e telefone) — restrinja o acesso à API ao mínimo necessário
- O **JID do grupo interno** não deve ser exposto — as notificações de pedidos chegam diretamente à equipe com dados completos dos clientes
- A **URL do PDF do cardápio** no Supabase é enviada diretamente ao cliente — certifique-se de que o arquivo está em storage público com link não-indexável
- O agente possui regra embutida de **envio único do cardápio PDF** por conversa — não remova essa instrução do system prompt sem testar o comportamento resultante
