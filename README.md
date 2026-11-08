# TP1 SI — Modelo Multidimensional: Energia em Portugal

Trabalho prático de Sistemas de Informação: desenho de um modelo multidimensional (data warehouse) que cruza dados de consumo de energia, produção solar para autoconsumo, temperatura e população, por município/freguesia em Portugal Continental.

**Grupo:** Tiago Sanina (n.º 20318), Benvinda, Soares
**Entrega:** 16/11/2026
**Tema:** Energia (E-REDES + IPMA + Pordata)

## Fontes de dados

| Fonte | Formato | Conteúdo | Grão original | Período |
|---|---|---|---|---|
| E-REDES | CSV | Energia ativa faturada | mês × freguesia × nível de tensão | nov/2020 a mai/2026 |
| IPMA | 18 CSV (1 por distrito) | Temperatura mín./média/máx. + normais climatológicas | mês × distrito | 1981 a 2026 |
| Pordata (INE) | XLSX | População residente | ano × município | 2020 a 2025 |
| UPAC mensal (E-REDES) | XLSX | Unidades de produção para autoconsumo (instalações e potência) | mês × freguesia × tecnologia × nível de tensão | jan/2023 a set/2026 |

## Modelo

Constelação de factos (*fact constellation*) com a dimensão Geografia em floco de neve (Distrito > Concelho > Freguesia), com quatro tabelas de factos:

- `fact_consumo` — energia ativa faturada (mês × freguesia × nível de tensão)
- `fact_temperatura` — temperatura e normais climatológicas (mês × distrito)
- `fact_populacao` — população residente (ano × município)
- `fact_energia_sustentavel` — produção UPAC/solar (mês × freguesia × tecnologia × nível de tensão)

A justificação detalhada de todas as decisões (tipo de modelo, grão e aditividade das medidas, dimensões, ligações entre factos e dimensões, tratamento de dados no ETL e limitações) está em [Ficheiros_para_relatório/decisoes_modelo_tp1.md](Ficheiros_para_relatório/decisoes_modelo_tp1.md).

## Estrutura do repositório

```
├── Diagrama/                          # Diagrama do modelo multidimensional (draw.io + imagem)
├── Ficheiros_para_relatório/          # Notas e justificações de decisões, para o relatório
├── dados/
│   ├── exploracao_fontes.pbix         # Exploração inicial dos dados no Power BI
│   └── ficheirosBrutos/               # Dados brutos das fontes (E-REDES, IPMA, Pordata, UPAC)
└── logs/
    └── log_limpeza_etl_TP1.xlsx       # Registo das operações de limpeza/ETL
```

## Ferramentas

- **Power BI** — exploração dos dados e construção do modelo
- **draw.io** — diagrama do modelo multidimensional
