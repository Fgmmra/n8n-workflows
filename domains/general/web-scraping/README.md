# 🚢 Scraping de Dados de Cruzeiros (Kroóze)

Workflow de coleta e normalização de dados de cruzeiros a partir do sinal paginado da plataforma **Kroóze**. Percorre todas as páginas disponíveis da fonte, processa os registros recebidos, remove duplicatas e entrega os dados estruturados prontos para exportação.

---

## 📁 Arquivos

| Arquivo | Descrição |
|---|---|
| [`Scraping_de_dados.json`](./Scraping_de_dados.json) | Workflow único — coleta, normaliza, agrega e deduplica cruzeiros via sinal paginado da API Kroóze |

---

## 🏗️ Arquitetura

```
Manual Trigger
      │
      ▼
Gerar Lista de Páginas     ← Gera 1 item por página do sinal (1 a 23)
      │
      ▼
Buscar na web (HTTP)       ← Requisição ao sinal por página
      │
      ▼
Processar e Normalizar     ← Extrai e estrutura os campos de cada registro
      │
      ▼
Agregar Todos os Cruzeiros ← Consolida todos os itens em um único array
      │
      ▼
Consolidar e Deduplicar    ← Ordena por data e remove duplicatas por sailingCode
      │
      ▼
Expandir para Linhas       ← Converte para itens individuais prontos para exportação
```

---

## 📋 Workflow

### `Scraping de Dados`

Workflow de execução manual. Deve ser rodado sempre que for necessário atualizar a base de cruzeiros disponíveis.

**Fluxo:**

```
Início (Manual Trigger)
  → Gerar Lista de Páginas (Code)
      Gera 23 itens, um por página do sinal:
        - page: número da página (1 a 23)
        - totalPages: total de páginas da fonte

  → Buscar na web (HTTP Request)
      Coleta o sinal de cada página da API Kroóze

  → Processar e Normalizar Dados (Code)
      Para cada registro recebido do sinal, extrai:
        - Identificação: codigoViagem, codigoItinerario
        - Rota: embarque, desembarque (nome + código)
        - Destino e itinerário: destino, itinerario
        - Companhia e navio: companhiaCruzeiro, navio (nome + código)
        - Datas: dataPartida (dd/mm/aaaa), noites
        - Preços: melhorPrecoTotal, melhorPrecoCabine, precosCabine (por tipo de cabine)
        - Extras: beneficios, vooDisponivel, pacotePrePos, reservaOnline

  → Agregar Todos os Cruzeiros (Aggregate)
      Consolida todos os registros processados em um único campo "cruzeiros"

  → Consolidar e Deduplicar (Code)
      - Ordena por dataPartida (crescente)
      - Remove duplicatas pelo codigoViagem
      - Retorna: totalColetados, dataExtracao, cruzeiros[]

  → Expandir para Linhas (Code)
      Converte o array "cruzeiros" em itens individuais
      → Pronto para nó de exportação (Google Sheets, Excel, banco de dados, etc.)
```

**Estrutura do registro normalizado:**

```json
{
  "codigoViagem": "...",
  "codigoItinerario": "...",
  "dataPartida": "dd/mm/aaaa",
  "noites": 7,
  "embarque": "...",
  "desembarque": "...",
  "destino": "...",
  "itinerario": "...",
  "companhiaCruzeiro": "...",
  "navio": "...",
  "melhorPrecoTotal": 0.00,
  "melhorPrecoCabine": 0.00,
  "precosCabine": "{}",
  "beneficios": "...",
  "vooDisponivel": true,
  "pacotePrePos": false,
  "reservaOnline": true
}
```

---

## 🔌 Integrações

| Serviço | Uso |
|---|---|
| API Kroóze | Fonte do sinal paginado de cruzeiros |

---

## ⚙️ Configuração

### Pré-requisitos

- Instância n8n ativa (self-hosted ou cloud)
- Acesso ao sinal da plataforma Kroóze
- Destino de exportação definido (Google Sheets, Excel, banco de dados, etc.)

### Instalação

1. **Importe o workflow** `Scraping_de_dados.json` no n8n.

2. **Configure o nó `Buscar na web`** (atualmente desabilitado no arquivo):
   - Defina a URL do sinal da API Kroóze
   - Configure a autenticação conforme necessário
   - Passe o parâmetro de página via query string

3. **Ajuste o total de páginas** no nó `Gerar Lista de Páginas` conforme a paginação atual da fonte:
   ```javascript
   const TOTAL_PAGES = 23;
   ```

4. **Conecte um nó de exportação** após `Expandir para Linhas` — Google Sheets, Excel ou banco de dados.

5. **Execute manualmente** clicando em **Execute Workflow**.

---

## 🏷️ Tags

`n8n` `scraping` `kroóze` `cruzeiros` `api` `paginação` `etl` `dados`
