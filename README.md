# Dashboard Gestão Comercial — Lucralize

Dashboard de Business Intelligence para acompanhamento comercial da Lucralize Contabilidade e Lucralize Tech, com dados lidos diretamente do SharePoint via Microsoft Graph API.

**URL de produção:**
`https://lucralize-comercial.github.io/gestao-comercial/dashboard-lucralize-sharepoint.html`

---

## Arquitetura

```
GitHub Pages (frontend estático)
    └── dashboard-lucralize-sharepoint.html
            ├── Autenticação PKCE OAuth 2.0 (Microsoft Entra ID)
            ├── Microsoft Graph API → SharePoint Lists
            └── Agendor Proxy (Railway) → CRM Agendor
```

O dashboard é um **arquivo HTML único** sem dependências de build. Todo o JavaScript está inline. Os dados são carregados em tempo de execução diretamente do SharePoint do tenant da Lucralize.

---

## Configuração técnica

| Parâmetro | Valor |
|---|---|
| **App Azure** | `lucralize-gestao-comercial` |
| **Client ID** | `c0868f3b-764c-4c5b-a9fc-4af4b6eb0baf` |
| **Tenant ID** | `5173aa83-66e1-49f3-9128-f2251b43294d` |
| **Redirect URI** | `https://lucralize-comercial.github.io/gestao-comercial/dashboard-lucralize-sharepoint.html` |
| **Autenticação** | PKCE OAuth 2.0 (Authorization Code Flow) |
| **Token TTL** | 4 horas (salvo em localStorage) |

### SharePoint — Lucralize Contabilidade
- **Site:** `lucralize.sharepoint.com/`
- **Lista:** `Controle de Clientes`

### SharePoint — Lucralize Tech
- **Site:** `lucralize.sharepoint.com/sites/LUCRALIZETECH`
- **Lista:** `Base de Usuários`

### Agendor Proxy
- **URL:** `https://agendo-proxy-production.up.railway.app`
- **Repositório:** `lucralize-comercial/agendo-proxy`
- **Usado em:** aba Compartilhar → card CRM Agendor

---

## Permissões necessárias no Azure AD

No registro do app (`lucralize-gestao-comercial`), as seguintes permissões delegadas devem estar concedidas:

- `Sites.Read.All` — leitura dos dados do SharePoint
- `User.Read` — identificação do usuário autenticado

---

## Estrutura de campos SharePoint

### Contabilidade
| Campo lógico | Nome interno SharePoint |
|---|---|
| Situação | `Situa_x00e7__x00e3_o` |
| Data de entrada | `DatadoContrato` |
| Competência de saída | `Compet_x00ea_nciadeSa_x00ed_da` |
| Honorário | `Honor_x00e1_rio` |
| Regime tributário | `Regime_x0020_Tribut_x00e1_rio_x0020_Ano_x0020_Atual` |
| Responsável | `Respons_x00e1_vel_x0020_Cont_x00e1_bil` |
| Motivo de saída | `Motivo_x0020_da_x0020_Sa_x00ed_da` |
| Tipo de entrada | `Tipo_x0020_de_x0020_entrada` |
| Núcleo | `N_x00fa_cleo` |
| Origem | `Origem_x0020_` |

### Tech
| Campo lógico | Nome interno SharePoint |
|---|---|
| Situação | `Situa_x00e7__x00e3_o` |
| Competência de entrada | `field_11` (MM/AAAA) |
| Data do contato | `field_16` (ISO — usada no gráfico semanal) |
| Competência de saída | `field_26` (MM/AAAA) |
| Plano | `PlanoLookupId` (1=Essencial, 2=Exclusivo, 3=Plus) |
| Valor mensal | `Valor` |
| Estado | `field_7` |
| Responsável | `Respons_x00e1_vel_x0028_Escolha_` |
| Tipo de entrada | `TipodeEntrada` |
| Motivo de saída | `MotivodaSa_x00ed_da` |
| Regime tributário | `RegimeTribut_x00e1_rioAnoAtual` |
| Forma de pagamento | `FormadePagamento` |
| Origem | `Origem` |

**Planos Tech:**
- `1` = Essencial — R$ 189/mês
- `2` = Exclusivo — R$ 259/mês
- `3` = Plus — R$ 469/mês

> O campo `Valor` tem prioridade sobre o valor do plano. Se estiver vazio, usa o valor fixo do plano.

---

## Abas do dashboard

### 📊 Visão Geral
Consolidado Contabilidade + Tech. Inclui:
- Cards Meta vs Realizado (Clientes Ativos, Entradas, Saídas, MRR Consolidado, Faturamento Anual)
- KPIs do mês anterior e mês atual (Entradas e Saídas)
- Gráficos de evolução histórica e MRR
- Distribuição por origem, regime, tipo de entrada

### 💻 Lucralize Tech
Dados exclusivos da base Tech. Mesma estrutura da Visão Geral, com adição de distribuição por plano, estado e responsável.

### 📋 Contabilidade
Dados exclusivos da base Contabilidade. Inclui distribuição por núcleo, classificação e responsável.

### 🚀 Nosso Crescimento
- Histórico anual 2021–2026
- Sazonalidade editável (média dos últimos 3 anos)
- Projeção acumulada 2026 com gap de entradas
- Acompanhamento mensal MVR (Meta vs Realizado) por mês

### 💳 Migração de Pagamento
Acompanhamento da migração de boleto para cartão de crédito recorrente (Tech):
- Cards por status: Cartão de Crédito, Boleto, Acionar próximo mês, Verificar se foi concretizado, Não definido
- Barra segmentada proporcional por status
- Lista de clientes ativos dividida em 3 colunas por plano (Essencial · R$189 / Exclusivo · R$259 / Plus · R$469)
- Filtro por forma de pagamento com contagem `X de Y`

### 📤 Compartilhar
Resumo executivo para o time de gestão:
- Cards MVR Contab + Tech lado a lado
- Gráficos mensais de Entradas e Saídas
- Saldo líquido (mês anterior + atual)
- Top motivos de saída
- CRM Agendor (via proxy Railway) com filtro por funil
- Evolução semanal (últimos 6 meses)
- Botão Imprimir / Salvar PDF

---

## Lógica de negócio

### Status de clientes
Valores possíveis: `ATIVO`, `INATIVO`, `BAIXADA`, `SUSPENSO`, `EM CANCELAMENTO`

- **Saídas** contabilizam: `INATIVO`, `BAIXADA` e `EM CANCELAMENTO` com data de competência preenchida
- **SUSPENSO** não tem data de saída — aparece apenas como snapshot atual
- **EM CANCELAMENTO** é contado nas saídas mas exibido separadamente nos KPIs

### Cálculo de MRR
Soma dos honorários/valores dos clientes com status `ATIVO` no momento da consulta.

### Cálculo de Faturamento Anual 2026
Para cada cliente ativo em 2026:
- `mês_início` = `max(jan/2026, mês de entrada do cliente)`
- `mês_fim` = `min(dez/2026, mês de saída)` — ou `dez/2026` se ainda ativo
- `contribuição` = `honorário × número de meses`

### Crescimento incremental
O sufixo das barras de progresso nos cards MVR usa crescimento incremental para Clientes Ativos:

```
% = (real - base_2025) / (meta - base_2025)
sufixo = "X de Y" onde X = crescimento realizado, Y = crescimento planejado
```

Para Entradas, Saídas, MRR e Faturamento, a base é zero (acumulado do ano).

---

## Metas 2026

```js
META = {
  ativos_total: 750,   ativos_contab: 263,  ativos_tech: 487,
  entradas_total: 473, entradas_contab: 93,  entradas_tech: 380,
  saidas_total: 212,   saidas_contab: 55,   saidas_tech: 110,
  mrr_contab: 205000,  mrr_tech: 114000,
  fat_contab: 2460000, fat_tech: 1370000,   fat_total: 3830000,
  ticket_contab: 780,  ticket_tech: 234
}
```

---

## Deploy

O dashboard é hospedado via **GitHub Pages** no repositório `lucralize-comercial/gestao-comercial`, branch `principal`, pasta raiz.

Para atualizar:
1. Baixar o arquivo `dashboard-lucralize-sharepoint.html` gerado
2. Fazer upload no repositório substituindo o arquivo existente
3. O GitHub Pages publica automaticamente em 1–2 minutos

### Configuração do Azure AD
Em caso de mudança de URL, atualizar o **Redirect URI** em:
`portal.azure.com → Registros de aplicativo → lucralize-gestao-comercial → Autenticação`

E também atualizar a variável `redirectUri` dentro do arquivo HTML (linha ~572).

---

## Dependências externas

| Biblioteca | Versão | Uso |
|---|---|---|
| Chart.js | 4.4.1 | Todos os gráficos |
| Comfortaa | Google Fonts | Títulos e logo |
| Poppins | Google Fonts | Corpo do texto |

Nenhum framework JS (React, Vue, etc.). Sem processo de build. Arquivo estático puro.
