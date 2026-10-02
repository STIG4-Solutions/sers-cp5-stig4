# Checkpoint 02 — APIs, energias renováveis e aprendizado de máquina

- Gabriel Fagundes - RM: 569074
- Gabriel Freitas - RM: 572943
- Giovanni Merlotti - RM: 573721
- Glauco Kelly - RM: 572840
- Sergio Augusto Amaral - RM: 570184
- Thiago Renatino - RM: 569073

Ciência da Computação — 1º ano, 2º semestre

## Objetivo

Consumir duas APIs públicas de dados de energia/clima, gerar dois conjuntos de dados e resolver duas tarefas independentes de aprendizado de máquina, comparando **três algoritmos em cada uma**:

1. **Classificação:** prever a fonte de um empreendimento de geração (*Solar*, *Eólica* ou *Hidráulica*) usando apenas potência outorgada e localização.
2. **Regressão:** estimar a radiação solar global horizontal (W/m²) em Petrolina (PE) a partir de variáveis meteorológicas e da hora do dia.

## Origem e período dos dados

| Arquivo | Fonte (API pública, sem token) | Conteúdo | Período / recorte |
|---|---|---|---|
| `aneel_classificacao_orange.csv` | [ANEEL — SIGA](https://dadosabertos.aneel.gov.br/dataset/siga-sistema-de-informacoes-de-geracao-da-aneel) (CKAN `datastore_search`, recurso `11ec447d-698d-4ab8-977f-b424d5deee6a`) | 3.876 empreendimentos: `potencia_kw`, `latitude`, `longitude`, `fonte` | Cadastro vigente na consulta; até 1.200 registros por sigla (UFV, EOL, UHE, PCH, CGH) |
| `meteo_regressao_orange.csv` | [Open-Meteo — Historical Weather API](https://open-meteo.com/en/docs/historical-weather-api) | 1.001 horas: temperatura, umidade, nuvens, vento, hora e radiação | 01/04/2025 a 30/06/2025, 7h–17h, fuso `America/Recife`, coord. −9,39 / −40,50 |

Observações: a potência da ANEEL é **outorgada** (não é energia gerada) e a quantidade por classe vem do limite da consulta, não da matriz energética brasileira. Os dados do Open-Meteo são **estimados por modelo/reanálise**, não medidos num painel.

## Estrutura do repositório

```
├── README.md
├── CP02_Energias_Renovaveis_ML.ipynb   # notebook completo (APIs + 6 modelos + análises)
├── aneel_classificacao_orange.csv      # gerado pelo notebook (Tarefa 1)
└── meteo_regressao_orange.csv          # gerado pelo notebook (Tarefa 2) 
```

## Como executar

**Google Colab:** abra o notebook pelo GitHub (`Arquivo > Abrir notebook > GitHub`) e use `Ambiente de execução > Executar tudo`.

**Local:**

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install pandas numpy matplotlib scikit-learn jupyter
jupyter notebook CP02_Energias_Renovaveis_ML.ipynb
```

Rode as células **em ordem** (`Run All`). As primeiras células de cada tarefa consultam a API e regravam o CSV. Se a API estiver fora do ar ou bloqueada, o notebook carrega automaticamente a cópia publicada no repositório da disciplina e continua. Não é necessário token. Todas as divisões e modelos usam `random_state=42`.

## Resultados

### Tarefa 1 — Classificação (ANEEL)

Pré-processamento: removi 47 registros com coordenada (0, 0) (erro de cadastro) e usei `log10(1 + potência)` por causa da escala muito assimétrica. KNN e Regressão Logística têm `StandardScaler` dentro de um pipeline (ajustado só no treino).
Avaliação: **mesma divisão para os três**: 80/20 **estratificada**, `random_state=42` (3.063 treino / 766 teste). Precision, Recall e F1 com média **macro**.

| Modelo | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) |
|---|---|---|---|---|
| **Random Forest** (200 árvores) | **0,977** | **0,977** | **0,976** | **0,977** |
| KNN (k = 7, padronizado) | 0,967 | 0,968 | 0,966 | 0,967 |
| Regressão Logística (padronizada) | 0,832 | 0,847 | 0,828 | 0,825 |

**Conclusões:** o Random Forest foi o melhor, com o KNN logo atrás; a Regressão Logística perde porque só traça fronteiras lineares, e as fontes ocupam regiões irregulares do mapa. A classe mais confundida é a **Solar** (com Eólica no Nordeste e com Hidráulica no Sul/Sudeste). Porém o resultado está **inflado**: 61% dos solares da amostra estão concentrados em duas células de 1°×1° do mapa, 64% são microgeração de ~1 kW e 23% dos pontos do teste têm coordenada quase idêntica no treino. Potência e localização não descrevem o recurso físico (rio, vento, sol), então o modelo não deve ser usado como classificador real sem uma amostra representativa e uma validação por região.

### Tarefa 2 — Regressão (Open-Meteo)

Entradas: `temperatura_c`, `umidade_pct`, `nuvens_pct`, `vento_kmh`, `hora`. Alvo: `radiacao_w_m2`.
Avaliação: **mesma divisão temporal para os três**: primeiras 80% das horas para treino (800 h, 01/04 a 12/06) e 20% finais para teste (201 h, 12/06 a 30/06), sem embaralhar.

| Modelo | MAE (W/m²) | MSE ((W/m²)²) | R² |
|---|---|---|---|
| **Random Forest** (300 árvores, `min_samples_leaf=3`) | **69,3** | **7.860** | **0,832** |
| Árvore de Decisão (`max_depth=6`) | 90,9 | 15.127 | 0,678 |
| Regressão Linear (padronizada) | 145,2 | 30.034 | 0,360 |

**Conclusões:** o Random Forest é o melhor. A **hora** é a variável mais importante, mas sua relação com a radiação tem formato de sino (pico ao meio-dia), o que os modelos de árvore capturam e a regressão linear não. O teste (fim de junho, inverno) tem radiação média menor que o treino, o que torna a avaliação temporal mais exigente e realista. Estimar radiação **não é prever geração elétrica**: W/m² horizontal ≠ kWh; a geração depende de área, inclinação e orientação dos painéis, eficiência (que cai com o calor), perdas do inversor, sujeira e sombreamento.

## Referências

- ANEEL — SIGA: https://dadosabertos.aneel.gov.br/dataset/siga-sistema-de-informacoes-de-geracao-da-aneel
- Open-Meteo — Historical Weather API: https://open-meteo.com/en/docs/historical-weather-api
- scikit-learn: https://scikit-learn.org/stable/
