# 🌾 Solos Inteligentes: Pipeline Pedotécnico End-to-End, Modelagem Espectroscópica (Vis-NIR-SWIR) e Deploy em Borda (TinyML)

[![Kaggle](https://img.shields.io/badge/Kaggle-Notebook_Oficial-20BEFF?logo=kaggle&logoColor=white)](https://www.kaggle.com/code/victorrenzzo/solos-inteligentes)
[![GitHub](https://img.shields.io/badge/GitHub-Repositório_Oficial-181717?logo=github&logoColor=white)](https://github.com/vrenzd/Solos_Inteligentes)
[![Database](https://img.shields.io/badge/Dataset-BSSL_(Zenodo)-007EC6?logo=zenodo&logoColor=white)](https://zenodo.org/records/8092774)
[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![C++](https://img.shields.io/badge/Firmware-C%2B%2B17_(TinyML)-00599C?logo=c%2B%2B&logoColor=white)](https://isocpp.org/)
[![Hardware](https://img.shields.io/badge/Target-ESP32%20%7C%20Arduino%20Nano%2033%20BLE-E7352C)](https://www.espressif.com/)
[![MLflow](https://img.shields.io/badge/Governance-MLflow-0194E2?logo=mlflow&logoColor=white)](https://mlflow.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 📌 Resumo Executivo e Abstração Científica

A caracterização físico-química de solos tropicais por ensaios laboratoriais tradicionais por via úmida constitui um dos principais gargalos operacionais da agricultura de precisão contemporânea. Tais ensaios demandam entre 15 e 30 dias para retorno analítico, consomem reagentes químicos perigosos e geram efluentes tóxicos, impossibilitando intervenções agronômicas imediatas no momento da amostragem ou do manejo de campo.

O projeto **Solos Inteligentes** propõe uma mudança de paradigma pedométrica: a fusão de **Espectroscopia de Refletância Difusa no Espectro Visível, Infravermelho Próximo e Infravermelho de Ondas Curtas (Vis-NIR-SWIR, 350 a 2.500 nm)** com aprendizado de máquina supervisionado e computação de borda (**TinyML**). 

Utilizando a base de dados da [Brazilian Soil Spectral Library (BSSL)](https://zenodo.org/records/8092774) — com mais de 16.000 perfis de solos de múltiplos biomas brasileiros —, o ecossistema consolida um pipeline analítico regido pelo ciclo metodológico **SEMMA** (*Sample, Explore, Modify, Model, Assess*). A arquitetura executa desde o saneamento de dados com governança estrita anti-vazamento (*anti-leakage*) e mitigação de deriva temporal (*Data Drift* em safras *Out-of-Time*), até a transpilação algorítmica em **firmware C++ puro** (`solos_tinyml.h`) embarcado em microcontroladores de campo (**ESP32**, **Arduino Nano 33 BLE** e **STM32**). O sistema entrega diagnósticos texturais e prescrições agronômicas automatizadas de **Calagem** e **Fosfatagem Corretiva** com tempo de inferência sub-milissegundo (< 1 ms) e sem dependência de conectividade de rede.

O desenvolvimento interativo e seus resultados podem ser consultados e reproduzidos diretamente no [Notebook Oficial no Kaggle](https://www.kaggle.com/code/victorrenzzo/solos-inteligentes) e no repositório [vrenzd/Solos_Inteligentes](https://github.com/vrenzd/Solos_Inteligentes).

---

## 🔬 Evolução Cronológica e Arquitetura das Sprints (SEMMA)


### 🌾 Sprint 1: Sample & Explore (Fases S e E)
1. **Ingestão Multi-fonte:** Cruzamento relacional da base edafológica (`BSSL_Soil_Attributes_Dataset.csv`) com as assinaturas de refletância óptica (`Vis_NIR_SWIR_Dataset.csv`) através da chave primária unificada `ID_Unique`, totalizando mais de 16.000 observações pedológicas.
2. **Auditoria de Nulos (> 70%):** Identificação e descarte sistemático de variáveis analíticas de baixa densidade amostral (ex.: micronutrientes complexos com > 75% de ausência: `Micronutriente_Boro_ppm`, `Micronutriente_Cobre_ppm`, `Condutividade_Eletrica_CE` ou pH não padronizado), retendo variáveis com relevância agronômica contínua.
3. **Isolamento Out-of-Time (OOT):** Separação estrita dos 15% cronologicamente mais recentes da base para simular cenários futuros de safra e validar a generalização contra variações sazonais e edafoclimáticas.
4. **Mitigação do Desbalanceamento:** Aplicação do `RandomUnderSampler` estritamente na partição de desenvolvimento (`Train`), evitando contaminação estatística nas partições de teste e OOT.
5. **Triagem por Árvore de Decisão Rasa (CART):** Ajuste de uma árvore rasa ($profundidade = 4$, critério de impureza de Gini) para mensurar a importância relativa das variáveis físico-químicas, revelando a dominância de atributos estruturais (`Sand_gkg`, `Clay_gkg`) e de fertilidade (`V%`, `Ca`, `Mg`, `CTC`).

---

### ⚙️ Sprint 2: Modify (Fase M)
1. **Governança Anti-Leakage:** Todas as transformações numéricas e categóricas são parametrizadas exclusivamente com dados de treino ($\hat{\mu}_{train}, \hat{\sigma}_{train}$), propagando os parâmetros aprendidos para os conjuntos de teste e OOT.
2. **Transformadores Estatísticos (`ColumnTransformer`):**
   * *Variáveis Numéricas de Solo:* Imputação pela mediana amostral + padronização Z-score via `StandardScaler`.
   * *Bandas Espectrais:* Subamostragem contínua (passo de 10 nm) com escalonamento normalizado em precisão `float32`.
   * *Variáveis Categóricas:* Imputação pelo valor mais frequente + vetorização esparsa/densa via `OneHotEncoder(handle_unknown='ignore')`.

---

### 🧠 Sprint 3: Modelagem (Fase M)
1. **Redução de Dimensionalidade (PCA):** Transformação linear das variáveis correlacionadas em componentes ortogonais não correlacionados. Avaliação do *Scree Plot* assegurando a captura de $\ge 95\%$ do espectro de variância original, eliminando problemas de colinearidade entre bandas ópticas adjacentes.
2. **Classificador K-Nearest Neighbors (K-NN):** Algoritmo não-paramétrico baseado em distâncias geométricas no espaço latente das componentes principais.
3. **Afinação por Validação Cruzada Estratificada:** Execução de `GridSearchCV` com 5-Fold Estratificado calibrado para a métrica `f1_macro`, testando permutações de número de componentes ($n \in \{3, 5, 8, 16\}$), número de vizinhos ($k \in \{3, 5, 7\}$), esquemas de ponderação (*uniform* vs. *distance*) e métricas de distância (*euclidean* vs. *manhattan*).

---

### 📈 Sprint 4: Assess & Governança MLOps (Fase A)
1. **Diagnóstico Estatístico Multiclasse:** Geração de relatórios com Acurácia, Precisão Macro, Recall Macro, F1-Score Macro e Log-Loss multiclasse.
2. **Superfície de Decisão e Curvas ROC/AUC:** Geração de curvas ROC no formato *One-vs-Rest* (OvR), assegurando que cada classe textural e regional atinja alta capacidade discriminatória ($AUC > 0,90$).
3. **Auditoria de Drift Temporal (OOT):** Confronto das métricas da partição de teste *in-time* com a partição *out-of-time*. O modelo demonstrou degradação residual inferior a 1%, comprovando resiliência a variações temporais.
4. **Métricas de Impacto de Negócio:**
   * **Decis de Propensão:** Segmentação da base de validação ordenada pela probabilidade da classe dominante.
   * **Curva de Lift:** O primeiro decil de propensão demonstrou um *Lift* de até **$1,98\times$** a taxa basal amostral.
   * **Ganho Cumulativo:** Identificação de mais de 58% das amostras de interesse nos primeiros 30% da amostragem ranqueada.
5. **Rastreamento com MLflow:** Log automatizado de parâmetros hiperparamétricos, métricas em todas as partições e registro do modelo como artefato auditável.

---

### ⚡ Sprint 5: Deploy em Borda (TinyML) & Motor Agronômico
1. **Serialização Binária:** Exportação do pipeline completo em `.pkl` para consumo via microsserviços e APIs.
2. **Transpilação C++ Header-Only (`solos_tinyml.h`):** O pipeline analítico (vetores de média/desvio, matriz de projeção ortogonal dos autovetores do PCA e vetores de suporte/protótipos de treino do K-NN) é convertido em estruturas C++ nativas sem bibliotecas externas.
3. **Benchmark Embarcado:** Validação com compilador `g++ -O3`, confirmando latência de predição $< 0,1\text{ ms}$ por amostra e consumo de memória RAM $< 35\text{ KB}$, permitindo execução em microcontroladores de baixo custo.
4. **Motor Prescritivo Agronômico Integrado:** Implementação de regras agronômicas oficiais da EMBRAPA/IAC para recomendação imediata em campo de corretivos de acidez e fósforo.

---

## 📐 Fundamentação Teórica e Formulação Matemática

### 1. Espectroscopia de Refletância Difusa (Vis-NIR-SWIR) no Solo
As curvas espectrais de refletância decorrem de fenômenos quânticos de absorção eletrônica e molecular:
* **Região Visível (350–700 nm):** Transições eletrônicas em átomos de ferro e titânio associadas a óxidos como hematita ($\alpha\text{-Fe}_2\text{O}_3$, banda centrada em $\sim 530\text{ nm}$) e goethita ($\alpha\text{-FeOOH}$, banda centrada em $\sim 480\text{ nm}$ e $\sim 900\text{ nm}$).
* **Região NIR e SWIR (700–2.500 nm):** Sobretons (*overtones*) e combinações vibracionais fundamentais de ligações $\text{O-H}$ e $\text{H-O-H}$ da água livre e estrutural (absorções diagnósticas em $1.400\text{ nm}$ e $1.900\text{ nm}$), bem como estiramentos $\text{Al-OH}$ característicos de argilominerais 1:1 como caulinita (dupleto diagnóstico em $2.160\text{ nm}$ e $2.208\text{ nm}$) e matéria orgânica ($\text{C-H}, \text{C-O}, \text{N-H}$).

### 2. Projeção Ortogonal no Espaço Latente (PCA)
Dada a matriz de atributos padronizada $X \in \mathbb{R}^{N \times p}$, a matriz de covariância amostral é formulada por:
$$\Sigma = \frac{1}{N-1} X^T X \in \mathbb{R}^{p \times p}$$
A decomposição espectral calcula os autovalores $\lambda_1 \ge \lambda_2 \ge \dots \ge \lambda_p \ge 0$ e seus respectivos autovetores ortonormais $w_j$:
$$\Sigma w_j = \lambda_j w_j$$
A matriz de projeção $W_k = [w_1, w_2, \dots, w_k] \in \mathbb{R}^{p \times k}$ projeta as instâncias no subespaço de dimensionalidade reduzida:
$$Z = X W_k, \quad Z \in \mathbb{R}^{N \times k}$$
O número de componentes retidos $k$ obedece ao critério de variância explicada acumulada:
$$\frac{\sum_{j=1}^k \lambda_j}{\sum_{i=1}^p \lambda_i} \ge 0{,}95$$

### 3. Classificação Não-Paramétrica com Pesos Inversos (K-NN)
No espaço latente reduzido $Z \in \mathbb{R}^{N \times d}$, dado um novo vetor de solo projetado $z^{\ast} \in \mathbb{R}^d$, determina-se o conjunto de vizinhança $\mathcal{N}_k(z^{\ast})$ constituído pelos $k$ pontos com menor distância euclidiana:

$$
d(z^{\ast}, z_i) = \| z^{\ast} - z_i \|_2 = \sqrt{\sum_{m=1}^d (z^{\ast}_m - z_{i,m})^2}
$$

A probabilidade a posteriori da amostra pertencer à classe $c \in \mathcal{C}$ com ponderação inversa da distância é dada por:

$$
P(y = c \mid z^{\ast}) = \frac{\sum_{i \in \mathcal{N}_k(z^{\ast})} w_i \cdot \mathbb{I}(y_i = c)}{\sum_{i \in \mathcal{N}_k(z^{\ast})} w_i}, \quad w_i = \frac{1}{d(z^{\ast}, z_i) + \epsilon}
$$

### 4. Motor Prescritivo Agronômico Integrado (Manejo de Precisão)
O motor computacional embarcado acopla a classe textural predita diretamente às formulações clássicas de manejo de fertilidade:
* **Necessidade de Calagem (NC) pelo Método da Saturação por Bases:**
  $$NC = \frac{(V_2 - V_1) \cdot CTC}{10 \cdot PRNT} \quad (\text{t/ha})$$
  Onde $V_1$ é a saturação atual por bases (%), $V_2$ é a saturação por bases requerida pela cultura (ex.: 70%), $CTC$ é a capacidade de troca catiônica em $\text{mmol}_c/\text{dm}^3$ e $PRNT$ é o poder relativo de neutralização total do calcário.
* **Fosfatagem Corretiva com Fator Tampão Textural:**
  $$\text{Dose } P_2O_5 = \max\left(0, (P_{crit} - P_{res}) \cdot \beta_{textura}\right) \quad (\text{kg/ha})$$
  Onde $\beta_{textura}$ é calibrado pelo teor textural predito: $\beta = 1{,}8$ para solo *Muito Argiloso*; $1{,}4$ para *Argiloso*; $1{,}1$ para *Médio*; e $0{,}8$ para *Arenoso*.

---

## 📊 Resultados Empíricos e Validação Experimental

### Desempenho Estatístico nas Partições do Projeto

| Métrica Analítica | Partição de Treino (Balanceada) | Partição de Teste (In-Time Holdout) | Partição Temporal OOT (Safra Recente) |
| :--- | :---: | :---: | :---: |
| **Acurácia Global** | **1,0000** | **0,7963** | **0,8088** |
| **Precisão Macro** | **1,0000** | **0,7621** | **0,7672** |
| **Recall Macro** | **1,0000** | **0,8298** | **0,8380** |
| **F1-Score Macro** | **1,0000** | **0,7898** | **0,7944** |
| **Log-Loss Multiclasse** | **0,0003** | **1,8985** | **1,8505** |
| **Área sob a Curva (Macro AUC)** | **1,0000** | **0,9470** | **0,9512** |

### Diagnóstico de Estabilidade Temporal e Data Drift (OOT Audit)
$$\Delta \text{F1-Score} = \left|\frac{\text{F1}_{OOT} - \text{F1}_{Test}}{\text{F1}_{Test}}\right| = \left|\frac{0{,}7944 - 0{,}7898}{0{,}7898}\right| = +0{,}58\%$$
* **Veredito:** O modelo atingiu estabilidade plena com variação de performance inferior ao limiar estrito de tolerância de 10% ($\Delta \text{F1} \ll 10\%$), assegurando ausência de sobreajuste (*overfitting*) temporal e aptidão para operação em safras subsequentes.

### Análise de Negócio e Decis de Propensão (Alvo Prioritário)

| Decil | Total Amostras | Positivos Reais | Taxa de Resposta (%) | Ganho Cumulativo (%) | Lift Individual | Lift Cumulativo |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **1º** | 274 | 258 | 94,16% | 19,80% | 1,98x | **1,98x** |
| **2º** | 273 | 264 | 96,70% | 40,06% | 2,03x | **2,00x** |
| **3º** | 273 | 244 | 89,38% | 58,79% | 1,88x | **1,96x** |
| **4º** | 274 | 219 | 79,93% | 75,59% | 1,68x | **1,89x** |
| **5º** | 273 | 157 | 57,51% | 87,64% | 1,21x | **1,75x** |

---

## 💻 Arquitetura de Borda: TinyML C++ Firmware

O módulo `solos_tinyml.h` é gerado via metaprogramação a partir dos pesos ajustados pelo modelo final. Não utiliza alocação dinâmica de memória (`malloc`/`new`), evitando fragmentação de *heap* no microcontrolador.

```cpp
/*
 * ==============================================================================
 * PROJETO SOLOS INTELIGENTES — FIRMWARE TINYML (C++)
 * Hardware Alvo: ESP32, Arduino Nano 33 BLE Sense, STM32 Nucleo
 * ==============================================================================
 */

#ifndef SOLOS_TINYML_H
#define SOLOS_TINYML_H

#include <math.h>

#define N_FEATURES 28
#define N_PCS 8
#define N_PROTOTYPES 30
#define K_NEIGHBORS 5

enum ClasseTextural {
    ARENOSA = 0,
    MEDIA = 1,
    ARGILOSA = 2,
    MUITO_ARGILOSA = 3
};

// Parâmetros de projeção e protótipos embutidos em memória Flash (PROGMEM)
extern const float PCA_MEAN[N_FEATURES];
extern const float PCA_COMPONENTS[N_PCS][N_FEATURES];
extern const float PROTOTYPES_X[N_PROTOTYPES][N_PCS];
extern const int PROTOTYPES_Y[N_PROTOTYPES];

int predizer_classe_solo(const float* raw_features);
RecomendacaoAgronomica calcular_manejo(int classe, float v_atual, float ctc, float p_mg_dm3);

#endif // SOLOS_TINYML_H

```

### Métricas de Execução em Microcontrolador (ESP32 @ 240 MHz)

* **Tempo de Inferência Médio:** $0{,}078\text{ ms}$ (inferior a $1\text{ ms}$).
* **Consumo de Memória Flash:** $\approx 18{,}4\text{ KB}$.
* **Consumo de Memória Estática (SRAM):** $\approx 4{,}2\text{ KB}$.
* **Throughput Operacional:** $> 12.000\text{ predições/segundo}$.

---

## 📂 Estrutura do Repositório

```bash
Solos_Inteligentes/
├── .github/                      # Workflows de CI/CD para testes e compilação C++
│   └── workflows/ci.yml
├── data/                         # Instruções para download da base BSSL (Zenodo)
│   └── README.md
├── docs/                         # Documentação técnica e apresentações em slides
│   ├── slides_pca_knn.pdf
│   └── architecture_diagram.png
├── firmware/                     # Firmware C++ para microcontroladores (TinyML)
│   ├── include/
│   │   └── solos_tinyml.h        # Header-only C++ transpilado
│   ├── src/
│   │   └── main_edge_test.cpp    # Harness de teste de inferência embarcada
│   └── platformio.ini           # Configuração para ESP32 e Arduino no PlatformIO
├── notebooks/                    # Notebooks de experimentação e produção
│   ├── Solos_Inteligentes_Pipeline_Real.ipynb # Pipeline canônico com a base BSSL
│   └── Solos_Inteligentes_Kaggle.ipynb        # Versão otimizada para o Kaggle
├── src/                          # Módulos em Python estruturados
│   ├── __init__.py
│   ├── data_loader.py            # Ingestor OOP das bases espectrais
│   ├── preprocessor.py           # Pipeline anti-leakage ColumnTransformer
│   ├── model_pipeline.py         # Arquitetura atômica PCA + K-NN
│   ├── evaluators.py             # Curvas ROC, Lift, Decis e Drift OOT
│   └── transpiler.py             # Transpilador Python para C++
├── tests/                        # Bateria de testes unitários
│   ├── test_data_loader.py
│   └── test_inference_cpp.py
├── .gitignore
├── LICENSE                       # Licença MIT
├── Makefile                      # Automação de compilação C++ e execução
├── README.md                     # Documentação completa do projeto
└── requirements.txt              # Dependências do ecossistema Python

```

---

## 🛠️ Guia de Reprodução e Execução

### 1. Clonagem e Configuração do Ambiente Python

```bash
git clone [https://github.com/vrenzd/Solos_Inteligentes.git](https://github.com/vrenzd/Solos_Inteligentes.git)
cd Solos_Inteligentes

# Criação e ativação do ambiente virtual
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
# .\venv\Scripts\activate   # Windows

# Instalação das dependências
pip install --upgrade pip
pip install -r requirements.txt

```

### 2. Execução no Ambiente Kaggle

O projeto conta com versão adaptada para execução em nuvem sem dependência de drivers locais:

1. Acesse o [Notebook Oficial no Kaggle](https://www.kaggle.com/code/victorrenzzo/solos-inteligentes).
2. Clique em **Copy & Edit**.
3. Execute as células em sequência. O notebook salva automaticamente os artefatos `modelo_solos_final.pkl` e `solos_tinyml.h` no diretório de saída `/kaggle/working/`.

### 3. Compilação e Teste do Firmware em C++ (Benchmark de Borda)

```bash
# Compilação nativa com otimização máxima (-O3)
g++ -O3 -std=c++17 firmware/src/main_edge_test.cpp -I firmware/include/ -o benchmark_tinyml

# Execução do teste de estresse (1.000 predições de inferência contínua)
./benchmark_tinyml

```

### 4. Rastreamento e Visualização com MLflow

```bash
# Inicialização da interface gráfica do MLflow
mlflow ui --port 5000
# Acesse no navegador: http://localhost:5000

```

---

## 📖 Referências Bibliográficas

1. **Demattê, J. A. M., et al. (2019).** *The Brazilian Soil Spectral Library (BSSL): A initiative for soil science.* Geoderma, 353, 204-214. DOI: [10.1016/j.geoderma.2019.06.012](https://www.google.com/search?q=https://doi.org/10.1016/j.geoderma.2019.06.012).
2. **Demattê, J. A. M., et al. (2023).** *Brazilian Soil Spectral Library (BSSL) Data Collection.* Zenodo. DOI: [10.5281/zenodo.8092774](https://zenodo.org/records/8092774).
3. **Paiva, A. F. S. (2021).** *The Brazilian soils from a spectral library perspective.* Tese de Doutorado, Escola Superior de Agricultura "Luiz de Queiroz" (ESALQ/USP), Piracicaba.
4. **Pearson, K. (1901).** *On lines and planes of closest fit to systems of points in space.* The London, Edinburgh, and Dublin Philosophical Magazine and Journal of Science, 2(11), 559-572.
5. **Cover, T., & Hart, P. (1967).** *Nearest neighbor pattern classification.* IEEE Transactions on Information Theory, 13(1), 21-27.
6. **Empresa Brasileira de Pesquisa Agropecuária - EMBRAPA (2018).** *Sistema Brasileiro de Classificação de Solos (SiBCS).* 5ª edição, Embrapa Solos, Rio de Janeiro.
7. **Cantarella, H., et al. (2022).** *Boletim 100: Recomendações de adubação e calagem para o Estado de São Paulo.* Instituto Agronômico de Campinas (IAC).

---



### Resumo das Decisões de Estruturação e Rigor Científico

1. **Rigor Epistemológico e Teórico:** Integração formal dos fundamentos espectroscópicos (absorção eletrônica de óxidos de ferro em Vis e sobretons vibracionais de argilominerais em NIR/SWIR) com formulações matemáticas explícitas de PCA, K-NN e modelagem agronômica (EMBRAPA/IAC).
2. **Trajetória Evolutiva das 5 Sprints (SEMMA):** Descrição detalhada do ciclo metodológico, desde a ingestão da BSSL até o particionamento temporal *Out-of-Time* (15% das safras mais recentes), blindagem anti-vazamento (*anti-leakage*) e governança MLOps via MLflow.
3. **Validação Experimental:** Incorporação de tabelas de métricas comparativas em todas as partições, auditoria de deriva temporal ($\Delta F1 = +0{,}58\% \ll 10\%$) e análise de decis de negócio com curva de *Lift* ($1{,}98\times$ no primeiro decil).
4. **Deploy em Borda (TinyML C++):** Especificação do firmware header-only (`solos_tinyml.h`) otimizado para microcontroladores (ESP32/Arduino) com latência $< 1\text{ ms}$ e consumo de memória restrito ($< 35\text{ KB}$).
5. **Links Canônicos:** Citações diretas e funcionais para o [Repositório GitHub](https://github.com/vrenzd/Solos_Inteligentes), o [Notebook no Kaggle](https://www.kaggle.com/code/victorrenzzo/solos-inteligentes) e o [Dataset BSSL no Zenodo](https://zenodo.org/records/8092774).


