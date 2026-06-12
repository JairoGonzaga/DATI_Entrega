# CNN-IDS para Redes Automotivas

Notebooks que implementam e adaptam um sistema de detecção de intrusão (IDS) baseado em CNN 2D para redes automotivas.

Baseado no artigo:
> *"A lightweight intrusion detection system for connected vehicles based on a convolutional neural network"* — Elsevier, 2021

---

## Pipeline Comum

Ambos os notebooks compartilham a mesma pipeline de feature engineering:

```
Pacotes/Mensagens
     ↓
Seleção de bytes relevantes
     ↓
Diferença temporal entre pacotes consecutivos (mod 256)
     ↓
Split em nibbles (alto e baixo) → representação 2D
     ↓
Janela deslizante W=44 → tensor (44, N_nibbles, 1)
     ↓
CNN 2D  ×3 blocos: Conv2D → BatchNorm → MaxPool → Dropout
     ↓
GlobalAveragePooling → Dense(64) → Sigmoid
     ↓
StratifiedKFold 5 folds
```

---

## Notebooks

### 1. `original-notebook.ipynb` — Reprodução do Artigo (AVTP / Ethernet)

**Objetivo**: Reproduzir o artigo original usando dados AVTP (Audio Video Transport Protocol) em Ethernet automotivo.

**Dataset**: `nairoooooos/ethernet` — arquivos `.pcap` com tráfego *indoors* e *driving*, nas versões original e com injeção de pacotes MPEG.

| Parâmetro | Valor |
|---|---|
| Protocolo | AVTP (Ethernet automotivo) |
| Tamanho de pacote | 438 bytes |
| Bytes selecionados | 58 |
| Input CNN | `(44, 116, 1)` |
| Ataque modelado | Injeção de frame MPEG (`single-MPEG-frame.pcap`) |
| Validação | 5-Fold Stratified Cross-Validation |

**Resultados:**

| Métrica | Média | Desvio Padrão |
|---|---|---|
| Accuracy | 99.90% | ±0.02% |
| Precision | 99.78% | ±0.04% |
| Recall | 99.90% | ±0.02% |
| F1-Score | 99.84% | ±0.03% |
| ROC-AUC | 99.998% | ±0.001% |

**Outputs gerados** (`Resultados_original`):
- `models_metrics.csv` — métricas por fold
- `metrics_summary.csv` — médias e desvio padrão
- `loss_curves.png` — curvas de loss por fold
- `metrics_per_fold.png` — gráfico de barras das métricas
- `confusion_matrix.png` — matriz de confusão consolidada

---

### 2. `comma-notebook.ipynb` — Adaptação para CAN Bus (Comma2K19)

**Objetivo**: Adaptar a metodologia do artigo para dados CAN bus reais do Comma2K19. Como o dataset não possui ataques reais, são simulados *replay attacks* sobre os dados de condução.

**Dataset**: `tkm2261/comma2k19-ld` — logs de condução real com estrutura `processed_log/CAN/raw_can/`.

| Parâmetro | Valor |
|---|---|
| Protocolo | CAN bus |
| Campos usados | CAN ID (2 bytes) + payload (8 bytes) = 10 bytes/msg |
| Input CNN | `(44, 20, 1)` |
| Barramento | `src=0` (62% das mensagens — preserva sequência temporal) |
| Ataque modelado | Replay attack simulado |
| Segmentos | 20 segmentos × ~89k mensagens |
| Validação | 5-Fold Stratified Cross-Validation |

**Diferenças em relação ao artigo:**

| | Artigo (AVTP) | Esta adaptação (CAN) |
|---|---|---|
| Bytes úteis | 58 bytes | 10 bytes |
| Nibbles/linha | 116 | 20 |
| Input CNN | (44, 116, 1) | **(44, 20, 1)** |
| Labels | Injeção real (pcap) | Replay attack simulado |

**Resultados:**

| Métrica | Média | Desvio Padrão |
|---|---|---|
| Accuracy | 89.56% | ±0.42% |
| Precision | 92.02% | ±0.85% |
| Recall | 96.21% | ±0.67% |
| F1-Score | 94.06% | ±0.20% |
| ROC-AUC | 91.19% | ±1.00% |

> A queda de performance em relação ao artigo é esperada — os ataques são *simulados* sobre dados reais, tornando o problema mais difícil do que injeções controladas em laboratório.

**Outputs gerados** (`Resultados_Replica_comma`):
- `metricas_por_fold.csv` — métricas por fold
- `metricas_resumo.csv` — médias e desvio padrão
- `loss_curves.png` — curvas de loss por fold
- `metricas_por_fold.png` — gráfico de barras das métricas
- `confusion_matrix.png` — matriz de confusão consolidada

---

## Comparativo

| Notebook | Dataset | Input CNN | Accuracy | F1-Score |
|---|---|---|---|---|
| `original-notebook` | AVTP Ethernet (pcap) | (44, 116, 1) | **99.90%** | **99.84%** |
| `comma-notebook` | Comma2K19 CAN bus | (44, 20, 1) | 89.56% | 94.06% |

---

## Arquitetura CNN

```
Input (44, N, 1)
  ↓
Conv2D(32, 3×3, relu) → BatchNorm → MaxPool(2×2) → Dropout(0.2)
  ↓
Conv2D(64, 3×3, relu) → BatchNorm → MaxPool(2×2) → Dropout(0.3)
  ↓
Conv2D(128, 3×3, relu) → GlobalAveragePooling2D → Dropout(0.4)
  ↓
Dense(64, relu) → Dense(1, sigmoid, dtype=float32)
```

Otimizações GPU aplicadas em ambos os notebooks:
- Mixed Precision (`float16`)
- XLA JIT compilation
- Memory Growth
- `tf.data` com `cache()` + `prefetch(AUTOTUNE)`

---

## Como Rodar no Kaggle

1. Faça fork do notebook desejado
2. Adicione os datasets listados acima
3. Ative aceleração **GPU P100**
4. Execute todas as células (`Run All`)

**Dependências** (ambiente Kaggle):
- TensorFlow 2.x, NumPy, Pandas, scikit-learn, Matplotlib
- Scapy — instalada via `!pip install scapy` no `original-notebook`

---

## Referência

```bibtex
@article{atallah2021lightweight,
  title={A lightweight intrusion detection system for connected vehicles
         based on a convolutional neural network},
  journal={Elsevier},
  year={2021}
}
```
