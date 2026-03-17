# 📚 Notion Lesson Generator

Automação n8n que monitora uma pasta no Google Drive e, ao detectar uma exportação `.zip` do Notion, extrai automaticamente os documentos internos e usa **Google Gemini** para gerar materiais de aula personalizados para uma professora de inglês.

---

## 📁 Arquivos

| Arquivo | Descrição |
|---|---|
| [`Notion_ZIP_Extractor_-_Drive_v2.json`](./Notion_ZIP_Extractor_-_Drive_v2.json) | Workflow principal — extração, filtragem e geração de material de aula via IA |

---

## 🏗️ Arquitetura

```
Google Drive (pasta monitorada)
         │
         ▼ (novo arquivo .zip detectado)
┌──────────────────────────────────────────────────────┐
│  Google Drive Trigger                                │
│         │                                            │
│         ▼                                            │
│  Download file                                       │
│         │                                            │
│         ▼                                            │
│  Decompress Nível 1  ──→  É ZIP interno?             │
│                               │         │            │
│                            Sim         Não           │
│                               ▼         │            │
│                      Decompress Nível 2 │            │
│                               │         │            │
│                               └────┬────┘            │
│                                    ▼                 │
│                           Merge Arquivos             │
│                                    │                 │
│                                    ▼                 │
│                       Filtrar e Converter Binários   │
│                                    │                 │
│                               Split Out              │
│                                    │                 │
│                           Switch por Tipo            │
│                    ┌───────────────┼───────────────┐ │
│                    ▼               ▼               ▼ │
│               text (.md)        image            audio│
│          Extract from File   Captura Binário  Captura Binário
│                    │               │               │ │
│                    │         Analyze an image  Analyze audio
│                    └───────────────┴───────────────┘ │
│                                    │                 │
│                                  Merge               │
│                                    │                 │
│                                Aggregate             │
│                                    │                 │
│                          Message a model (Gemini)    │
│                                    │                 │
│                           Captura Output             │
│                                    │                 │
│                       Create file from text          │
│                         (Google Drive)               │
└──────────────────────────────────────────────────────┘
```

---

## 📋 Workflow

### `Notion ZIP Extractor - Drive v2`

**Fluxo completo:**

```
Google Drive Trigger
  → Download file
      Baixa o ZIP recém-depositado na pasta monitorada

  → Decompress Nível 1
      Descomprime o arquivo externo

  → Binário → Texto (Nível 1)
      Converte binários do primeiro nível

  → É ZIP interno?
      ├── Sim → Decompress Nível 2 (abre o ZIP aninhado gerado pelo Notion)
      └── Não → segue direto

  → Merge Arquivos
      Unifica os arquivos de ambos os caminhos

  → Filtrar e Converter Binários
      Classifica cada arquivo por tipo: text | image | audio

  → Split Out
      Itera item a item sobre os arquivos filtrados

  → Switch por Tipo
      ├── text  → Extract from File → lê o conteúdo real dos .md
      ├── image → Captura Binário image → Analyze an image (Gemini)
      └── audio → Captura Binário audio → Analyze audio (Gemini)

  → Merge
      Reúne os resultados dos três caminhos

  → Aggregate
      Consolida todo o conteúdo extraído em um único payload

  → Message a model (Gemini)
      Recebe o conteúdo agregado e gera o material de aula estruturado

  → Captura Output
      Extrai e organiza a resposta do Gemini

  → Create file from text (Google Drive)
      Salva o material gerado de volta no Drive
```

---

## 🎓 Material gerado

Para cada aluno identificado nos documentos, o Gemini produz um material completo com as seguintes seções:

| Seção | Conteúdo |
|---|---|
| 📘 **Cabeçalho** | Título criativo + perfil do aluno (nome, nível, tópico gramatical) |
| 🎧 **Transcription** | Gancho narrativo em estilo roteiro de áudio |
| 🔥 **Warm Up** | Perguntas profundas de conexão pessoal |
| 🎯 **Vocabulary** | Tabela de palavras avançadas extraídas dos documentos |
| 💬 **Quote & Reflection** | Citação + análise crítica |
| 🧠 **Story Chain** | Exercício colaborativo de produção livre |

---

## 🔌 Integrações

| Serviço | Uso | Autenticação |
|---|---|---|
| Google Drive | Monitoramento da pasta, download do ZIP e salvamento do material | OAuth2 via credencial Google Drive |
| Google Gemini | Análise de imagens, análise de áudio e geração do material de aula | API Key via credencial Google PaLM |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Credencial **Google Drive** configurada no n8n (OAuth2)
- Credencial **Google Gemini / PaLM API** configurada no n8n
- Pasta no Google Drive destinada a receber os arquivos `.zip` do Notion

### Instalação

1. **Importe o workflow** `Notion_ZIP_Extractor_-_Drive_v2.json` no n8n.

2. **Configure a credencial Google Drive:**
   - Vá em **Credentials → New → Google Drive OAuth2**
   - Autorize o acesso à conta Google

3. **Configure a credencial Google Gemini:**
   - Vá em **Credentials → New → Google PaLM API**
   - Insira a API Key do Google AI Studio

4. **Abra o nó `Google Drive Trigger`** e selecione a pasta monitorada.

5. **Ative o workflow.**

### Uso

1. Exporte uma página ou banco de dados do Notion como `.zip`
2. Deposite o arquivo na pasta monitorada do Google Drive
3. O workflow é disparado automaticamente
4. O material de aula é salvo no Drive ao final da execução

---

## 🏷️ Tags

`n8n` `google-drive` `notion` `gemini` `ai` `education` `english-teaching` `automation`
