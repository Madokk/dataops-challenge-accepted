# Desafio Técnico — Coleta, Processamento e Validação de Dados INMET

##  Contexto

O **INMET** (Instituto Nacional de Meteorologia) publica, por ano, um arquivo ZIP com dados horários de todas as estações meteorológicas do Brasil.

* Cada estação é um **CSV** com todas as variáveis medidas naquela estação.
* Os arquivos estão disponíveis em: [https://portal.inmet.gov.br/dadoshistoricos](https://portal.inmet.gov.br/dadoshistoricos).

 Exemplo de ZIP anual:
[2025.zip](https://portal.inmet.gov.br/uploads/dadoshistoricos/2025.zip)

 Exemplo de arquivo dentro do ZIP:
`INMET_NE_PI_A354_OIEIRAS_01-01-2025_A_31-07-2025.csv`

O padrão do nome inclui:

* **Região** (NE/SE/S/CO/N)
* **UF** (ex.: PI)
* **Código da estação** (ex.: A354)
* **Nome da cidade**
* **Intervalo de datas**

Os CSVs possuem cabeçalhos em PT-BR (com acentos).

---

##  Parte 1 — Coleta e Processamento das Variáveis

### Objetivo

Escrever um script em Python que:

1. Baixe automaticamente o ZIP de um ano informado (ex.: 2025) do portal do INMET.
2. Leia todos os CSVs de estações dentro do ZIP.
3. Para cada estação, gere **dois arquivos CSV processados**, um para cada variável:

#### (a) Precipitação

* **Origem**: `PRECIPITAÇÃO TOTAL, HORÁRIO (mm)`
* **Saída**: CSV com colunas:

  * `datetime` (`YYYY-MM-DDTHH:MM:SS`)
  * `value` (mm)
* **Nome**: `{codigo_da_estacao}.csv`

#### (b) Temperatura do Ar (2 m)

* **Origem**: `TEMPERATURA DO AR - BULBO SECO, HORARIA (°C)`
* **Saída**: CSV com colunas:

  * `datetime` (`YYYY-MM-DDTHH:MM:SS`)
  * `value` (°C)
* **Nome**: `{codigo_da_estacao}.csv`

### Estrutura de diretórios de saída

```
{RAIZ}/inmet_stations/processed/total_precipitation/{ANO}/{CODIGO}.csv
{RAIZ}/inmet_stations/processed/2m_air_temperature/{ANO}/{CODIGO}.csv
```

 Exemplos:

```
./data/inmet_stations/processed/total_precipitation/2025/A701.csv
./data/inmet_stations/processed/2m_air_temperature/2025/A701.csv
```

---

##  Interface Esperada (CLI)

```bash
python inmet_process.py --years 2024 2025 --out-root ./data --workers 4 --station-filter A701,A354
```

### Parâmetros

* `--years`: um ou mais anos.
* `--out-root`: raiz de saída.
* `--workers`: número de workers em paralelo.
* `--station-filter`: processar apenas códigos específicos (opcional).

---

##  Detalhes e Regras de Transformação

* **Data/Hora**: construir `datetime = DATA + HORA` → formato `YYYY-MM-DDTHH:MM:SS`.
* **Encoding**: `latin1`.
* **Delimitador**: `;`.
* **Decimal**: `,` → `.`.
* **Valores faltantes**: descartar vazios, `NaN`, `-9999`, `null`, strings em branco.
* **Cabeçalhos variantes**: normalizar acentos, caixa e espaços.
* **Duplicatas**: se houver `datetime` duplicado → manter **primeiro registro**.
* **Performance**: processar em chunks, não carregar tudo na memória.
* **Logs**: registrar progresso (anos, estações, colunas ausentes, descartes etc.).
* **Resiliência**: retries no download e verificação de integridade.
* **Idempotência**: se a saída já existir, pular (a menos que `--force`).

---

##  Saídas Esperadas

### Dados processados

```csv
# .../total_precipitation/2025/A354.csv
datetime,value
2025-01-01T00:00:00,0.0
2025-01-01T01:00:00,1.2
```

```csv
# .../2m_air_temperature/2025/A354.csv
datetime,value
2025-01-01T00:00:00,24.6
2025-01-01T01:00:00,24.2
```

---

##  Parte 2 — Verificação de Completude dos Dados

### Conceito

Os dados são horários: para um intervalo de N horas, espera-se N registros.

* **Completude** = `valid_records / expected_records`
* `valid_records`: número de linhas com `datetime` único e `value` válido.
* `expected_records`: número de horas entre `start_dt` e `end_dt` (inclusivo).

Exemplo:

* Intervalo: `01/01/2025 00:00:00 → 31/12/2025 23:00:00`
* `365 dias * 24 = 8.760 registros esperados`
* `valid_records = 6.832` → `6.832 / 8.760 = 0,78 (78%)`

---

### Saídas da Parte 2

Gerar relatórios auxiliares:

```
{RAIZ}/total_precipitation_{ANO}.csv
{RAIZ}/2m_air_temperature_{ANO}.csv
```

Formato:

```csv
station_code;completeness
A354;0.78
A701;0.95
```

---

### Como Calcular

1. `start_dt = menor datetime` no CSV processado.
2. `end_dt = maior datetime`.
3. `expected_records = int((end_dt - start_dt).total_seconds()/3600) + 1`.
4. `valid_records = linhas com value numérico válido`.
5. `completeness = valid_records / expected_records` (arredondar 2 casas decimais).

 Regras:

* Estação sem dados → `0.00`.
* `expected_records <= 0` → logar e marcar `0.00`.
* Intervalo com buracos → **não ajustar denominador**.

---

##  Pipeline Recomendado

1. Rodar **Parte 1** → gerar `{variable}/{ANO}/{CODIGO}.csv`.
2. Rodar **Parte 2** → calcular completude por estação.
3. Agregar em um DataFrame por variável.
4. Exportar relatórios (`;` como separador).

---

##  Exemplos de Casos

* **Faixa**: `2025-01-01T00:00:00 → 2025-01-02T23:00:00` → 48 horas.
* `valid_records = 42` → `42/48 = 0.88`.
* Se houver duplicatas removidas → ok.
* Se houver 3 valores vazios → não contam.

---

##  Entregáveis

* **Código-fonte** em Python 3.10+.
* **Organização modular** (funções como `download_zip`, `parse_station_code`, `extract_variable`, `compute_completeness_for_year` etc.).
* **README.md** (este documento).
* **requirements.txt** (dependências).
* **Demonstração** (rodar para 2025, ao menos 2 estações processadas por variável).
* **Testes rápidos** cobrindo parsing, datetime, decimais, completude.

---

##  Critérios de Avaliação

### Obrigatórios

* Parte 1: download, processamento, saída correta.
* Parte 2: cálculo e exportação de completude.
* Tratamento de encoding, separador, decimal, valores inválidos.
* Código modular, legível, README claro.

### Desejáveis

* CLI com múltiplos anos e filtros.
* Paralelismo e barra de progresso.
* Retries e validação no download.
* Testes unitários.
* Linting (ruff/flake8), type hints (mypy).
* Logging estruturado.

---

##  Pontos de Atenção

* Robustez a variações de cabeçalho.
* Idempotência e clareza de logs.
* Tratamento consistente de duplicatas.
* Performance aceitável para 1 ano inteiro.
* Cálculo de completude consistente (inclusive vs exclusivo).

---

##  Dicas Práticas

* `requests + zipfile + io.BytesIO` → processar direto em memória.
* `pandas.read_csv(..., sep=";", encoding="latin1", decimal=",", dtype=str)` → leitura segura.
* Normalização de colunas → `unidecode + str.lower() + trim`.
* `pd.to_datetime(..., dayfirst=True, errors="coerce")` → construção de datetime.
* Regex para código da estação: `INMET_.*_([A-Z]\d{3})_.*`.
* Exportar relatórios com `;`.
