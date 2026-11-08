# Sprint 1 – Extração e perfil de qualidade das fontes

**Responsável:** Soares · **Data:** 02/11/2026 · **Ferramenta:** Power BI Desktop (Power Query)
**Ficheiro de trabalho:** `exploracao_fontes.pbix`

## 1. Fontes escolhidas

| # | Fonte | Dataset | Formato | Granularidade | Período | Download |
|---|---|---|---|---|---|---|
| 1 | E-Redes Open Data | Consumos mensais por concelho | CSV | Mensal · distrito / concelho / freguesia · nível de tensão | nov-2020 a mai-2026 | 02/11/2026 |
| 2 | Pordata (INE) | População residente, estimativas a 31 de dezembro: total e por sexo (Municípios) | XLSX | Anual · município · sexo | 2020–2025 | 02/11/2026 |
| 3 | IPMA – DataClima | Temperatura média mensal do ar, por distrito (18 ficheiros) | CSV | Mensal · distrito | 01/1981 a 09/2026 | 02/11/2026 |

**Justificação:** 3 fontes e 2 formatos (CSV e XLSX). A E-Redes fornece os factos (consumo de energia). A Pordata e o IPMA dão contexto (população → consumo por habitante; temperatura → relação clima/consumo). As fontes ligam-se por concelho/distrito e por ano/mês.

## 2. Perfil de qualidade

Análise feita no Power Query com *Column quality*, *Column distribution* e *Column profile* sobre o **conjunto de dados completo**.

### 2.1 E-Redes – Consumos mensais por concelho
- 300 003 linhas, 12 colunas.
- Colunas: Ano, Mês, Data, mes_int, Distrito, Concelho, Freguesia, Nível de Tensão, Energia Ativa, coddistrito, coddistritoconcelho, coddistritoconcelhofreguesia.
- 18 distritos (só continente), 296 concelhos (278 reais + 18 pseudo-concelhos "OUTROS <DISTRITO>"), 2 936 freguesias (por código; 2 722 nomes distintos), 2 níveis de tensão.
- Problemas identificados:
  - `Energia Ativa` lida como texto (decimal com ponto e Power BI em português).
  - 2020 (só nov–dez) e 2026 (até mai) são anos incompletos.
  - 18 pseudo-concelhos "OUTROS <DISTRITO>" (um por distrito, 1 344 linhas): consumos agregados por RGPD.
  - `Freguesia` com 85 linhas vazias (o código existe sempre); nomes em maiúsculas, sem acentos e com abreviaturas ("UF ...").
  - Colunas redundantes: `mes_int` = `Mês`; `Data` = `Ano` + `Mês`.
- Colunas de código: 0% erros e 0% vazios.

### 2.2 Pordata – População residente por município
- Folha "Quadro": 394 linhas; dados úteis nas colunas 1–20 (6 anos × total/masculino/feminino).
- 308 municípios (inclui 30 das Regiões Autónomas).
- Problemas identificados:
  - Título e cabeçalhos repartidos pelas linhas 1–12.
  - Formato largo (anos em colunas) → precisa de *unpivot*.
  - Agregados (Portugal, NUTS I/II/III) misturados com municípios, identificáveis pela coluna de nível.
  - Legenda de símbolos no fim do ficheiro (linhas 382–386) → 5 erros de tipo.
  - Colunas e linhas vazias (formatação do Excel).
  - Quebra de série em 2021 (⊥, Censos); valores de 2021–2024 revistos pelo INE a 22/06/2026.
- Folha "Códigos": código DICO de 4 dígitos por município (texto). Permite ligar à E-Redes (`coddistritoconcelho`, numérico, sem zero inicial).

### 2.3 IPMA – Temperatura média mensal por distrito
- 18 ficheiros CSV (um por distrito) combinados com *Get data → Folder*: 9 882 linhas (18 × 549 meses).
- Colunas: Date, média mínima, média, média máxima, normal 1981-2010, normal 1991-2020 (°C).
- Problemas identificados:
  - Sem coluna de distrito: o distrito só se identifica pelo nome do ficheiro (`Source.Name`).
  - Temperaturas lidas como texto (decimal com ponto).
  - Codificação UTF-8 (no Excel os acentos aparecem mal: "MÃ‰DIA").
  - Série muito mais longa do que o necessário (filtrar 2020–2026).
  - Dados modelados (interpolação), não medidos numa estação.
- 0% erros e 0% vazios.

## 3. Decisões tomadas
1. Escolha de 3 fontes complementares (E-Redes, Pordata, IPMA).
2. IPMA ao nível do **distrito** (18 séries) em vez do concelho (278), para reduzir o volume de downloads.
3. Ficheiros do IPMA renomeados na extração com o nome do distrito igual ao da E-Redes (ex.: `algarve.csv` → `Faro.csv`).

> Nota: no perfil inicial só foram identificados 2 registos "OUTROS", porque a lista do filtro do Power Query se baseia na pré-visualização e estava incompleta. A contagem completa (18) foi confirmada depois sobre o ficheiro inteiro.
4. Nesta fase não se fizeram transformações: os dados ficam no estado original para documentar a qualidade.
5. Desligada a deteção automática de relações no Power BI (as relações serão definidas no modelo em estrela).

## 4. Ficheiros entregues
- `dados_brutos/` – CSV da E-Redes, `pordata.xlsx`, pasta com os 18 CSV do IPMA.
- `exploracao_fontes.pbix` – perfil de qualidade das 3 fontes.

> Nota: ao abrir o `.pbix` noutro computador, corrigir os caminhos em *Transform data → Data source settings → Change Source*.
