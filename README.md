# Case Técnico Data Science — iFood

Solução para o desafio de otimização de distribuição de cupons e ofertas para clientes do iFood, utilizando dados de transações, ofertas e perfis de clientes.

## Objetivo

Desenvolver um modelo que auxilie na decisão de qual oferta enviar para cada cliente, maximizando a conversão **incremental** — ou seja, identificar clientes cuja compra foi de fato influenciada pela oferta, e não clientes que comprariam de qualquer forma.

## Estrutura do repositório

```
ifood-case/
├── data/
│   ├── raw/                    # offers.json, profile.json, transactions.json
│   └── processed/              # base_modelagem.parquet (dataset unificado)
├── notebooks/
│   ├── 1_data_processing.ipynb # Limpeza, parsing e criação do target
│   └── 2_modeling.ipynb        # Treino, avaliação e ajuste dos 3 modelos
├── presentation/                # Slides para stakeholders (5 páginas)
├── README.md
└── requirements.txt
```

## Como executar

1. Acesse o [Databricks Community Edition](https://community.cloud.databricks.com/)
2. Crie um catálogo (ex: `ifood`) com schema `default` e um Volume `raw`
3. Faça upload dos 3 arquivos JSON originais para o Volume
4. Execute `1_data_processing.ipynb` — gera `base_modelagem` (Parquet) no Volume
5. Execute `2_modeling.ipynb` — treina e avalia os modelos, salva os thresholds finais

Ambos os notebooks usam PySpark e rodam em compute **Serverless**.

## Premissas assumidas

Como o dicionário de dados fornecido no case não especifica todos os detalhes, as seguintes premissas foram adotadas e documentadas ao longo da análise:

- **`time_since_test_start`** está em **dias**, com granularidade de 6h (múltiplos de 0,25) — confirmado pelo valor máximo (~29,75) compatível com um teste de ~30 dias.
- **`duration`** (em `offers`) está na mesma unidade (dias), permitindo comparação direta com `time_since_test_start`.
- **`age = 118`** é um valor sentinela para perfil incompleto (não uma idade real), sempre acompanhado de `gender` e `credit_card_limit` nulos. Criada uma flag `perfil_incompleto` para sinalizar esses casos, em vez de excluí-los ou imputá-los sem marcação.
- **`gender = "O"`** é uma categoria real e intencional (perfil completo), distinta de `null` (perfil incompleto). Tratadas separadamente: `O` foi agrupado com `F`, e os `null` receberam categoria própria (`I`), evitando misturar "escolha ativa" com "dado ausente".

## Definição do evento-alvo (target)

O ponto central da solução é tratar a conversão como um problema de **incrementalidade**, não de classificação simples sobre `offer completed`. A sequência de eventos por oferta (`account_id` + `offer_id` + instância de envio) foi reconstruída da seguinte forma:

**BOGO e Discount** (ofertas com recompensa):
- Considera-se conversão **efetiva** quando `offer viewed` ocorre **antes** da transação que gera o `offer completed`, dentro da janela de `duration`.
- Se o `viewed` ocorre **depois** da transação (ou não ocorre), a compra é tratada como orgânica — o cliente compraria de qualquer forma, e a oferta é descartada como "não efetiva" mesmo tendo sido completada.

**Informational** (sem recompensa, sem evento `completed`):
- Considera-se conversão efetiva quando existe uma `transaction` **depois** do `viewed`, dentro da janela de `duration`.

Essa distinção foi o principal ponto de cuidado do projeto: tratar `offer completed` como sinônimo de "oferta funcionou" gera um modelo que aprende a prever quem já ia comprar, não quem foi de fato influenciado.

## Estratégia de modelagem

Testou-se um modelo único (Regressão Logística) para os 3 tipos de oferta, e a performance variou fortemente por tipo:

| Tipo | AUC-ROC (modelo único) |
|---|---|
| Discount | 0,76 |
| BOGO | 0,67 |
| Informational | 0,58 |

Essa diferença motivou a divisão em **3 modelos especializados**, um por tipo de oferta, com seleção de features e ajuste de threshold específicos para cada um.

### Resultado final por modelo (teste)

| Tipo | AUC-ROC | Recall (classe 1) | Threshold usado |
|---|---|---|---|
| Discount | 0,76 | 0,77 | 0,50 (padrão) |
| BOGO | 0,69 | 0,51 | 0,50 (padrão) |
| Informational | 0,62 | 0,90 | **0,28** (ajustado) |

O threshold do modelo de Informational foi reduzido de 0,50 para 0,28, priorizando Recall — dado que o custo de disparar esse tipo de oferta (sem desconto embutido) é menor que o custo de deixar de engajar um cliente que teria convertido.

## Principais descobertas

- **Perfil incompleto** reduz a probabilidade de conversão em todos os 3 tipos de oferta — abre uma oportunidade de negócio: incentivar o preenchimento do cadastro pode ter retorno multiplicado.
- **Canal social** é o mais relevante para BOGO e Discount, mas tem pouco efeito em ofertas Informational.
- **Duração da oferta** tem efeito oposto entre grupos: cupons mais longos ajudam BOGO/Discount, mas prejudicam Informational (perde urgência/relevância com o tempo).
- **BOGO é o tipo mais difícil de prever** com as features disponíveis — sugere que fatores fora do dataset (categoria do restaurante, tamanho do pedido) podem influenciar mais que o perfil do cliente.

## Limitações e próximos passos

- O modelo atual estima probabilidade de conversão condicional à oferta, não o **efeito causal incremental** de forma explícita. Um próximo passo natural é um modelo de **uplift (T-learner)**: treinar um modelo para quem recebeu a oferta e outro para quem não recebeu, usando a diferença de probabilidades como métrica de recomendação.
- O modelo de BOGO teria ganho relevante com features adicionais não presentes no dataset atual (categoria/ticket médio do pedido).
- O threshold de decisão foi otimizado apenas para o modelo de Informational; os modelos de BOGO e Discount seguem com o threshold padrão (0,50) e poderiam passar pelo mesmo processo de ajuste caso o negócio priorize Recall sobre Precision.