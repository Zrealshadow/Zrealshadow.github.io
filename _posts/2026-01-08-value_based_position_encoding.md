# Value-Based Position Encoding for Numerical Features in Tabular Foundation Models

## 1. Data Types in Tabular Learning

In tabular data, features can be categorized into two types with fundamentally different mathematical properties:

| Feature Type | Mathematical Structure | Example | What Matters |
|--------------|----------------------|---------|--------------|
| **Categorical** | Nominal set | Color ∈ {red, blue, green} | Distinguishability only |
| **Numerical** | Ordered field (ℝ) | Age ∈ {25, 30, 35} | Magnitude + Order |

**Key difference:**

- **Categorical:** Only equality matters. "red ≠ blue" is all we need to know.
- **Numerical:** Order and distance matter. "25 < 30 < 35" and "|30-25| < |35-25|" are meaningful.

**Consider a toy dataset with two features:**

| Sample | Color (categorical) | Age (numerical) |
|--------|-------------------|-----------------|
| 1 | red | 25 |
| 2 | blue | 30 |
| 3 | red | 45 |

**Step 1: Categorical encoding** (ordinal encoder)
```
Color: red → 1, blue → 0
```

**Step 2: Unified linear embedding**

All values (categorical and numerical) are treated as scalars and mapped through a shared linear layer:

$$f(x) = Wx + b, \quad W \in \mathbb{R}^{128}, \quad b \in \mathbb{R}^{128}$$

Note: Preprocessing (normalization/standardization) may be applied but is omitted here for simplicity.

**Embeddings:**

Color column: $f(0) = b$, $f(1) = W + b$

Age column: $f(25) = 25W + b$, $f(30) = 30W + b$, $f(45) = 45W + b$

**Requirements:**

- **Categorical (Color):** Need $f(0) \neq f(1)$ ✓ Achieved if $W \neq 0$
- **Numerical (Age):** Need $f(25) < f(30) < f(45)$ and $|f(30)-f(25)| < |f(45)-f(25)|$
- **Question:** Are these implicit information in different types of data guaranteed?



## 2. Why Value Proximity Matters for In-Context Learning

### From Order to Proximity

Numerical data has a fundamental property: **order induces proximity**.

On the number line, values close to each other are considered similar:

$$\text{Position nearness} \iff \text{Value similarity}$$

For example: $25$ is more similar to $30$ than to $45$ because $|30 - 25| < |45 - 25|$.

This proximity concept is **structural**—it comes from the mathematical nature of numerical data, not from any specific dataset.

### ICL Requires Proximity-Based Attention

**ICL prediction mechanism:** For test sample with feature value $x_{\text{test}}$:

$$\hat{y}_{\text{test}} = \sum_{i=1}^{n} \alpha_i y_i, \quad \text{where } \alpha_i \propto \text{attention}(x_{\text{test}}, x_i^{\text{train}})$$

**Smoothness assumption:** For numerical features, conditional label distribution is smooth:

$$|x_1 - x_2| \text{ small} \implies p(y | x_1) \approx p(y | x_2)$$

This holds for many real-world relationships (age→income, temperature→failure rate, etc.).

**Implication:** For effective ICL, attention weights should respect value proximity:

$$|x_i - x_{\text{test}}| < |x_j - x_{\text{test}}| \implies \alpha_i > \alpha_j$$

In other words: **value proximity in 1D should translate to attention similarity in the model**.

For categorical data, we just distinguish two different value is enough.

For numerical data, we need to more fine-grained encoding to preserve this promixmity.



## 3. Current Encoding Limitation

### The Problem

Standard embedding $f(x) = Wx + b$ maps 1D values to high-dimensional space $\mathbb{R}^d$.

**Question:** Does attention similarity in $\mathbb{R}^d$ respect the original value proximity in 1D?

### Analysis

Attention similarity is computed as:

$$\text{sim}(f(x_1), f(x_2)) = f(x_1)^\top f(x_2)$$

Substituting $f(x) = Wx + b$:

$$
\begin{align}
\text{sim}(f(x_1), f(x_2)) &= (Wx_1 + b)^\top (Wx_2 + b) \\
&= x_1 x_2 \|W\|^2 + (x_1 + x_2) W^\top b + \|b\|^2
\end{align}
$$

**Key observation:** Similarity depends on:

1. **Product** $x_1 x_2$ (not distance $|x_1 - x_2|$)
2. Sum $x_1 + x_2$ weighted by arbitrary $W^\top b$
3. Constant offset $\|b\|^2$

### Counterexample

Consider three values: $x_1 = 1, x_2 = 2, x_3 = 100$

**Value proximity:** $|x_2 - x_1| = 1 < 99 = |x_3 - x_1|$ (so $x_2$ is much closer to $x_1$)

**Attention similarity:**
$$
\begin{align}
\text{sim}(f(x_1), f(x_2)) &= 1 \cdot 2 \|W\|^2 + 3 W^\top b + \|b\|^2 = 2\|W\|^2 + 3W^\top b + \|b\|^2\\
\text{sim}(f(x_1), f(x_3)) &= 1 \cdot 100 \|W\|^2 + 101 W^\top b + \|b\|^2 = 100\|W\|^2 + 101W^\top b + \|b\|^2
\end{align}
$$

If $W^\top b$ is large and positive, $\text{sim}(f(x_1), f(x_3))$ can be **much larger** than $\text{sim}(f(x_1), f(x_2))$, despite $x_3$ being far from $x_1$.

### Conclusion

**No architectural guarantee** that:

$$|x_i - x_{\text{test}}| < |x_j - x_{\text{test}}| \implies \text{sim}(f(x_i), f(x_{\text{test}})) > \text{sim}(f(x_j), f(x_{\text{test}}))$$

The mapping from **value proximity** (1D structure) to **attention similarity** (model behavior) is broken.





## 4. TabICL's Column-Wise Module

TabICL uses a three-stage architecture. Our proposal targets **Stage 1: Column-wise Embedding**, where each feature column is processed independently using Set Transformer.

**Column-wise processing:**

$$
\begin{align}
\text{features} &= \text{Linear}(\text{raw\_values}) \quad \in \mathbb{R}^{B \times T \times 128} \\
\text{context} &= \text{SetTransformer}(\text{features}) \\
\text{weights}, \text{biases} &= \text{Linear}(\text{context}) \\
\text{embeddings} &= \text{features} \odot \text{weights} + \text{biases}
\end{align}
$$

**Key property:** Set Transformer is permutation-invariant. It treats $[25, 30, 45]$ and $[45, 25, 30]$ identically, designed to capture distributional statistics independent of order.

**Implication:** For numerical columns, the proximity information (induced by order) is not explicitly encoded in the attention mechanism.



## 5.Proposal: Value-Based Position Encoding with RoPE

### Core Idea

**Add value-based inductive bias through RoPE to explicitly encode proximity in column-wise attention.**

Instead of relying on learned embeddings to capture value proximity, we propose to inject **proximity information** explicitly through value-based encoding in the attention mechanism for numerical column.

**Key insight:** This is not traditional "positional encoding" (which encodes sequence position). Rather, it's an **explicit inductive bias** that encodes the proximity structure inherent in numerical data—bridging value nearness (1D) to attention similarity (model).

---

### Proposed Method

#### Type-Aware Column Processing

| Column Type | Processing Strategy | Rationale |
|-------------|---------------------|-----------|
| **Numerical** | Value-based RoPE in Set Transformer | Encode proximity relationships |
| **Categorical** | Standard Set Transformer (no RoPE) | Order is meaningless |

#### Value Normalization

Normalize values to common range before using as positions:

$$p(x) = \frac{x - x_{\min}}{x_{\max} - x_{\min}}$$

where $x_{\min}, x_{\max}$ are column minimum and maximum.

**Properties:**
- Maps to $[0, 1]$ range (scale-invariant)
- Preserves relative distances: $|p(x_2) - p(x_1)| \propto |x_2 - x_1|$
- Efficient: $O(n)$ computation

#### RoPE with Value-Based Positions

**Standard attention:**

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^\top}{\sqrt{d}}\right) V$$

**Value-aware attention:**

$$
\begin{align}
Q', K' &= \text{RoPE}(Q, K, \text{positions}=p(\text{values})) \\
\text{Attention}(Q', K', V) &= \text{softmax}\left(\frac{Q'K'^\top}{\sqrt{d}}\right) V
\end{align}
$$

Attention weight between samples $i$ and $j$ depends on:
1. Content similarity: $Q_i \cdot K_j$ (learned)
2. Value proximity: RoPE bias based on $|p(x_i) - p(x_j)|$ (explicit)

#### Temperature Scaling (Optional)

$$\theta_{ij} = \frac{p(x_i) - p(x_j)}{\tau}$$

where $\tau$ is learnable. Higher $\tau$ → weaker bias; lower $\tau$ → sharper distance-based attention.



## 6. Evaluation

### Claim 1: Explicit Encoding of Partial Order

**Statement:** Value-based RoPE explicitly encodes partial order in attention mechanism.

**Justification:**

Standard attention: $A_{ij} \propto \exp\left(\frac{Q_i \cdot K_j}{\sqrt{d}}\right)$

RoPE attention: $A_{ij} \propto \exp\left(\frac{Q'_i \cdot K'_j}{\sqrt{d}}\right)$ where $Q', K'$ are rotated by $\theta_{ij} \propto \text{pos}_i - \text{pos}_j$

With $\text{pos}_i = p(\text{value}_i)$:

$$\theta_{ij} \propto p(\text{value}_i) - p(\text{value}_j) \propto \text{value}_i - \text{value}_j$$

**Effect:** Attention weight $A_{ij}$ biased by value difference. Small $|\text{value}_i - \text{value}_j|$ → higher attention.

**Conclusion:** Order relationship explicitly encoded in geometry, not learned from data.

---

### Claim 2: Compatibility with In-Context Learning

**Statement:** Value-based encoding aligns with ICL's similarity-based reasoning.

**Justification:**

ICL principle: $\hat{y}_{\text{test}} \approx \sum_i \alpha_i y_i$ where $\alpha_i \propto \text{similarity}(x_{\text{test}}, x_i^{\text{train}})$

Under smoothness assumption $p(y|x_1) \approx p(y|x_2)$ when $|x_1 - x_2|$ small:
- Training samples with values close to test sample should have higher $\alpha_i$
- Value-based RoPE implements this via attention bias
- Provides structural inductive bias aligned with ICL

**Related work:** Positional encodings improve performance in vision (spatial) and language (sequential) transformers.



## 7. Summary

### The Core Problem

**Proximity structure is lost in the mapping from values to attention in numerical data**

Numerical data has a fundamental property: **order induces proximity**. On the number line, position nearness = similarity.

For ICL, this proximity should translate to attention similarity (smoothness assumption: nearby values → similar labels).

**Current gap:** Standard embedding $f(x) = Wx + b$ maps to $\mathbb{R}^d$, but attention similarity $f(x_1)^\top f(x_2)$ depends on value **product** $x_1 x_2$, not **proximity** $|x_1 - x_2|$.

**Consequence:** No architectural guarantee that value proximity → attention similarity.

---

**Add value-based inductive bias through RoPE to explicitly encode proximity.**

**Method:**

1. Normalize: $p(x) = \frac{x - x_{\min}}{x_{\max} - x_{\min}}$
2. Type-aware processing:
   - Numerical: Apply RoPE with $\text{positions} = p(\text{values})$
   - Categorical: Standard Set Transformer
3. Effect: Attention bias based on $|p(x_i) - p(x_j)|$ (proximity-aware)

**Key distinction:** Not traditional positional encoding—explicit inductive bias bridging value proximity (1D structure) to attention similarity (model behavior).

**Central Insight:** For numerical features in tabular foundation models, **order induces proximity**, and proximity should translate to attention similarity. This structural mapping should be explicitly encoded, not implicitly learned.