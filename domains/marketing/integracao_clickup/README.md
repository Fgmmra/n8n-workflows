# 🚀 Onboarding Automático de Clientes — ClickUp + Drive + WhatsApp (n8n)

Sistema de onboarding automático de novos clientes operado via **Schedule Trigger**, construído em n8n. A automação monitora o **ClickUp** em busca de tasks sem gestor atribuído (clientes recém-chegados via formulário de onboarding), cria uma **pasta no Google Drive** e um **grupo no WhatsApp** personalizado de acordo com o tipo de contrato do cliente — tudo sem intervenção manual.

---

## 📁 Arquivo

| Arquivo | Descrição |
|---|---|
| [`clickup.json`](./clickup.json) | Workflow único — monitora o ClickUp, filtra novos clientes e provisiona Drive + grupo WhatsApp |

> ⚠️ **Dados sensíveis foram removidos.** Substitua os IDs de credenciais, IDs de listas/espaços do ClickUp e o ID da Data Table pelos valores reais antes de importar no n8n.

---

## 🏗️ Arquitetura

```
Schedule Trigger (a cada 3 min)
         │
         ▼
┌─────────────────────────────────────────┐
│   Busca e Filtragem                     │
│                                         │
│  Get many tasks (ClickUp)               │  ← Todas as tasks da lista de clientes
│         │                               │
│  Code: filtra sem Gestor                │  ← Detecta campo "Gestor " vazio
│         │                               │     Extrai task_id, nome, telefone
│  Get a task (por ID)                    │  ← Dados completos da task
│         │                               │
│  Edit Fields                            │  ← Normaliza nome, contrato, instância
│         │                               │
│  Data Table: consulta "clients"         │  ← Verifica se cliente já foi processado
│         │                               │
│  Code: filtro de duplicatas             │  ← Bloqueia reprocessamento
│         │                               │
│  Get a task1 (por ID)                   │  ← Segunda busca para extração do telefone
│         │                               │
│  Code: "só o telefon"                   │  ← Extrai e formata número BR (+55)
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│   Switch por Tipo de Contrato           │
│                                         │
│  ├── "implementação"  → branch IMPLEM   │
│  ├── "cliente parceiro" → branch CP     │
│  └── "clientes ativos" → branch CA     │
└────────┬────────────────────────────────┘
         │
         ▼ (por branch, lógica idêntica)
┌─────────────────────────────────────────┐
│   Provisionamento (Loop por cliente)    │
│                                         │
│  Create folder (Google Drive)           │  ← Pasta com nome + tipo de contrato
│         │                               │
│  Loop Over Items                        │
│         │                               │
│  Criar grupo (Evolution API)            │  ← Nome e descrição personalizados
│         │                               │
│  Buscar link de convite                 │  ← inviteUrl do grupo criado
│         │                               │
│  Enviar texto (WhatsApp)               │  ← Invite enviado no PV do cliente
│         │                               │
│  attClients (Data Table)               │  ← Registra cliente como processado
│         │                               │
│  Wait → próximo item do loop            │
└─────────────────────────────────────────┘
```

---

## 📋 Detalhamento do Fluxo

### Etapa 1 — Busca e Filtragem Inteligente

O workflow dispara a cada **3 minutos** via Schedule Trigger e busca **todas as tasks** da lista de clientes no ClickUp.

O primeiro nó `Code in JavaScript` resolve o problema central do projeto: o ClickUp não expõe um marcador explícito de "task nova". A solução foi identificar o `custom_field` do tipo `users` chamado `"Gestor "` — quando ele está vazio, a task representa um cliente recém-chegado que ainda não recebeu atendimento interno. Tasks com gestor já atribuído são descartadas.

```
custom_fields[] → detecta campo "Gestor " (type: users)
               → se vazio: task passa
               → se preenchido: task descartada
```

> **Por que esse campo?** O ClickUp dispara automações internas que podem alterar IDs de campos numerados sempre que o cliente reconfigura o dashboard. Usar o nome + tipo do campo como identificador torna o filtro robusto a essas mudanças.

### Etapa 2 — Anti-reprocessamento com Data Table

Após identificar as tasks sem gestor, o workflow consulta a **Data Table nativa do n8n** (`clickup clients`) que funciona como registro local de clientes já processados.

Um segundo `Code` compara os `task_id` retornados com os registros da tabela e deixa passar **apenas os genuinamente novos** — impedindo que a mesma pessoa receba duplicatas de pasta e grupo caso o ciclo de 3 minutos rode antes de um gestor ser atribuído.

### Etapa 3 — Switch por Tipo de Contrato

O nó `Switch` roteia cada cliente para a branch correta com base no `status` da task no ClickUp, que espelha o modelo de contrato:

| Status no ClickUp | Branch | Nome do Grupo |
|---|---|---|
| `implementação` | IMPLEM | `{Nome} - IMPLEMENTAÇÃO - TRAFEGO PAGO` |
| `cliente parceiro` | CP | `{Nome} - TRAFEGO PAGO` |
| `clientes ativos` | CA | `{Nome} - TRAFEGO PAGO` |

### Etapa 4 — Provisionamento em Loop

Para cada cliente (podendo haver múltiplos por ciclo), o `Loop Over Items` garante que cada execução crie **uma pasta e um grupo distintos**, sem que nenhum item seja pulado.

A lógica do grupo WhatsApp resolve uma limitação real da plataforma: se o cliente tiver o perfil configurado para **bloquear adição a grupos**, a tentativa de add direta falha silenciosamente. A automação contorna isso sempre buscando o link de convite (`inviteUrl`) e **enviando no PV** do cliente — funcionando independentemente da configuração de privacidade dele.

Ao final de cada iteração, os dados do cliente são inseridos na Data Table como registro de conclusão.

---

## 🔌 Integrações

| Serviço | Uso | Autenticação |
|---|---|---|
| ClickUp | Leitura de tasks de onboarding | OAuth2 via credencial n8n |
| Google Drive | Criação de pasta por cliente | OAuth2 via credencial n8n |
| Evolution API (WhatsApp) | Criação de grupo, busca de invite e envio de mensagem | API Key via credencial n8n |
| n8n Data Table | Registro local anti-duplicata | Nativo n8n (sem credencial externa) |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Instância **Evolution API** conectada a um número WhatsApp
- Conta ClickUp com OAuth2 configurado no n8n
- Credencial Google Drive (OAuth2) configurada no n8n
- **Data Table** criada no n8n com as colunas `task_id` (string), `phone_number` (string) e `Nome` (string)
- Lista de clientes no ClickUp com o campo `"Gestor "` do tipo `users` configurado

### Instalação

1. **Importe o workflow** `clickup.json` no n8n.

2. **Configure as credenciais** no n8n:
   - `clickUpOAuth2Api` — OAuth2 ClickUp
   - `googleDriveOAuth2Api` — OAuth2 Google Drive
   - `evolutionApi` — URL base e token da instância Evolution API

3. **Atualize os IDs do ClickUp** no nó `Get many tasks`:
   - `team` — ID do seu workspace
   - `space` — ID do seu Space
   - `folder` — ID da pasta
   - `list` — ID da lista de clientes

4. **Atualize o ID da Data Table** nos nós `clients`, `attClients`, `attClients1` e `attClients2` com o ID da tabela `clickup clients` criada na etapa de pré-requisitos.

5. **Configure a instância WhatsApp** no nó `Edit Fields`, campo `instancia`, com o nome da sua instância Evolution API.

6. **Personalize os nomes e descrições dos grupos** nos nós `Criar grupo IMPLEM`, `Criar grupo ATIVOS/PARCEIROS` e `Criar grupo ATIVOS/PARCEIROS1` conforme o padrão de nomenclatura do cliente.

7. **Ajuste o intervalo do Schedule Trigger** conforme a necessidade (padrão: 3 minutos).

8. **Ative o workflow.**

---

## 🧠 Decisões Técnicas

**Por que Schedule Trigger e não Webhook?**
O ClickUp dispara entre 15 e 16 requisições por evento — a execução pai mais todas as automações internas do dashboard do cliente. Filtrar qual delas é a "certa" via webhook é impraticável dado que nenhuma possui uma flag explícita de identificação. O Schedule Trigger elimina esse problema por completo: ao invés de reagir a eventos, o workflow simplesmente consulta o estado atual da lista em intervalos regulares.

**Por que dois `Get a task` (Get a task + Get a task1)?**
O primeiro get logo após o `Code` serve para obter os dados completos da task filtrada. O segundo (`Get a task1`) foi necessário pois o n8n não propagava corretamente as variáveis dos nós `Set` que vinham depois dos nós `Code` — extrair o telefone diretamente de um novo get resolveu a incompatibilidade de contexto.

**Por que Data Table e não Google Sheets?**
Para esse caso de uso específico — um registro local de controle de idempotência — a Data Table nativa do n8n cumpre o papel sem dependência de serviço externo, com latência menor e sem necessidade de credenciais adicionais.

---

## 🔒 Segurança

- Nunca versione o arquivo JSON com credenciais reais ou IDs de workspace expostos
- Restrinja as permissões OAuth2 do Google Drive ao mínimo necessário (criação de pastas apenas)
- A Data Table `clickup clients` contém telefones de clientes — mantenha o acesso restrito ao ambiente n8n
- A instância Evolution API deve ter autenticação por token ativa — nunca exponha a URL base publicamente
