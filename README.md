# Electricity | Classificação do preço da eletricidade em NSW

Projeto da disciplina Introdução ao Aprendizado de Máquina, Análise e Desenvolvimento de Sistemas, IFSP Campinas, 2º semestre de 2026.
Professor: Everton Meyer da Silva.

**Integrantes:** Estevan Hernandes (CP3031187), Marcus Vinicius (CP3031721) e Pedro Henrique (CP3029352).

## Objetivo

Classificar cada período de 30 minutos do mercado de eletricidade de New South Wales (NSW) como `down` (preço abaixo da média das últimas 24 horas) ou `up` (acima). A classe de interesse é `down`.

## Base de dados

- **Fonte:** OpenML, [Electricity (dataset 151)](https://www.openml.org/d/151). Harries, M. (1999), University of New South Wales.
- **Conteúdo:** mercado de eletricidade australiano, de maio de 1996 a dezembro de 1998, com dados de NSW e de Victoria. Cada linha é um período de 30 minutos, e todos os valores estão normalizados entre 0 e 1.
- **Divisão (fornecida pelo professor), respeitando a ordem do tempo:**

| Conjunto | Períodos | Trecho                            |
| -------- | -------: | --------------------------------- |
| Treino   |   36.240 | parte inicial da sequência        |
| Teste    |    9.072 | parte final (cerca de 27 semanas) |

## Estrutura do repositório

```
├── data/
│   ├── electricity_train.csv
│   └── electricity_test.csv
├── notebooks/
│   └── electricity.ipynb   # todo o projeto: dados, EDA, modelos, teste e bônus
├── requirements.txt
└── README.md
```

## Como executar

Requer Python 3.13 (o projeto foi feito com a versão 3.13.7).

1. Criar e ativar o ambiente virtual, na pasta do projeto (PowerShell):

   ```powershell
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   ```

2. Instalar as dependências:

   ```powershell
   pip install -r requirements.txt
   ```

3. Abrir `notebooks/electricity.ipynb` no VS Code (extensões Python e Jupyter), escolher o kernel `.venv` e usar **Run All**.

O notebook lê os dados com o caminho `../data/`, então ele deve ser executado a partir da pasta `notebooks/`, que é o padrão do VS Code. Para usar o Jupyter no navegador, instale antes com `pip install notebook`.

## Etapas do notebook

| Etapa                        | Conteúdo                                                                                                                                             |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Base e preparação         | Origem dos dados, checagens de qualidade, faltantes disfarçados em Victoria, outliers e inconsistências da data                                      |
| 2. Análise exploratória      | Oito perguntas sobre o preço, o horário, os dias da semana, as correlações, a persistência da classe, a origem do alvo e a mudança ao longo do tempo |
| 3. Preparação das variáveis  | 15 variáveis: as 5 numéricas, o indicador `vic_sem_dado`, o horário em seno e cosseno e o dia da semana em one-hot                                   |
| 4. Comparação dos modelos    | Validação cruzada temporal (`TimeSeriesSplit`, 5 rodadas) com Baseline, Regressão Logística, KNN, Random Forest, SVM e MLP                           |
| 5. Ajuste dos modelos        | `GridSearchCV` com validação temporal e escolha do limiar de decisão                                                                                 |
| 6. Avaliação no teste        | Resultado no conjunto de teste e diagnóstico da queda de desempenho                                                                                  |
| 7. Bônus: aprendizado online | Regressão Logística com a biblioteca river, aprendendo um período por vez                                                                            |
| 8. Conclusão e limitações    | Síntese dos resultados e limites do trabalho                                                                                                         |

## Principais decisões

- **Validação temporal:** os dados são uma sequência no tempo, então a validação sempre treina no passado e avalia no período seguinte (`TimeSeriesSplit`), nunca com embaralhamento.
- **Faltantes disfarçados em Victoria:** em 48,1% do treino, as colunas de Victoria têm valores fixos no lugar dos dados que faltavam. As linhas foram mantidas e marcadas com o indicador `vic_sem_dado`. Um experimento mostrou que treinar só com o período completo não melhorava os modelos.
- **Alvo determinístico:** a EDA mostrou que o alvo é exatamente "preço abaixo da média dos 48 períodos anteriores". Variáveis que refariam essa conta (preços anteriores, média móvel, classe anterior) foram deixadas de fora, porque com elas qualquer modelo acertaria 100% e a comparação perderia o sentido.
- **Métricas:** F1 e PR-AUC da classe `down`, além da acurácia. As classes são um pouco desbalanceadas (cerca de 58% `down` no treino), e a PR-AUC avalia se o modelo ordena bem os períodos sem depender de um limiar.
- **Escalonamento dentro do `Pipeline`:** a padronização é calculada só com os dados de treino de cada rodada, sem vazamento.

## Resultados

### Validação cruzada (Etapa 4, configurações iniciais)

| Modelo                  | F1 (`down`) |    PR-AUC |  Acurácia |
| ----------------------- | ----------: | --------: | --------: |
| **Regressão Logística** |   **0,811** | **0,865** | **0,740** |
| MLP                     |       0,804 |     0,855 |     0,737 |
| Random Forest           |       0,777 |     0,826 |     0,708 |
| SVM (RBF)               |       0,795 |     0,802 |     0,729 |
| KNN                     |       0,791 |     0,793 |     0,723 |
| Baseline                |       0,732 |     0,578 |     0,578 |

O SVM foi de longe o mais lento para treinar, e a Regressão Logística e o KNN foram os mais rápidos. Os tempos exatos variam conforme a máquina.

### Ajuste de hiperparâmetros (Etapa 5)

| Modelo                  | Melhores parâmetros                       | PR-AUC antes | PR-AUC depois |
| ----------------------- | ----------------------------------------- | -----------: | ------------: |
| MLP                     | `alpha=10`, `hidden_layer_sizes=(32, 16)` |        0,855 |         0,869 |
| **Regressão Logística** | `C=0.01`                                  |        0,865 |         0,868 |
| Random Forest           | `max_depth=10`, `min_samples_leaf=20`     |        0,826 |         0,863 |
| KNN                     | `n_neighbors=201`                         |        0,793 |         0,831 |

**Modelo escolhido: Regressão Logística** (`C=0.01`, limiar de decisão 0,60). Ela empatou com a MLP e é mais simples, mais rápida e interpretável. Em todos os modelos, a melhor configuração foi a mais contida, porque os modelos complexos tendiam a decorar o treino.

### Teste (Etapa 6)

| Modelo                                       | F1 (`down`) | PR-AUC | Acurácia |
| -------------------------------------------- | ----------: | -----: | -------: |
| Regressão Logística, limiar 0,60 (escolhido) |       0,574 |  0,889 |    0,661 |
| Regressão Logística, limiar 0,5              |       0,677 |  0,889 |    0,716 |
| Baseline                                     |       0,709 |  0,549 |    0,549 |

O modelo continuou ordenando bem os períodos (PR-AUC alta), mas acertou menos do que na validação. A causa foi a subida do nível de preço no teste. O preço mediano foi de 0,031 no fim do treino para 0,059 no teste. O modelo, treinado uma vez só, continuou achando caros preços que tinham virado normais, e previu `down` em 24,7% dos períodos, quando o real era 54,9%.

### Bônus: aprendizado online com river (Etapa 7)

Aqui, o mesmo algoritmo (Regressão Logística) aprende um período por vez. Ela prevê cada período e só depois aprende a classe real dele.

| Modelo               | F1 (`down`) | Precisão |    Recall |    PR-AUC |  Acurácia |
| -------------------- | ----------: | -------: | --------: | --------: | --------: |
| Fixa (limiar 0,60)   |       0,574 |    0,924 |     0,417 |     0,889 |     0,661 |
| **Online com river** |   **0,873** |    0,856 | **0,892** | **0,937** | **0,858** |

O modelo online acompanhou a mudança de preço e acertou mais que o modelo fixo em 25 das 27 semanas do teste. As configurações do river não foram ajustadas, então o teste não foi usado para nenhuma escolha.

## Conclusão

Neste problema, a mudança do mercado ao longo do tempo pesou mais que a escolha do algoritmo. Depois do ajuste, os quatro modelos ficaram entre 0,831 e 0,869 de PR-AUC na validação. Já no teste, a Regressão Logística foi de F1 0,574 (treinada uma vez só) para 0,873 (atualizada a cada período).

A recomendação é uma Regressão Logística atualizada continuamente, que é simples, rápida, interpretável e acompanha as mudanças de preço.

## Limitações

- O modelo classifica o período atual usando o preço dele mesmo. Ele não prevê o próximo período.
- O alvo é uma regra exata, e a informação que a refaria foi deixada de fora de propósito. Os resultados valem para essa versão do problema.
- O limiar escolhido na validação não se manteve no teste, porque o mercado mudou depois do treino.
- O modelo online depende de receber a classe real logo após cada período.
- Os dados estão normalizados, então não é possível interpretar valores em unidades reais.
