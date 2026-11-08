# TP1 SI — Decisões do modelo multidimensional

**Grupo:** Tiago Sanina (n.º 20318), Benvinda, Soares · **Entrega:** 16/11/2026
**Tema:** Energia (E-REDES + IPMA + Pordata) · **Tarefa:** Desenhar modelo multidimensional (Sprint 1, 3/11/2026)

> Notas para o relatório. Cada decisão tem a justificação, para a defesa ("explicar, justificar e reconstruir").

---

## 1. Fontes de dados

| Fonte | Formato | Conteúdo | Grão original | Período |
|---|---|---|---|---|
| E-REDES | CSV (`;`, UTF-8 com BOM) | Energia ativa faturada | mês × freguesia × nível de tensão | nov/2020 a mai/2026 |
| IPMA | 18 CSV (1 por distrito) | Temperatura mín./média/máx. + normais 8110 e 9120 | mês × distrito | 01/1981 a 09/2026 |
| Pordata (INE) | XLSX | População residente a 31/12, total/homens/mulheres | ano × município | 2020 a 2025 |
| UPAC mensal (E-REDES) | XLSX (15 MB) | Unidades de produção para autoconsumo: n.º de instalações e potência instalada (kW) | mês × freguesia × código postal × tecnologia × nível de tensão × escalão | jan/2023 a set/2026 |

### Factos verificados nos dados
- E-REDES: 300 003 linhas, sem nulos em Energia Ativa, 18 distritos, 296 concelhos (278 reais + 18 pseudo-concelhos `OUTROS <DISTRITO>`), 2 936 freguesias.
- O grão (mês, freguesia, nível de tensão) é único: 300 003 combinações distintas, 0 duplicados.
- 85 linhas com nome de freguesia vazio (código sempre preenchido).
- Dois níveis de tensão: "Baixa Tensão" e "Muito Alta, Alta e Média Tensões".
- IPMA: 549 meses por distrito (18 × 549 = 9 882 linhas), sem coluna de distrito (só no nome do ficheiro).
- Pordata: 308 municípios (inclui ilhas), anos em colunas, cabeçalhos nas linhas 1–12, agregados NUTS misturados com municípios, quebra de série em 2021 (valores revistos pelo INE).
- UPAC: 531 758 linhas, 45 meses (2023-01 a 2026-09), 18 distritos, 278 concelhos, 3 179 freguesias; sem vazios e sem duplicados no grão de 6 colunas. Códigos de distrito/concelho/freguesia no mesmo formato que a E-REDES. **É um stock** (cada mês mostra o acumulado: 132 120 instalações em jan/2023 e 295 470 em set/2026), logo as medidas são semi-aditivas.
- UPAC: 99,26% da potência é Solar (set/2026). Valores sujos: `Fotovoltaica` (= Solar), `Não Atribuído`, `nan` como texto (21 linhas na tecnologia, 15 no nível de tensão). Níveis de tensão `BTN`, `BTE`, `MT`, `AT` (a E-REDES tem só 2 grupos). Freguesias: 2 913 comuns com a E-REDES, 266 só na UPAC e 23 só na E-REDES (alterações à divisão administrativa).

---

## 2. Tipo de modelo

**Constelação de factos (*fact constellation / galaxy schema*) com a dimensão Geografia em floco de neve (*snowflake*).**

- Não é um esquema em estrela puro: tem **quatro** tabelas de factos, a geografia está em **três tabelas** encadeadas e o nível de tensão em **duas**.
- **Porquê constelação:** as fontes têm grãos diferentes (freguesia/mês, distrito/mês, município/ano, freguesia/mês/tecnologia/tensão). Forçá-las numa só tabela obrigaria a repetir valores (por exemplo, a temperatura em todas as freguesias do distrito) e a somar medidas não aditivas.
- **Porquê floco de neve na geografia e na tensão:** cada facto liga ao nível em que os seus dados existem (consumo ao grupo de tensão, UPAC ao nível de tensão; consumo e UPAC à freguesia, população ao concelho, temperatura ao distrito), sem criar linhas artificiais numa dimensão única.
- **Alternativa rejeitada (estrela):** uma `dim_geografia` desnormalizada, com linhas extra para concelho e distrito (freguesia vazia). Foi rejeitada porque mistura níveis na mesma tabela e há risco de contar linhas a mais ao somar o consumo por distrito.

---

## 3. Tabelas de factos

### 3.1 `fact_consumo`
- **Grão:** uma linha por **mês**, por **freguesia** e por **nível de tensão**.
- **Chave:** PK composta (`id_data`, `cod_freguesia`, `id_grupo_tensao`).
- **Medida:** `energia_ativa` — **aditiva**, soma em todas as dimensões (tempo, geografia, tensão).
- **Dimensões:** `dim_tempo`, `dim_freguesia`, `dim_grupo_tensao`.

### 3.2 `fact_temperatura`
- **Grão:** uma linha por **distrito** e por **mês**.
- **Chave:** PK composta (`id_data`, `cod_distrito`).
- **Medidas (todas **não aditivas**):**

| Medida | Origem (IPMA) | Agregação ao subir no tempo |
|---|---|---|
| `temp_media` | MÉDIA MENSAL (AVG) | média |
| `temp_min` | MÉDIA MENSAL (MIN) | mínimo |
| `temp_max` | MÉDIA MENSAL (MAX) | máximo |
| `normal_8110` | NORMAL MENSAL 8110 | média |
| `normal_9120` | NORMAL MENSAL 9120 | média |

- **Dimensões:** `dim_tempo`, `dim_distrito`.
- **Justificação das 5 medidas:** o custo é residual (≈1 200 linhas úteis) e permite analisar amplitude térmica (mín./máx.) e desvio face ao normal climatológico. Acrescentar uma medida depois de fazer o ETL dá mais trabalho do que tirá-la dos relatórios.
- **Regra:** nunca somar temperaturas (a soma de 12 médias mensais dá ≈200 °C, sem sentido físico).

### 3.3 `fact_populacao`
- **Grão:** uma linha por **município** e por **ano** (ligado a **dezembro** desse ano).
- **Chave:** PK composta (`id_data`, `cod_concelho`).
- **Medidas:** `pop_total`, `pop_homens`, `pop_mulheres` — **semi-aditivas**:
  - **geografia:** soma (municípios → distrito);
  - **tempo:** **não** se soma; usar **média** ou **último valor** do período (é um retrato a 31/12).
- **Sexo como medidas (3 colunas)** e não como dimensão: mais simples e o sexo não é central na análise de energia. O total é redundante mas conveniente.
- **Dimensões:** `dim_tempo` (mês de dezembro), `dim_concelho`.

### 3.4 `fact_energia_sustentavel` (UPAC)
- **Grão:** uma linha por **mês**, por **freguesia**, por **tecnologia** e por **nível de tensão**.
- **Chave:** PK composta (`id_data`, `cod_freguesia`, `id_tecnologia`, `id_nivel_tensao`).
- **Medidas:** `n_instalacoes` e `potencia_kw` — **semi-aditivas** (cada mês é um **stock acumulado**): soma na geografia, tecnologia e tensão; no tempo, **último valor** ou **média**, nunca soma.
- **Dimensões:** `dim_tempo`, `dim_freguesia`, `dim_tecnologia`, `dim_nivel_tensao`.
- **Grão agregado e não fino:** o ficheiro tem também código postal e escalão de potência. Foram deixados de fora: o código postal não tem dimensão no modelo (a freguesia já dá a geografia) e inflaciona as linhas; o escalão pode ser acrescentado depois como pequena dimensão, se houver tempo. Agregar somando as linhas do mesmo mês é válido (soma de stocks no mesmo instante).
- **Ligação com o consumo:** partilham mês, freguesia e tensão; cruzam-se ao nível do **grupo de tensão**.
- **Período comum com a E-REDES:** jan/2023 a mai/2026.

---

## 4. Dimensões

### 4.1 `dim_tempo`
- **Grão:** mês (não há dia em nenhuma fonte; o nível mais fino é o mês).
- **Chave:** `id_data` inteiro no formato `AAAAMM` (ex.: `202104`). É legível, ordena cronologicamente e é fácil de derivar das três fontes.
- **Atributos:** mês (número e nome), trimestre, ano, estação do ano (opcional).
- **Hierarquia:** **Ano > Trimestre > Mês**.
- **Semestre não incluído:** não acrescenta análise útil face ao ano e ao trimestre.
- **Trimestre incluído:** permite ver sazonalidade e *drill-down* sem ir mês a mês.
- **Âmbito:** nov/2020 a set/2026 (união das fontes: E-REDES até mai/2026, IPMA até set/2026, Pordata até dez/2025). 2020 e 2026 são anos incompletos — a ter em conta nas comparações ano a ano.

### 4.2 Geografia (floco de neve)
Hierarquia: **Distrito > Concelho > Freguesia**. Todas as chaves são **códigos em texto** (preservar zeros à esquerda; `0101` lido como número vira `101` e a ligação falha em silêncio).

| Tabela | Chave (PK) | Atributos | FK |
|---|---|---|---|
| `dim_distrito` | `cod_distrito` (`coddistrito`) | `nome_distrito` | — |
| `dim_concelho` | `cod_concelho` (`coddistritoconcelho`, 4 dígitos) | `nome_concelho` | `cod_distrito` |
| `dim_freguesia` | `cod_freguesia` (`coddistritoconcelhofreguesia`) | `nome_freguesia` ("Não definida" se vazio) | `cod_concelho` |

- **Códigos e não nomes como chave:** nomes podem repetir-se ou vir com erros de acentuação; o código DICO é a ligação natural entre E-REDES e Pordata.

### 4.3 Nível de tensão (floco de neve, grupo > nível)
A E-REDES tem 2 grupos de tensão e a UPAC tem 4 níveis, por isso a dimensão é dividida em duas tabelas:

| Tabela | Chave (PK) | Atributos | FK |
|---|---|---|---|
| `dim_grupo_tensao` | `id_grupo_tensao` (1, 2) | `nome_grupo`: "Baixa Tensão", "Muito Alta, Alta e Média Tensões" | — |
| `dim_nivel_tensao` | `id_nivel_tensao` (1 a 4) | `sigla`: BTN, BTE, MT, AT | `id_grupo_tensao` |

- Correspondência proposta: `BTN` e `BTE` → Baixa Tensão; `MT` e `AT` → Muito Alta, Alta e Média Tensões. **A validar** com o glossário da E-REDES/ERSE (a citar nas referências).
- `fact_consumo` liga ao **grupo**; `fact_energia_sustentavel` liga ao **nível**.
- Tabela própria (e não dimensão degenerada na tabela de factos): coerência com o resto do modelo, espaço para atributos futuros e segmentações mais limpas no Power BI.

### 4.4 `dim_tecnologia`
- **Chave:** `id_tecnologia` (1 = Solar, 2 = Outras). **Atributo:** `nome_tecnologia`.
- **Solar:** `Solar` + `Fotovoltaica` (mesma tecnologia). **Outras:** Eólica, Biogás, Hídrica, Biomassa, Cogeração (renovável e não renovável) e `Não Atribuído`.
- `nan` (21 linhas na tecnologia, 15 no nível de tensão): removidos na limpeza.
- Justificação: o solar pesa 99,26% da potência em set/2026; as restantes tecnologias são residuais. `Não Atribuído` mantido em "Outras" (0,23% da potência, 5 instalações em set/2026) para não perder instalações reais.

---

## 5. Ligações entre factos e dimensões

| Facto | `dim_tempo` | `dim_freguesia` | `dim_concelho` | `dim_distrito` | `dim_grupo_tensao` | `dim_nivel_tensao` | `dim_tecnologia` |
|---|---|---|---|---|---|---|---|
| `fact_consumo` | `id_data` | `cod_freguesia` | (via freguesia) | (via concelho) | `id_grupo_tensao` | — | — |
| `fact_energia_sustentavel` | `id_data` | `cod_freguesia` | (via freguesia) | (via concelho) | (via nível) | `id_nivel_tensao` | `id_tecnologia` |
| `fact_populacao` | `id_data` (dezembro) | — | `cod_concelho` | (via concelho) | — | — | — |
| `fact_temperatura` | `id_data` | — | — | `cod_distrito` | — | — | — |

Cadeias: `dim_freguesia` → `dim_concelho` → `dim_distrito` e `dim_nivel_tensao` → `dim_grupo_tensao`.
As quatro tabelas de factos partilham `dim_tempo`; o **ponto de encontro geográfico** é o distrito (e o concelho para consumo vs população, e a freguesia para consumo vs energia sustentável). O ponto de encontro da tensão é o **grupo**.

---

## 6. Decisões de integração e tratamento de dados (ETL)

| Tema | Decisão | Justificação / consequência |
|---|---|---|
| População anual vs grão mensal | Ligar cada linha anual ao **mês de dezembro** | O Pordata é o retrato a 31/12. Evita repetir a população 12 vezes (que inflaria somas). A população deve usar-se ao nível do **ano**; por mês ou trimestre aparecem meses vazios. |
| IPMA sem coluna de distrito | Corrigir nomes dos ficheiros (`Bragan_a`, `_vora`, `Satar_m`, `Set_bal`…), criar tabela de correspondência `Distrito ↔ coddistrito` a partir dos 18 pares distintos da E-REDES e juntar pelo nome | Valida-se que as 18 junções funcionam; qualquer linha sem código indica nome mal corrigido. |
| `OUTROS` (18, um por distrito) | **Manter** como concelho "Outros" em cada distrito | Mantém o consumo total por distrito correto (RGPD: consumo agregado sem identificação). O Pordata não tem "Outros", pelo que ficam **fora** de análises *per capita*. Confirmar que têm código de freguesia. |
| Ilhas (30 municípios do Pordata) | **Filtrar** | Não há dados de E-REDES nem IPMA para as ilhas (ambos só Continente). |
| Freguesias sem nome (85) | Manter com nome "Não definida" | O código existe sempre; perde-se só o nome. |
| Energia Ativa lida como texto | Converter para decimal (ponto decimal) | Verificado no Power BI: sem símbolo Σ. |
| `coddistrito` lido como número | Converter para texto | Preservar zeros à esquerda. |
| Pordata (cabeçalhos/anos em colunas) | Remover linhas 1–12, promover cabeçalhos, *unpivot* dos anos, separar agregados NUTS dos municípios, ignorar legenda e colunas/linhas vazias | Formato original não está pronto para análise. |
| Quebra de série 2021 (Pordata) | Registar como limitação; cuidado em comparações 2020 vs 2021+ | Valores de 2021–2024 revistos pelo INE em 22/06/2026. |
| Série do IPMA | Filtrar para o período em comum (2020–2026) | O IPMA tem 45 anos, mas só se cruza com as outras fontes de 2020 em diante. |
| Anos incompletos (2020, 2026) | Manter, assinalando a limitação | 2020 só tem nov–dez; 2026 vai até mai (E-REDES). |
| UPAC: ficheiro XLSX de 15 MB | Converter para CSV no ETL | Carregamento mais leve no Power BI. |
| UPAC: tecnologia suja | Juntar `Solar` e `Fotovoltaica`; `Não Atribuído` fica em "Outras"; remover `nan` | Ver 4.4. |
| UPAC: nível de tensão `nan` (15 linhas) | Remover | Residual, não identifica nenhum nível. |
| UPAC: grão com código postal e escalão | Agregar para mês × freguesia × tecnologia × nível de tensão, somando potência e instalações | Mesmo grão que o consumo; ver 3.4. |
| UPAC: freguesias que não existem na E-REDES (266) | A decidir: incluir em `dim_freguesia` (união das duas fontes) ou excluir | Pendente. Afeta a análise consumo vs UPAC por freguesia. |
| UPAC: stock acumulado | Medidas semi-aditivas: último valor ou média no tempo | Evita somar acumulados de vários meses. |

---

## 7. Limitações a mencionar no relatório
- IPMA e E-REDES cobrem só o **Continente**.
- Período em comum curto: ~nov/2020 a 2025.
- Granularidades diferentes (freguesia/mês, distrito/mês, município/ano): a ligação entre fontes é feita a níveis agregados.
- Temperatura é **modelada** (IPMA) e igual para todas as freguesias do distrito.
- População com quebra de série em 2021.
- Tabela de consumo **esparsa** (nem todas as freguesias têm dados todos os meses): ≈4 500 linhas por mês para 2 936 freguesias × 2 níveis de tensão possíveis.
- UPAC só começa em **jan/2023**: o cruzamento consumo vs produção solar cobre jan/2023 a mai/2026.
- UPAC: o nível de tensão (4 níveis) tem de ser alinhado com os 2 grupos da E-REDES; a correspondência BTE → Baixa Tensão deve ser validada em fonte externa.
- UPAC mostra potência **instalada** (capacidade), não energia produzida: não se pode comparar diretamente com o consumo em kWh sem cuidado.

---

## 9. Registo de uso de IA (para a Declaração de Uso de IA)
- **Ferramenta:** Claude (Anthropic), em modo tutor.
- **Tarefas apoiadas:** análise dos datasets; explicação de conceitos (grão, aditividade, dimensões, tipos de esquema); revisão das minhas propostas.
- **Prompts usados:** frases do tipo *"… analisa a minha proposta de solução de …"* e *"Não me dês a solução; quero que sejas o meu tutor."*
- **Verificação humana:** decisões tomadas por mim a cada passo; verifiquei nos dados o grão único, as contagens (linhas, concelhos, freguesias) e o diagrama final.