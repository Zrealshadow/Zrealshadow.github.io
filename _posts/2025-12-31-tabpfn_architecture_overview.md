---
layout: post
title: "Tabular Foundation Model: TabPFN model architecture"
author: "Lingze Zeng"
image: tfm.png
tags: [tabular learning]
---
## 1. Architecture Overview

### Model: PerFeatureTransformer

**Default Configuration**:
- Embedding dimension: **192**
- Attention heads: **6** (both feature-wise and item-wise)
- Transformer layers: **12**
- Max classes: **10**
- Positional embedding: **"subspace"** (NOT RoPE)

### Complete Pipeline

```
Input: X_train (100, 10) + X_test (50, 10), y_train (100,)
    ↓
Categorical Preprocessing (ordinal encoding per column)
    ↓
Linear Embedding (shared weights across features)
    Shape: (150, batch, 10, 192)
    ↓
Positional Encoding (unique per feature)
    Shape: (150, batch, 10, 192)
    ↓
12 × Transformer Layers (dual attention: feature-wise + item-wise)
    Shape: (batch, 150, 10, 192)
    ↓
Extract Test Encodings (target column only)
    Shape: (50, batch, 192)
    ↓
Decoder MLP (2-layer with GELU)
    Shape: (50, batch, 10) — logits
    ↓
Softmax → Argmax → Label Decoding
    Output: (50,) — predicted classes
```

### Dual Attention Mechanism

Each transformer layer applies **two attention operations sequentially**:

1. **Feature-wise Attention** (cross-column communication)
   - Features attend to each other across columns
   - Enables the model to learn feature interactions

2. **Item-wise Attention** (cross-row communication)
   - Samples attend to each other across rows
   - Enables in-context learning (test samples attend to training samples)

Both use **residual connections** to preserve information.


## 2. Feature Embedding: Raw Value → 192-dim Vector

### Step 1: Categorical Preprocessing

Categorical features are encoded using **ordinal encoding, independently per column**:

```python
# Example
Original:
  Color: ['red', 'blue', 'red']
  Size:  ['small', 'large', 'small']

After Ordinal Encoding (alphabetical per column):
  Color: [1, 0, 1]  # 'blue'=0, 'red'=1
  Size:  [1, 0, 1]  # 'large'=0, 'small'=1
```

**Important**: Same ordinal values in different columns (both =1) initially get the **same embedding** until positional encoding distinguishes them.

### Step 2: Linear Transformation

**For X (Features)**:
```python
# All features share the same linear layer
encoder = nn.Linear(1, 192)
```
Shape: `(seq_len, batch, 1)` → `(seq_len, batch, 192)`

**For y (Target Labels)**:
```python
# Step 1: NaN Handling
nan_indicators = torch.isnan(y) * -2.0 + torch.isinf(y) * 2.0
y[torch.isnan(y)] = training_mean  # Replace NaN with mean

# Step 2: Encode [value, nan_indicator] together
y_encoder = nn.Linear(2, 192)
```
Shape: `(seq_len, batch)` → `(seq_len, batch, 2)` → `(seq_len, batch, 192)`

This allows the model to know both the value **and** whether it was originally missing.

### Step 3: Feature Positional Encoding

**Method**: Additive "subspace" embedding (NOT RoPE)

```python
# 1. Generate deterministic random vectors (FIXED by seed=42)
random_vecs = torch.randn((num_features, 48), generator=fixed_rng)

# 2. Load pre-computed vectors (for reproducibility)
random_vecs = COL_EMBEDDING[:num_features]

# 3. Project to full dimension via learned linear layer
pos_embs = nn.Linear(48, 192)(random_vecs)  # (num_features, 192)

# 4. Add to feature embeddings
x += pos_embs[None, None]  # Broadcast across batch and sequence
```

**Result**: Features at different column positions get different embeddings.

### Complete Example

```
Input:
  Color='red', Size='small'

After Ordinal Encoding:
  Color=1, Size=1

After Linear Embedding (shared weights):
  Both → [0.1, -0.3, 0.5, ..., 0.2]  (same 192-dim vector)

After Positional Encoding (different per column):
  Color: [0.1, -0.3, ...] + [0.05,  0.12, ...]  = [0.15, -0.18, ...]
  Size:  [0.1, -0.3, ...] + [-0.08, 0.03, ...]  = [0.02, -0.27, ...]

Result: Different final embeddings for each feature
```



## 3. Output Decoding: 192-dim Vector → Predicted Class

### Step 1: Extract Test Set Encodings

After 12 transformer layers, extract only **test samples** and **target column**:

```python
# encoder_out: (batch, seq_len, num_features, 192)
test_out = encoder_out[:, single_eval_pos:, -1]  # (batch, num_test, 192)
```

- `single_eval_pos`: Index separating train/test (e.g., 100)
- `-1`: Target column (last feature)

### Step 2: Decoder MLP

```python
decoder = nn.Sequential(
    nn.Linear(192, 768),
    nn.GELU(),
    nn.Linear(768, n_classes)
)
```

Shape: `(num_test, batch, 192)` → `(num_test, batch, 10)` — logits

**Note**: Model outputs 10 logits (max classes). If the task has only 3 classes, only the first 3 are meaningful.

### Step 3: Prediction

```python
# 1. Softmax
probas = softmax(logits / temperature)  # (num_test, 10)

# 2. Argmax
class_indices = argmax(probas, axis=1)  # (num_test,) → [0, 1, 2, ...]

# 3. Decode to original labels
predictions = label_encoder.inverse_transform(class_indices)
```

### Complete Decoding Flow

```
Transformer Output:
  (50, batch, 192)
    ↓
Decoder MLP:
  (50, 10) — Logits: [-2.1, 3.5, 0.8, -1.2, ...]
    ↓
Softmax:
  (50, 10) — Probabilities: [0.01, 0.87, 0.06, 0.02, ...]
    ↓
Argmax:
  (50,) — Class indices: [1, 1, 0, 1, 2, ...]
    ↓
Label Decoder:
  (50,) — Original labels: ['cat', 'cat', 'dog', 'cat', 'bird', ...]
```

