# TP1 SI — Modelo Multidimensional: Energia em Portugal

Trabalho prático de Sistemas de Informação: desenho de um modelo multidimensional (data warehouse) que cruza dados de consumo de energia, produção solar para autoconsumo, temperatura e população, por município/freguesia em Portugal Continental.

| | |
|---|---|
| **Grupo** | Tiago Sanina, Benvinda, Soares |
| **Tema** | Energia (E-REDES) |
| **Entrega** | 16/11/2026 |

## Objetivo

Construir um modelo multidimensional que permita analisar o consumo de energia elétrica e a produção solar para autoconsumo (UPAC) em Portugal Continental, relacionando-os com fatores de contexto como a temperatura e a população residente, a diferentes níveis de granularidade geográfica e temporal.

## Fontes de dados

| Fonte | Formato | Conteúdo | Grão original | Período |
|---|---|---|---|---|
| E-REDES | CSV | Energia ativa faturada | mês × freguesia × nível de tensão | nov/2020 a mai/2026 |
| IPMA | 18 CSV (1 por distrito) | Temperatura mín./média/máx. + normais climatológicas | mês × distrito | 1981 a 2026 |
| Pordata (INE) | XLSX | População residente | ano × município | 2020 a 2025 |
| UPAC mensal (E-REDES) | CSV (originalmente XLSX, convertido no ETL) | Unidades de produção para autoconsumo (instalações e potência) | mês × freguesia × tecnologia × nível de tensão | jan/2023 a set/2026 |

## Modelo

Constelação de factos (*fact constellation*) com a dimensão Geografia em floco de neve (Distrito > Concelho > Freguesia), com quatro tabelas de factos:

| Tabela de factos | Conteúdo | Grão |
|---|---|---|
| `fact_consumo` | Energia ativa faturada | mês × freguesia × nível de tensão |
| `fact_temperatura` | Temperatura e normais climatológicas | mês × distrito |
| `fact_populacao` | População residente | ano × município |
| `fact_energia_sustentavel` | Produção UPAC/solar | mês × freguesia × tecnologia × nível de tensão |

O diagrama completo do modelo está em [Diagrama/modelo_tp1.jpg](Diagrama/modelo_tp1.jpg) (fonte editável em [Diagrama/modelo_tp1.drawio](Diagrama/modelo_tp1.drawio)). A justificação detalhada de todas as decisões — tipo de modelo, grão e aditividade das medidas, dimensões, ligações entre factos e dimensões, tratamento de dados no ETL e limitações — está documentada em [relatorio/](relatorio/).

## Estrutura do repositório

```
├── Diagrama/                          # Diagrama do modelo multidimensional (draw.io + imagem)
├── relatorio/                         # Relatório final e justificação das decisões do modelo
├── dados/
│   ├── exploracao_fontes.pbix         # Exploração inicial dos dados no Power BI
│   └── ficheirosBrutos/               # Dados brutos das fontes (E-REDES, IPMA, Pordata, UPAC)
└── logs/
    └── log_limpeza_etl_TP1.xlsx       # Registo das operações de limpeza/ETL
```

## Ferramentas

- **Github** — Para controlo de versões
- **Trello** — https://trello.com/b/a35TObFL/si-tp1-e-redes-data-warehouse-olap
- **Power BI** — exploração dos dados e construção do modelo
- **draw.io** — diagrama do modelo multidimensional
