# Day 8: Machine Learning Fundamentals & XGBoost

**Navigation:** [← Previous: Day 7](README-DAY-007.md) | [Main Guide](README.md) | [Next: Day 9 →](README-DAY-009.md)

---

## Table of Contents

1. [Overview](#overview)
2. [Understanding Machine Learning](#understanding-machine-learning)
3. [Supervised Learning Deep Dive](#supervised-learning-deep-dive)
4. [Classification for Fraud Detection](#classification-for-fraud-detection)
5. [Decision Trees and Gradient Boosting](#decision-trees-and-gradient-boosting)
6. [XGBoost Algorithm Explained](#xgboost-algorithm-explained)
7. [Project Structure Setup](#project-structure-setup)
8. [Complete Code Implementation](#complete-code-implementation)
9. [Installation & Environment Setup](#installation--environment-setup)
10. [Simple XGBoost Examples](#simple-xgboost-examples)
11. [Understanding Model Evaluation](#understanding-model-evaluation)
12. [Testing & Verification](#testing--verification)
13. [Next Steps](#next-steps)

---

## Overview

### What You'll Learn Today

Welcome to Day 8! Today is **conceptual and foundational** - we're building your understanding of machine learning and setting up the infrastructure for fraud detection ML models. **We will NOT train a fraud model today** - that comes later after you understand the fundamentals.

**Day 8 Objectives:**
- ✅ Understand machine learning fundamentals
- ✅ Learn supervised vs unsupervised learning
- ✅ Deep dive into classification problems
- ✅ Understand decision trees and gradient boosting
- ✅ Master XGBoost algorithm concepts
- ✅ Set up ML project structure
- ✅ Install ML packages (scikit-learn, XGBoost, pandas)
- ✅ Run simple XGBoost examples (iris dataset)
- ✅ Learn model evaluation metrics

**What We WON'T Do Today:**
- ❌ Train fraud detection models (Day 9)
- ❌ Feature engineering for transactions (Day 9)
- ❌ Model deployment (Day 10+)
- ❌ Production ML pipelines (Day 10+)

**Time Estimate:** 3-4 hours (lots of reading and understanding)

**Prerequisites:**
- Completed Day 2: Database with transactions table
- Completed Day 3: Pydantic schemas for transactions
- Understanding of Python basics
- Basic statistics knowledge (mean, variance, probability)

**Philosophy:**
> "Machine learning is not magic - it's statistics and optimization applied to data. Understanding the fundamentals deeply will make you a better ML engineer than knowing 100 algorithms superficially."

---

## Understanding Machine Learning

### What is Machine Learning?

**Traditional Programming:**
```
Input Data + Rules (written by humans) → Output
Example: IF amount > $10,000 THEN fraud = True
```

**Machine Learning:**
```
Input Data + Expected Output → Rules (learned by algorithm)
Example: Algorithm learns: amount, velocity, location → fraud probability
```

**Definition:**
Machine Learning is the practice of using algorithms to parse data, learn from it, and then make predictions or decisions without being explicitly programmed to perform the task.

### The Three Types of Machine Learning

#### 1. Supervised Learning (What We'll Use)

**Definition:** Learning from labeled examples - you have input data AND the correct answers.

**Example - Email Spam Detection:**
```
Training Data:
Email 1: "Buy cheap watches now!" → Label: SPAM
Email 2: "Meeting tomorrow at 3pm" → Label: NOT SPAM
Email 3: "You won the lottery!" → Label: SPAM

Algorithm learns patterns:
- Words like "buy", "cheap", "won" → likely SPAM
- Words like "meeting", "tomorrow" → likely NOT SPAM
```

**For Fraud Detection:**
```
Training Data:
Transaction 1: amount=$50, velocity=2, location_change=0 → Label: NOT FRAUD
Transaction 2: amount=$5000, velocity=15, location_change=500mi → Label: FRAUD

Algorithm learns: high amount + high velocity + location change → FRAUD
```

**Supervised Learning Process:**
```
1. Collect labeled data (transactions with fraud/not fraud labels)
2. Split data: 80% training, 20% testing
3. Algorithm learns patterns from training data
4. Test algorithm on unseen test data
5. Measure accuracy, precision, recall
6. Deploy model to production
```

#### 2. Unsupervised Learning

**Definition:** Learning from unlabeled data - you only have inputs, no correct answers.

**Example - Customer Segmentation:**
```
Input: Customer purchase data (no labels)
Algorithm finds clusters:
- Cluster 1: High spenders, luxury items
- Cluster 2: Budget shoppers, practical items
- Cluster 3: Seasonal buyers
```

**For Fraud Detection (Anomaly Detection):**
```
Input: All transactions (no fraud labels)
Algorithm identifies outliers:
- Transaction is very different from normal patterns → Flag for review
```

**When to Use:**
- No labeled data available
- Discovering unknown patterns
- Anomaly detection
- Dimensionality reduction

#### 3. Reinforcement Learning

**Definition:** Learning through trial and error with rewards/penalties.

**Example - Game Playing:**
```
Agent plays chess:
- Makes move → Gets reward (+1 for winning, -1 for losing)
- Learns which moves lead to wins
```

**For Fraud Detection (Advanced):**
```
System decides: approve, reject, or manual review
- Correct fraud catch → +10 reward
- False positive (reject good transaction) → -5 penalty
- Missed fraud → -20 penalty

System learns optimal decision policy
```

**When to Use:**
- Sequential decision-making
- Trading off immediate vs long-term rewards
- Game playing, robotics, optimization

### Why Supervised Learning for Fraud Detection?

Fraud detection is a **perfect supervised learning problem** because:

1. **We have labels:** Transactions are labeled as fraud or legitimate (from chargebacks, reports)
2. **Clear objective:** Binary classification (fraud or not fraud)
3. **Historical data:** Banks have millions of past transactions
4. **Measurable outcome:** Can measure accuracy, false positives, false negatives
5. **Interpretability:** Need to explain why a transaction was flagged

**Real-World Example:**

PayPal uses supervised learning with:
- **10+ billion** labeled transactions
- **Hundreds** of features per transaction
- **Ensemble models** (multiple ML algorithms combined)
- **99.5%+ accuracy** in fraud detection

---

## Supervised Learning Deep Dive

### The Supervised Learning Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SUPERVISED LEARNING PIPELINE                      │
└─────────────────────────────────────────────────────────────────────┘

1. DATA COLLECTION
   ┌──────────────────────────────────────┐
   │ Collect historical transactions       │
   │ - Transaction ID, amount, time, user │
   │ - Label: fraud (1) or not fraud (0)  │
   └──────────────────────────────────────┘
                    ↓
2. EXPLORATORY DATA ANALYSIS (EDA)
   ┌──────────────────────────────────────┐
   │ - Check data quality                  │
   │ - Visualize distributions             │
   │ - Find correlations                   │
   │ - Handle missing values               │
   └──────────────────────────────────────┘
                    ↓
3. FEATURE ENGINEERING
   ┌──────────────────────────────────────┐
   │ - Create meaningful features          │
   │ - Encode categorical variables        │
   │ - Scale numerical features            │
   │ - Handle time-based features          │
   └──────────────────────────────────────┘
                    ↓
4. TRAIN/TEST SPLIT
   ┌──────────────────────────────────────┐
   │ Training Set (80%)                    │
   │ - Used to train the model             │
   │                                       │
   │ Test Set (20%)                        │
   │ - Used to evaluate model              │
   │ - Model has NEVER seen this data      │
   └──────────────────────────────────────┘
                    ↓
5. MODEL TRAINING
   ┌──────────────────────────────────────┐
   │ Algorithm learns patterns:            │
   │ Features → Label                      │
   │                                       │
   │ Adjusts internal parameters           │
   │ to minimize prediction errors         │
   └──────────────────────────────────────┘
                    ↓
6. MODEL EVALUATION
   ┌──────────────────────────────────────┐
   │ Test on unseen data                   │
   │ - Accuracy, Precision, Recall         │
   │ - Confusion Matrix                    │
   │ - ROC Curve, AUC                      │
   └──────────────────────────────────────┘
                    ↓
7. MODEL DEPLOYMENT
   ┌──────────────────────────────────────┐
   │ Deploy to production                  │
   │ - API endpoint                        │
   │ - Real-time predictions               │
   │ - Monitor performance                 │
   └──────────────────────────────────────┘
```

### Features vs Labels

Understanding the difference is crucial:

**Features (X):**
- **What:** Input variables used to make predictions
- **Also called:** Independent variables, predictors, attributes
- **In fraud detection:** transaction amount, time, location, user history

**Labels (y):**
- **What:** Output variable we want to predict
- **Also called:** Dependent variable, target, response
- **In fraud detection:** fraud (1) or not fraud (0)

**Example:**

```python
# Transaction data
transactions = [
    # Features                                    Label
    [100.00, 2, "New York", "12:00"],          # 0 (not fraud)
    [5000.00, 15, "Paris", "12:05"],           # 1 (fraud)
    [25.50, 1, "New York", "14:30"],           # 0 (not fraud)
]

# Feature matrix X (what we know)
X = [
    [100.00, 2, "New York", "12:00"],
    [5000.00, 15, "Paris", "12:05"],
    [25.50, 1, "New York", "14:30"],
]

# Label vector y (what we want to predict)
y = [0, 1, 0]
```

### Training vs Inference

Two distinct phases in machine learning:

**Training (Learning Phase):**
```python
# Learning from historical data
model = XGBoost()
model.fit(X_train, y_train)  # Learn patterns

# What happens internally:
# - Algorithm adjusts parameters
# - Minimizes prediction errors
# - Learns: "high amount + high velocity = fraud"
```

**Inference (Prediction Phase):**
```python
# Predicting on new, unseen data
new_transaction = [2500.00, 8, "London", "15:30"]
prediction = model.predict(new_transaction)  # → 0.85 (85% fraud probability)

# What happens internally:
# - Uses learned parameters
# - NO learning/updating
# - Fast (milliseconds)
```

**Key Differences:**

| Aspect | Training | Inference |
|--------|----------|-----------|
| **Frequency** | Once (or periodic retraining) | Every transaction |
| **Data Required** | Thousands/millions of examples | Single transaction |
| **Time** | Minutes to hours | Milliseconds |
| **Computational Cost** | High (GPU often used) | Low (CPU sufficient) |
| **Purpose** | Learn patterns | Apply patterns |

### Overfitting vs Underfitting

Critical concepts that affect model performance:

**Underfitting (Model Too Simple):**
```
Model doesn't learn enough from training data
Example: Using only transaction amount to predict fraud

Training Accuracy: 60%
Test Accuracy: 58%

Problem: Model misses important patterns
Solution: Add more features, use more complex model
```

**Perfect Fit (Just Right):**
```
Model learns general patterns

Training Accuracy: 95%
Test Accuracy: 93%

Goal: Close training and test accuracy
Solution: Proper model complexity and regularization
```

**Overfitting (Model Too Complex):**
```
Model memorizes training data instead of learning patterns
Example: Model learns "Transaction #12345 is fraud" (useless for new data)

Training Accuracy: 99.9%
Test Accuracy: 65%

Problem: Model fails on new data
Solution: Regularization, more data, simpler model
```

**Visual Representation:**

```
Underfitting:
  Actual data: ●  ●  ●    ●
                 ●    ●  ●
  Model fit:    ──────────────  (straight line - too simple)

Good Fit:
  Actual data: ●  ●  ●    ●
                 ●    ●  ●
  Model fit:    ╱──╲  ╱──╲  (smooth curve - captures pattern)

Overfitting:
  Actual data: ●  ●  ●    ●
                 ●    ●  ●
  Model fit:    ╱╲╱╲╱╲╱╲╱╲  (zigzag - memorizes noise)
```

**Detecting Overfitting:**
```python
# Large gap between training and test performance
training_accuracy = 0.99  # 99%
test_accuracy = 0.65      # 65%
gap = 0.34                # 34% gap → OVERFITTING!

# Solution:
# 1. Add more training data
# 2. Reduce model complexity
# 3. Use regularization
# 4. Cross-validation
```

---

## Classification for Fraud Detection

### What is Classification?

**Classification:** Predicting a categorical label (class) for a given input.

**Types:**
1. **Binary Classification:** 2 classes (fraud or not fraud)
2. **Multi-class Classification:** 3+ classes (low risk, medium risk, high risk)

Fraud detection is typically **binary classification.**

### Binary Classification Explained

**Goal:** Given features of a transaction, predict: Fraud (1) or Not Fraud (0)

**How it works:**
```
Input:
  Transaction features: [amount=1000, velocity=5, location_change=100]

Output:
  Probability: 0.85 (85% chance of fraud)
  Class: 1 (fraud) if probability > threshold (e.g., 0.5)
```

**Decision Boundary:**
```
                    Probability of Fraud
                    ↑
               1.0  |           ████████  (Fraud region)
                    |       ████
                    |   ████
Threshold → 0.5     |███──────────────── (Decision boundary)
                    |████
                    |    ████
               0.0  |        ████████   (Not Fraud region)
                    └──────────────────→
                         Feature value
```

**Example with Real Numbers:**
```python
# Transaction 1
features = [50.00, 1, 0]  # Small amount, low velocity, same location
probability = 0.12        # 12% fraud probability
prediction = 0            # Not fraud (< 0.5 threshold)

# Transaction 2
features = [5000.00, 20, 500]  # Large amount, high velocity, different location
probability = 0.92             # 92% fraud probability
prediction = 1                 # Fraud (> 0.5 threshold)
```

### Classification vs Regression

Understanding the difference:

**Classification (Discrete Output):**
```python
# Predict category
is_fraud = model.predict(transaction)  # Output: 0 or 1
risk_level = model.predict(transaction)  # Output: "low", "medium", "high"

Examples:
- Email spam detection (spam/not spam)
- Image recognition (cat/dog/bird)
- Disease diagnosis (positive/negative)
- Fraud detection (fraud/not fraud)
```

**Regression (Continuous Output):**
```python
# Predict number
price = model.predict(house_features)  # Output: $450,000
temperature = model.predict(weather_data)  # Output: 72.5°F

Examples:
- House price prediction
- Stock price forecasting
- Temperature prediction
- Sales forecasting
```

**For Fraud Detection:**
We use **classification** because:
- Output is binary (fraud or not fraud)
- Need probability score (for ranking/prioritization)
- Can set custom thresholds (trade off precision vs recall)

However, we could also use **regression** to predict:
- Fraud amount (how much money will be lost)
- Days until chargeback
- Customer lifetime value

---

## Decision Trees and Gradient Boosting

### Decision Trees Fundamentals

**What is a Decision Tree?**

A decision tree is a flowchart-like structure where:
- Each internal node represents a "test" on a feature
- Each branch represents the outcome of the test
- Each leaf node represents a class label (fraud/not fraud)

**Simple Example - Fraud Detection Tree:**

```
                    [Transaction Amount]
                           |
          ┌────────────────┴────────────────┐
          |                                  |
       < $100                             ≥ $100
          |                                  |
    [NOT FRAUD]                      [Velocity]
     (98% sure)                            |
                           ┌───────────────┴───────────────┐
                           |                               |
                        ≤ 5 tx/hr                     > 5 tx/hr
                           |                               |
                     [NOT FRAUD]                      [FRAUD]
                      (95% sure)                     (85% sure)
```

**How to Read This Tree:**
```
Rule 1: IF amount < $100 THEN not fraud (98% confidence)
Rule 2: IF amount ≥ $100 AND velocity ≤ 5 THEN not fraud (95% confidence)
Rule 3: IF amount ≥ $100 AND velocity > 5 THEN fraud (85% confidence)
```

**More Complex Real-World Tree:**

```
                                    [Amount]
                                       |
                    ┌──────────────────┴──────────────────┐
                    |                                      |
                 < $500                                 ≥ $500
                    |                                      |
              [NOT FRAUD]                            [Velocity]
                                                          |
                               ┌──────────────────────────┴──────────────────┐
                               |                                              |
                            ≤ 3 tx/hr                                    > 3 tx/hr
                               |                                              |
                     [Location Change]                              [Time of Day]
                               |                                              |
                ┌──────────────┴──────────────┐              ┌────────────────┴─────────────┐
                |                              |              |                              |
            < 50 miles                    ≥ 50 miles      Business Hours              Night (2-6 AM)
                |                              |              |                              |
          [NOT FRAUD]                      [FRAUD]      [NOT FRAUD]                      [FRAUD]
                                                          (Manual Review)               (High Risk)
```

### How Decision Trees Learn

**Algorithm (Simplified):**

```
1. Start with all training data at root node

2. For each feature:
   - Try different split points
   - Calculate "information gain" or "gini impurity"
   - Choose split that best separates fraud from not fraud

3. Create two child nodes based on best split

4. Repeat steps 2-3 for each child node

5. Stop when:
   - Node is pure (all fraud or all not fraud)
   - Maximum depth reached
   - Too few samples to split
```

**Example - Building a Tree:**

```
Training Data (10 transactions):
Amount  Velocity  Fraud?
$50     2         No
$100    1         No
$500    8         Yes
$1000   10        Yes
$80     3         No
$2000   15        Yes
$45     2         No
$750    12        Yes
$120    4         No
$3000   20        Yes

Step 1: Try splitting on Amount at $100
  Left (< $100): [No, No, No, No] → 100% not fraud ✓ Pure!
  Right (≥ $100): [No, Yes, Yes, Yes, Yes, No, Yes] → Mixed

Step 2: For right side, try splitting on Velocity at 5
  Left (≤ 5): [No, No] → 100% not fraud ✓ Pure!
  Right (> 5): [Yes, Yes, Yes, Yes, Yes] → 100% fraud ✓ Pure!

Final Tree:
  IF amount < $100 → NOT FRAUD
  ELIF velocity ≤ 5 → NOT FRAUD
  ELSE → FRAUD
```

### Decision Tree Advantages & Limitations

**Advantages:**
- ✅ **Easy to interpret:** Can visualize and explain to stakeholders
- ✅ **No feature scaling needed:** Works with raw values
- ✅ **Handles mixed data types:** Numbers and categories together
- ✅ **Feature importance:** Shows which features matter most
- ✅ **Fast prediction:** Just follow the tree path

**Limitations:**
- ❌ **Overfitting:** Can memorize training data (deep trees)
- ❌ **Unstable:** Small data changes can change tree structure
- ❌ **Biased:** Favors features with many categories
- ❌ **Cannot capture complex patterns:** Limited to axis-aligned splits
- ❌ **Poor generalization:** Single tree rarely performs well

**Solution:** Use **many trees together** → Ensemble Methods

### Ensemble Methods: Boosting

**Core Idea:**
Instead of one tree, use many trees and combine their predictions.

**Two Main Approaches:**

**1. Bagging (Bootstrap Aggregating):**
```
Train many trees independently on different subsets of data
Combine predictions by voting

Example: Random Forest
Tree 1 → Fraud
Tree 2 → Not Fraud
Tree 3 → Fraud
Final: Fraud (2/3 vote)
```

**2. Boosting (Sequential Learning):**
```
Train trees sequentially, each correcting errors of previous trees
Combine predictions by weighted sum

Example: XGBoost, AdaBoost, Gradient Boosting
Tree 1 → Predicts 70% of cases correctly
Tree 2 → Focuses on 30% that Tree 1 got wrong
Tree 3 → Focuses on remaining errors
Final: Weighted combination of all trees
```

**Boosting Process:**

```
Round 1:
  Train tree on all data
  Identify misclassified examples
  Increase weight of misclassified examples

Round 2:
  Train tree focusing on hard examples
  Identify new misclassified examples
  Increase their weights

Round 3:
  Train tree on hardest examples
  Continue...

Final Model:
  Weighted sum of all trees
  prediction = α₁×tree₁ + α₂×tree₂ + α₃×tree₃ + ...
```

**Why Boosting Works:**

```
Weak Learner (single tree):
  Accuracy: 60% (barely better than random)

Ensemble (100 trees):
  Accuracy: 95% (each tree fixes previous errors)
```

**Visualization:**

```
Data Points:        ● = fraud, ○ = not fraud

After Tree 1:
  Correct:   ●●●●  ○○○○○
  Errors:    ●● (missed)  ○ (false positive)

After Tree 2:
  Tree 1 + Tree 2 fixed most errors
  New errors:  ● (missed)

After Tree 100:
  Almost perfect classification
  Very few errors remain
```

### Gradient Boosting Explained

**What is Gradient Boosting?**

Gradient Boosting is a specific type of boosting that:
1. Uses **gradient descent** to minimize errors
2. Each tree predicts the **residual errors** of previous trees
3. Combines trees additively to improve predictions

**Mathematical Intuition (Simplified):**

```
Goal: Predict fraud probability P(fraud)

Step 1: Start with average
  prediction₁ = mean(y_train) = 0.1 (10% of transactions are fraud)

Step 2: Calculate errors (residuals)
  For transaction i:
    error_i = actual_i - prediction₁
    Example: actual=1, prediction=0.1 → error=0.9

Step 3: Train tree to predict errors
  tree₁ learns to predict the 0.9 error

Step 4: Update predictions
  prediction₂ = prediction₁ + learning_rate × tree₁
  Example: prediction₂ = 0.1 + 0.3 × 0.8 = 0.34

Step 5: Repeat
  Calculate new errors: actual - prediction₂
  Train tree₂ to predict these errors
  Update: prediction₃ = prediction₂ + learning_rate × tree₂

Continue for N trees...
```

**Key Parameters:**

```python
GradientBoostingClassifier(
    n_estimators=100,      # Number of trees
    learning_rate=0.1,     # Step size (smaller = more conservative)
    max_depth=3,           # Maximum tree depth
    min_samples_split=2,   # Minimum samples to split node
    subsample=0.8,         # Fraction of samples for each tree
)
```

**Learning Rate Tradeoff:**

```
High Learning Rate (0.5):
  - Fast training (fewer trees needed)
  - Risk of overfitting
  - May miss optimal solution

Low Learning Rate (0.01):
  - Slow training (many trees needed)
  - Better generalization
  - More likely to find optimal solution

Common practice: learning_rate=0.1, n_estimators=100
```

---

## XGBoost Algorithm Explained

### What is XGBoost?

**XGBoost (eXtreme Gradient Boosting)** is an optimized implementation of gradient boosting that:
- Trains faster (parallel processing, GPU support)
- Regularizes better (prevents overfitting)
- Handles missing values automatically
- Wins most ML competitions (Kaggle, etc.)

**Created by:** Tianqi Chen (2014) at University of Washington

**Used by:** Almost every tech company
- Airbnb: Fraud detection, pricing
- Uber: ETA prediction, fraud detection
- Microsoft: Malware detection
- PayPal: Fraud detection

### Why XGBoost for Fraud Detection?

**Advantages for Fraud:**

1. **Handles Imbalanced Data:**
   ```
   Typical fraud dataset:
   - 99.5% legitimate transactions
   - 0.5% fraudulent transactions

   XGBoost handles this well with:
   - scale_pos_weight parameter
   - Custom evaluation metrics
   ```

2. **Fast Inference:**
   ```
   Prediction time: < 1ms per transaction
   Critical for real-time fraud detection
   ```

3. **Feature Importance:**
   ```
   Shows which features matter:
   - Amount: 35% importance
   - Velocity: 28% importance
   - Location change: 22% importance
   - Time of day: 15% importance

   Helps explain decisions to investigators
   ```

4. **Robust to Overfitting:**
   ```
   Built-in regularization:
   - L1 regularization (lasso)
   - L2 regularization (ridge)
   - Maximum depth limits
   - Minimum child weight
   ```

5. **Handles Missing Data:**
   ```
   Transaction missing location data?
   XGBoost learns best direction:
   - If most fraud has missing location → go fraud direction
   - If not → go not fraud direction
   ```

### XGBoost Algorithm Deep Dive

**Core Algorithm:**

```python
# Simplified XGBoost training process

1. Initialize with base prediction (mean)
   f₀(x) = log(p/(1-p))  # Log odds of average fraud rate

2. For m = 1 to M trees:

   a. Calculate pseudo-residuals (gradient)
      For each sample i:
        g_i = ∂L/∂f(x_i)  # Gradient of loss function

   b. Calculate second-order gradient (Hessian)
      For each sample i:
        h_i = ∂²L/∂f(x_i)²  # Curvature information

   c. Build tree by optimizing:
      Gain = (G_L²)/(H_L + λ) + (G_R²)/(H_R + λ) - (G²)/(H + λ) - γ

      Where:
        G_L = sum of gradients in left child
        G_R = sum of gradients in right child
        H_L = sum of hessians in left child
        H_R = sum of hessians in right child
        λ = L2 regularization
        γ = Complexity penalty

   d. Update model:
      f_m(x) = f_{m-1}(x) + η × tree_m(x)

      Where η = learning rate

3. Final prediction:
   f_M(x) = f_0(x) + Σ(η × tree_m(x))
   probability = 1 / (1 + e^(-f_M(x)))  # Sigmoid transform
```

**What Makes XGBoost "Extreme"?**

**1. Second-Order Optimization:**
```
Regular Gradient Boosting:
  Uses first derivative (gradient)
  Like walking downhill blindfolded

XGBoost:
  Uses second derivative (Hessian)
  Like having a map of the terrain
  Faster convergence, better accuracy
```

**2. Regularization:**
```python
Objective = Loss + Regularization

Loss = Σ(y_i - ŷ_i)²  # How wrong are predictions

Regularization = α × |weights| + β × weights²
                 ↑                 ↑
               L1 (lasso)      L2 (ridge)

Prevents overfitting by penalizing complex models
```

**3. Sparsity-Aware:**
```
Missing values are common in real data:

Transaction features:
  amount: $100
  velocity: 5
  location: MISSING  ← XGBoost handles this automatically

Algorithm learns:
  "For missing location, go left in tree"
```

**4. Parallel Processing:**
```
Traditional Gradient Boosting:
  Build tree sequentially (slow)

  Node 1 → Node 2 → Node 3 → ... (one at a time)

XGBoost:
  Build tree levels in parallel (fast)

  Level 1: Node 1
  Level 2: Node 2, Node 3 (parallel)
  Level 3: Node 4, Node 5, Node 6, Node 7 (parallel)

Speed up: 10x faster on multi-core CPUs
```

**5. Cache Optimization:**
```
Data access pattern:
  Store data in blocks
  Compress blocks
  Use CPU cache efficiently

Result: Faster training, less memory
```

### XGBoost Key Parameters

Understanding these parameters is crucial for effective fraud detection:

```python
import xgboost as xgb

model = xgb.XGBClassifier(
    # ===== Tree Parameters =====
    max_depth=6,
    # Maximum depth of each tree
    # Higher = more complex model
    # Fraud detection: 3-6 typical
    # Too high → overfitting

    min_child_weight=1,
    # Minimum sum of instance weight needed in a child
    # Higher = more conservative (prevents overfitting)
    # Fraud detection: 1-10

    # ===== Boosting Parameters =====
    learning_rate=0.1,
    # Also called 'eta'
    # Step size shrinkage
    # Lower = more robust but needs more trees
    # Fraud detection: 0.01-0.3

    n_estimators=100,
    # Number of boosting rounds (trees)
    # More trees = better fit (until overfitting)
    # Fraud detection: 100-1000

    # ===== Regularization =====
    gamma=0,
    # Minimum loss reduction required to split
    # Higher = more conservative
    # Fraud detection: 0-5

    reg_alpha=0,
    # L1 regularization on weights
    # Higher = more feature selection
    # Fraud detection: 0-1

    reg_lambda=1,
    # L2 regularization on weights
    # Higher = more conservative
    # Fraud detection: 1-10

    # ===== Sampling Parameters =====
    subsample=0.8,
    # Fraction of samples used per tree
    # Lower = prevents overfitting
    # Fraud detection: 0.6-1.0

    colsample_bytree=0.8,
    # Fraction of features used per tree
    # Lower = prevents overfitting
    # Fraud detection: 0.6-1.0

    # ===== Imbalanced Data =====
    scale_pos_weight=1,
    # Balance positive/negative weights
    # fraud detection: (# not fraud) / (# fraud)
    # Example: If 0.5% fraud → scale_pos_weight=199

    # ===== Performance =====
    n_jobs=-1,
    # Number of parallel threads (-1 = use all cores)

    random_state=42,
    # Random seed for reproducibility

    # ===== Evaluation =====
    eval_metric='auc',
    # Metric for validation
    # Options: 'auc', 'logloss', 'error', 'map'
)
```

**Parameter Tuning Strategy:**

```python
# Step 1: Start with defaults
model = xgb.XGBClassifier()

# Step 2: Tune number of trees
# Use early stopping to find optimal n_estimators
model = xgb.XGBClassifier(
    n_estimators=1000,
    early_stopping_rounds=50,  # Stop if no improvement for 50 rounds
)

# Step 3: Tune max_depth and min_child_weight
# Try: max_depth=[3,4,5,6], min_child_weight=[1,3,5]

# Step 4: Tune learning_rate
# Lower learning_rate, increase n_estimators

# Step 5: Tune regularization
# Try: reg_alpha=[0,0.1,1], reg_lambda=[1,5,10]

# Step 6: Tune sampling
# Try: subsample=[0.6,0.8,1.0], colsample_bytree=[0.6,0.8,1.0]
```

### XGBoost vs Other Algorithms

Comparing XGBoost to alternatives:

| Algorithm | Speed | Accuracy | Interpretability | Handles Imbalanced | Best For |
|-----------|-------|----------|------------------|-------------------|----------|
| **XGBoost** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Structured/tabular data |
| **Random Forest** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Robust baseline |
| **Logistic Regression** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | Linear relationships |
| **Neural Networks** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐ | Complex patterns, images |
| **Decision Tree** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | Simple rules |

**When to Choose XGBoost:**
- ✅ Tabular data (features in columns)
- ✅ Need high accuracy
- ✅ Need fast predictions (< 10ms)
- ✅ Imbalanced datasets
- ✅ Mixed feature types (numbers, categories)
- ✅ Some interpretability needed

**When NOT to Choose XGBoost:**
- ❌ Image data (use CNNs)
- ❌ Text data (use transformers)
- ❌ Time series (use LSTMs or specialized models)
- ❌ Online learning (XGBoost is batch)
- ❌ Need perfect interpretability (use logistic regression)

---

## Project Structure Setup

### Directory Layout

We'll create a dedicated `ml` package for all machine learning code:

```
sentinel/
├── app/
│   ├── __init__.py                 # From Day 1
│   ├── main.py                     # From Day 1
│   ├── database.py                 # From Day 2
│   ├── models.py                   # From Day 2
│   ├── schemas.py                  # From Day 3
│   ├── config.py                   # From Day 3
│   ├── rules/                      # From Day 4
│   │   ├── __init__.py
│   │   ├── engine.py
│   │   └── ...
│   │
│   └── ml/                         # NEW - Day 8
│       ├── __init__.py             # ML package initialization
│       ├── config.py               # ML configuration settings
│       └── utils.py                # Helper functions
│
├── notebooks/                      # OPTIONAL - For exploration
│   └── 01_xgboost_basics.ipynb    # Jupyter notebooks
│
├── .env                            # From Day 1
├── requirements.txt                # Updated with ML packages
└── README.md                       # From Day 1
```

### File Responsibilities

| File | Purpose | Lines |
|------|---------|-------|
| `ml/__init__.py` | Package exports | ~15 |
| `ml/config.py` | ML configuration and constants | ~60 |
| `ml/utils.py` | Helper functions for ML tasks | ~150 |

### Why This Structure?

**Separation of Concerns:**
```
app/
  ├── rules/     # Rule-based fraud detection
  └── ml/        # Machine learning fraud detection

Both work independently, can be combined later
```

**Scalability:**
```
Future additions:
app/ml/
  ├── preprocessing.py   # Feature engineering
  ├── models.py          # Model definitions
  ├── training.py        # Training pipelines
  ├── evaluation.py      # Metrics and evaluation
  ├── inference.py       # Production inference
  └── explainability.py  # SHAP, feature importance
```

---

## Complete Code Implementation

### 1. app/ml/__init__.py

Package initialization and exports.

```python
"""
Sentinel Machine Learning Package

This package contains machine learning components for fraud detection:
- Model training and evaluation
- Feature engineering
- Model inference
- Explainability and interpretability

The ML package is designed to work alongside rule-based detection,
providing a hybrid approach to fraud detection.

Usage:
    from app.ml import MLConfig, load_model, predict_fraud

Author: Sentinel Team
Version: 0.1.0
"""

__version__ = "0.1.0"

# Package-level exports
__all__ = [
    "__version__",
]

# Note: We'll add more exports as we build ML components in future days
# Examples for later:
# from app.ml.config import MLConfig
# from app.ml.utils import load_model, predict_fraud
```

### 2. app/ml/config.py

ML configuration and constants.

```python
"""
Machine Learning Configuration

This module contains all configuration settings for ML models:
- Model hyperparameters
- Training configuration
- Feature engineering settings
- Evaluation metrics
- File paths

All settings can be overridden via environment variables.
"""

import os
from typing import Dict, Any, List
from pathlib import Path


# ============================================================================
# PATHS
# ============================================================================

# Base directory (project root)
BASE_DIR = Path(__file__).resolve().parent.parent.parent

# ML models directory
MODELS_DIR = BASE_DIR / "models"
MODELS_DIR.mkdir(exist_ok=True)

# Data directory (for training data, feature stores, etc.)
DATA_DIR = BASE_DIR / "data"
DATA_DIR.mkdir(exist_ok=True)

# Logs directory (for training logs, metrics, etc.)
LOGS_DIR = BASE_DIR / "logs" / "ml"
LOGS_DIR.mkdir(parents=True, exist_ok=True)


# ============================================================================
# MODEL CONFIGURATION
# ============================================================================

class MLConfig:
    """
    Machine Learning configuration settings.

    This class centralizes all ML-related configuration including:
    - XGBoost hyperparameters
    - Training settings
    - Evaluation metrics
    - Feature engineering settings
    """

    # ========================================================================
    # XGBoost Hyperparameters
    # ========================================================================

    XGBOOST_PARAMS: Dict[str, Any] = {
        # Tree parameters
        "max_depth": 6,
        # Maximum depth of a tree
        # Increasing this value will make the model more complex
        # Typical values: 3-10

        "min_child_weight": 1,
        # Minimum sum of instance weight needed in a child
        # Higher values prevent overfitting
        # Typical values: 1-10

        # Boosting parameters
        "learning_rate": 0.1,
        # Also called eta
        # Step size shrinkage to prevent overfitting
        # Typical values: 0.01-0.3

        "n_estimators": 100,
        # Number of boosting rounds
        # More rounds = better fit (until overfitting)
        # Typical values: 100-1000

        # Regularization
        "gamma": 0,
        # Minimum loss reduction required to make a split
        # Typical values: 0-5

        "reg_alpha": 0,
        # L1 regularization term on weights
        # Typical values: 0-1

        "reg_lambda": 1,
        # L2 regularization term on weights
        # Typical values: 1-10

        # Sampling
        "subsample": 0.8,
        # Subsample ratio of training instances
        # Typical values: 0.5-1.0

        "colsample_bytree": 0.8,
        # Subsample ratio of columns when constructing each tree
        # Typical values: 0.5-1.0

        # Class imbalance handling
        "scale_pos_weight": 1,
        # Balancing of positive and negative weights
        # For fraud: set to (# not fraud) / (# fraud)
        # Will be calculated automatically based on training data

        # Performance
        "n_jobs": -1,
        # Number of parallel threads (-1 = use all cores)

        "random_state": 42,
        # Random seed for reproducibility

        # Evaluation
        "eval_metric": "auc",
        # Metric for evaluation
        # Options: 'auc', 'logloss', 'error', 'map'
    }

    # ========================================================================
    # Training Configuration
    # ========================================================================

    # Train/test split ratio
    TRAIN_TEST_SPLIT = 0.8  # 80% training, 20% testing

    # Validation split for cross-validation
    CV_FOLDS = 5  # 5-fold cross-validation

    # Early stopping rounds (stop if no improvement)
    EARLY_STOPPING_ROUNDS = 50

    # Minimum samples required for training
    MIN_TRAINING_SAMPLES = 1000

    # Maximum samples to use (for faster training during development)
    MAX_TRAINING_SAMPLES = None  # None = use all data

    # Random state for reproducibility
    RANDOM_STATE = 42

    # ========================================================================
    # Feature Engineering
    # ========================================================================

    # Features to use for training
    FEATURE_COLUMNS: List[str] = [
        "amount",
        "transaction_count_1h",
        "transaction_count_24h",
        "avg_amount_1h",
        "avg_amount_24h",
        "location_change_km",
        "time_since_last_transaction",
        "hour_of_day",
        "day_of_week",
        "is_weekend",
    ]

    # Categorical features (will be encoded)
    CATEGORICAL_FEATURES: List[str] = [
        "day_of_week",
    ]

    # Numerical features (will be scaled)
    NUMERICAL_FEATURES: List[str] = [
        "amount",
        "transaction_count_1h",
        "transaction_count_24h",
        "avg_amount_1h",
        "avg_amount_24h",
        "location_change_km",
        "time_since_last_transaction",
        "hour_of_day",
    ]

    # Target variable
    TARGET_COLUMN = "is_fraud"

    # ========================================================================
    # Model Evaluation
    # ========================================================================

    # Classification threshold
    CLASSIFICATION_THRESHOLD = 0.5
    # Transactions with fraud probability > threshold are flagged as fraud

    # Evaluation metrics to track
    EVALUATION_METRICS = [
        "accuracy",
        "precision",
        "recall",
        "f1_score",
        "roc_auc",
        "confusion_matrix",
    ]

    # Minimum acceptable metrics for production
    MIN_ACCEPTABLE_METRICS = {
        "accuracy": 0.95,
        "precision": 0.80,
        "recall": 0.70,
        "roc_auc": 0.90,
    }

    # ========================================================================
    # Model Persistence
    # ========================================================================

    # Model file name format
    MODEL_FILENAME_FORMAT = "fraud_model_{timestamp}.pkl"

    # Keep only N most recent models
    MAX_MODELS_TO_KEEP = 5

    # Model metadata to save
    MODEL_METADATA = [
        "training_date",
        "training_samples",
        "feature_columns",
        "hyperparameters",
        "evaluation_metrics",
        "version",
    ]

    # ========================================================================
    # Logging and Monitoring
    # ========================================================================

    # Enable verbose training logs
    VERBOSE_TRAINING = True

    # Log frequency (every N iterations)
    LOG_FREQUENCY = 10

    # Enable model performance monitoring
    ENABLE_MONITORING = True

    # Alert thresholds for model degradation
    PERFORMANCE_DEGRADATION_THRESHOLD = 0.05  # 5% drop in performance

    # ========================================================================
    # Environment-Specific Overrides
    # ========================================================================

    @classmethod
    def get_config(cls, environment: str = None) -> Dict[str, Any]:
        """
        Get configuration for specific environment.

        Args:
            environment: Environment name (development, staging, production)
                        If None, reads from ENVIRONMENT env variable

        Returns:
            Dictionary of configuration settings
        """
        if environment is None:
            environment = os.getenv("ENVIRONMENT", "development")

        config = {
            "xgboost_params": cls.XGBOOST_PARAMS.copy(),
            "train_test_split": cls.TRAIN_TEST_SPLIT,
            "cv_folds": cls.CV_FOLDS,
            "early_stopping_rounds": cls.EARLY_STOPPING_ROUNDS,
            "feature_columns": cls.FEATURE_COLUMNS.copy(),
            "target_column": cls.TARGET_COLUMN,
            "classification_threshold": cls.CLASSIFICATION_THRESHOLD,
        }

        # Environment-specific adjustments
        if environment == "development":
            # Faster training for development
            config["xgboost_params"]["n_estimators"] = 50
            config["max_training_samples"] = 10000

        elif environment == "staging":
            # More thorough training for staging
            config["xgboost_params"]["n_estimators"] = 200
            config["early_stopping_rounds"] = 100

        elif environment == "production":
            # Full training for production
            config["xgboost_params"]["n_estimators"] = 500
            config["early_stopping_rounds"] = 100
            config["max_training_samples"] = None  # Use all data

        return config

    @classmethod
    def print_config(cls):
        """Print current configuration (useful for debugging)."""
        print("=" * 70)
        print("Machine Learning Configuration")
        print("=" * 70)
        print(f"\nModel: XGBoost")
        print(f"\nHyperparameters:")
        for key, value in cls.XGBOOST_PARAMS.items():
            print(f"  {key}: {value}")
        print(f"\nTraining Settings:")
        print(f"  Train/Test Split: {cls.TRAIN_TEST_SPLIT}")
        print(f"  CV Folds: {cls.CV_FOLDS}")
        print(f"  Early Stopping: {cls.EARLY_STOPPING_ROUNDS} rounds")
        print(f"\nFeatures: {len(cls.FEATURE_COLUMNS)}")
        for feature in cls.FEATURE_COLUMNS:
            print(f"  - {feature}")
        print(f"\nEvaluation Metrics:")
        for metric in cls.EVALUATION_METRICS:
            print(f"  - {metric}")
        print("=" * 70)


# ============================================================================
# CONVENIENCE FUNCTIONS
# ============================================================================

def get_model_path(model_name: str = None) -> Path:
    """
    Get path to model file.

    Args:
        model_name: Name of model file. If None, returns models directory.

    Returns:
        Path object
    """
    if model_name is None:
        return MODELS_DIR
    return MODELS_DIR / model_name


def get_data_path(data_name: str = None) -> Path:
    """
    Get path to data file.

    Args:
        data_name: Name of data file. If None, returns data directory.

    Returns:
        Path object
    """
    if data_name is None:
        return DATA_DIR
    return DATA_DIR / data_name


# ============================================================================
# MODULE TESTING
# ============================================================================

if __name__ == "__main__":
    """
    Test configuration module.

    Run this file directly to see configuration:
        python app/ml/config.py
    """
    print("\nTesting ML Configuration Module\n")

    # Print configuration
    MLConfig.print_config()

    # Test environment-specific configs
    print("\n\nEnvironment-Specific Configurations:")
    print("\n1. Development:")
    dev_config = MLConfig.get_config("development")
    print(f"   n_estimators: {dev_config['xgboost_params']['n_estimators']}")

    print("\n2. Staging:")
    staging_config = MLConfig.get_config("staging")
    print(f"   n_estimators: {staging_config['xgboost_params']['n_estimators']}")

    print("\n3. Production:")
    prod_config = MLConfig.get_config("production")
    print(f"   n_estimators: {prod_config['xgboost_params']['n_estimators']}")

    # Test path functions
    print("\n\nPath Configuration:")
    print(f"Models directory: {get_model_path()}")
    print(f"Data directory: {get_data_path()}")
    print(f"Logs directory: {LOGS_DIR}")

    print("\n✓ Configuration module test complete!\n")
```

### 3. app/ml/utils.py

Helper functions for ML tasks.

```python
"""
Machine Learning Utility Functions

This module provides helper functions for common ML tasks:
- Data loading and preprocessing
- Model saving and loading
- Feature extraction
- Prediction helpers
- Evaluation utilities

These utilities are designed to be reusable across different ML workflows.
"""

import pickle
import json
from datetime import datetime
from pathlib import Path
from typing import Dict, Any, Tuple, List, Optional
import numpy as np

from app.ml.config import MLConfig, get_model_path, LOGS_DIR


# ============================================================================
# MODEL PERSISTENCE
# ============================================================================

def save_model(
    model: Any,
    model_name: str = None,
    metadata: Dict[str, Any] = None
) -> Path:
    """
    Save trained model to disk with metadata.

    Args:
        model: Trained model object (XGBoost, scikit-learn, etc.)
        model_name: Name for model file. If None, generates timestamp-based name
        metadata: Dictionary of metadata to save with model

    Returns:
        Path to saved model file

    Example:
        >>> model = xgb.XGBClassifier()
        >>> model.fit(X_train, y_train)
        >>> path = save_model(model, metadata={'accuracy': 0.95})
        >>> print(f"Model saved to {path}")
    """
    # Generate model name if not provided
    if model_name is None:
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        model_name = f"fraud_model_{timestamp}.pkl"

    # Ensure .pkl extension
    if not model_name.endswith('.pkl'):
        model_name += '.pkl'

    # Get full path
    model_path = get_model_path(model_name)

    # Prepare metadata
    if metadata is None:
        metadata = {}

    metadata['saved_at'] = datetime.now().isoformat()
    metadata['model_type'] = type(model).__name__

    # Create model package
    model_package = {
        'model': model,
        'metadata': metadata,
    }

    # Save to disk
    with open(model_path, 'wb') as f:
        pickle.dump(model_package, f)

    print(f"✓ Model saved to: {model_path}")
    print(f"  Metadata: {metadata}")

    # Also save metadata as JSON for easy reading
    metadata_path = model_path.with_suffix('.json')
    with open(metadata_path, 'w') as f:
        json.dump(metadata, f, indent=2)

    return model_path


def load_model(model_name: str = None) -> Tuple[Any, Dict[str, Any]]:
    """
    Load trained model from disk.

    Args:
        model_name: Name of model file. If None, loads most recent model.

    Returns:
        Tuple of (model, metadata)

    Example:
        >>> model, metadata = load_model('fraud_model_20240115.pkl')
        >>> predictions = model.predict(X_test)
        >>> print(f"Model accuracy: {metadata['accuracy']}")
    """
    # If no name provided, find most recent model
    if model_name is None:
        models_dir = get_model_path()
        model_files = sorted(models_dir.glob('fraud_model_*.pkl'))

        if not model_files:
            raise FileNotFoundError(f"No models found in {models_dir}")

        model_path = model_files[-1]  # Most recent
        print(f"Loading most recent model: {model_path.name}")
    else:
        model_path = get_model_path(model_name)

    # Load model package
    with open(model_path, 'rb') as f:
        model_package = pickle.load(f)

    model = model_package['model']
    metadata = model_package.get('metadata', {})

    print(f"✓ Model loaded from: {model_path}")
    print(f"  Type: {type(model).__name__}")
    print(f"  Saved at: {metadata.get('saved_at', 'Unknown')}")

    return model, metadata


def list_models() -> List[Dict[str, Any]]:
    """
    List all saved models with metadata.

    Returns:
        List of dictionaries containing model information

    Example:
        >>> models = list_models()
        >>> for model in models:
        ...     print(f"{model['name']}: {model['accuracy']}")
    """
    models_dir = get_model_path()
    model_files = sorted(models_dir.glob('fraud_model_*.pkl'))

    models_info = []

    for model_path in model_files:
        # Try to load metadata from JSON file
        metadata_path = model_path.with_suffix('.json')

        if metadata_path.exists():
            with open(metadata_path, 'r') as f:
                metadata = json.load(f)
        else:
            metadata = {}

        models_info.append({
            'name': model_path.name,
            'path': str(model_path),
            'size_mb': model_path.stat().st_size / (1024 * 1024),
            'created': datetime.fromtimestamp(
                model_path.stat().st_ctime
            ).isoformat(),
            **metadata
        })

    return models_info


# ============================================================================
# DATA PREPROCESSING
# ============================================================================

def train_test_split_temporal(
    X: np.ndarray,
    y: np.ndarray,
    test_size: float = 0.2,
    time_column: Optional[np.ndarray] = None
) -> Tuple[np.ndarray, np.ndarray, np.ndarray, np.ndarray]:
    """
    Split data into train/test sets based on time.

    For fraud detection, we want to train on past data and test on future data
    (temporal split) rather than random split.

    Args:
        X: Feature matrix
        y: Labels
        test_size: Fraction of data for testing (default 0.2 = 20%)
        time_column: Column containing timestamps. If None, uses sequential order.

    Returns:
        Tuple of (X_train, X_test, y_train, y_test)

    Example:
        >>> X_train, X_test, y_train, y_test = train_test_split_temporal(X, y)
        >>> print(f"Training samples: {len(X_train)}")
        >>> print(f"Test samples: {len(X_test)}")
    """
    n_samples = len(X)
    split_index = int(n_samples * (1 - test_size))

    if time_column is not None:
        # Sort by time
        sort_indices = np.argsort(time_column)
        X = X[sort_indices]
        y = y[sort_indices]

    # Split
    X_train = X[:split_index]
    X_test = X[split_index:]
    y_train = y[:split_index]
    y_test = y[split_index:]

    print(f"Temporal train/test split:")
    print(f"  Training samples: {len(X_train)} ({len(X_train)/n_samples*100:.1f}%)")
    print(f"  Test samples: {len(X_test)} ({len(X_test)/n_samples*100:.1f}%)")

    return X_train, X_test, y_train, y_test


def calculate_class_weights(y: np.ndarray) -> Dict[int, float]:
    """
    Calculate class weights for imbalanced datasets.

    For fraud detection, we typically have:
    - 99.5% legitimate transactions (class 0)
    - 0.5% fraudulent transactions (class 1)

    Class weights help the model pay more attention to the minority class.

    Args:
        y: Array of labels (0 or 1)

    Returns:
        Dictionary mapping class labels to weights

    Example:
        >>> y = np.array([0, 0, 0, 0, 0, 0, 0, 0, 0, 1])  # 10% fraud
        >>> weights = calculate_class_weights(y)
        >>> print(weights)
        {0: 0.56, 1: 5.0}
    """
    unique, counts = np.unique(y, return_counts=True)
    total = len(y)

    weights = {}
    for label, count in zip(unique, counts):
        # Weight inversely proportional to class frequency
        weights[int(label)] = total / (len(unique) * count)

    print("Class distribution and weights:")
    for label, count in zip(unique, counts):
        pct = count / total * 100
        weight = weights[int(label)]
        print(f"  Class {label}: {count:,} samples ({pct:.2f}%) - weight: {weight:.2f}")

    return weights


# ============================================================================
# PREDICTION UTILITIES
# ============================================================================

def predict_with_threshold(
    model: Any,
    X: np.ndarray,
    threshold: float = 0.5
) -> Tuple[np.ndarray, np.ndarray]:
    """
    Make predictions with custom threshold.

    Args:
        model: Trained model
        X: Feature matrix
        threshold: Classification threshold (default 0.5)

    Returns:
        Tuple of (predictions, probabilities)
        - predictions: Binary predictions (0 or 1)
        - probabilities: Fraud probabilities (0.0 to 1.0)

    Example:
        >>> predictions, probabilities = predict_with_threshold(model, X_test, threshold=0.7)
        >>> print(f"Flagged {sum(predictions)} transactions as fraud")
    """
    # Get probabilities
    if hasattr(model, 'predict_proba'):
        probabilities = model.predict_proba(X)[:, 1]  # Probability of fraud (class 1)
    else:
        probabilities = model.predict(X)

    # Apply threshold
    predictions = (probabilities >= threshold).astype(int)

    return predictions, probabilities


def get_prediction_confidence(probabilities: np.ndarray) -> np.ndarray:
    """
    Calculate prediction confidence.

    Confidence is the distance from 0.5 threshold:
    - Probability 0.9 → Confidence 0.4 (high confidence fraud)
    - Probability 0.1 → Confidence 0.4 (high confidence not fraud)
    - Probability 0.5 → Confidence 0.0 (low confidence)

    Args:
        probabilities: Array of fraud probabilities

    Returns:
        Array of confidence scores (0.0 to 0.5)

    Example:
        >>> probs = np.array([0.9, 0.1, 0.5, 0.7])
        >>> confidence = get_prediction_confidence(probs)
        >>> print(confidence)
        [0.4, 0.4, 0.0, 0.2]
    """
    return np.abs(probabilities - 0.5)


# ============================================================================
# EVALUATION UTILITIES
# ============================================================================

def print_classification_report(
    y_true: np.ndarray,
    y_pred: np.ndarray,
    probabilities: Optional[np.ndarray] = None
):
    """
    Print detailed classification report.

    This is a simplified version. In Day 9, we'll use sklearn metrics.

    Args:
        y_true: True labels
        y_pred: Predicted labels
        probabilities: Predicted probabilities (optional)
    """
    # Calculate basic metrics
    tp = np.sum((y_true == 1) & (y_pred == 1))  # True positives
    fp = np.sum((y_true == 0) & (y_pred == 1))  # False positives
    tn = np.sum((y_true == 0) & (y_pred == 0))  # True negatives
    fn = np.sum((y_true == 1) & (y_pred == 0))  # False negatives

    # Calculate metrics
    accuracy = (tp + tn) / (tp + tn + fp + fn)
    precision = tp / (tp + fp) if (tp + fp) > 0 else 0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0

    # Print report
    print("=" * 60)
    print("CLASSIFICATION REPORT")
    print("=" * 60)
    print(f"\nConfusion Matrix:")
    print(f"                Predicted")
    print(f"              Not Fraud  Fraud")
    print(f"Actual Not    {tn:8d}  {fp:5d}")
    print(f"       Fraud  {fn:8d}  {tp:5d}")
    print(f"\nMetrics:")
    print(f"  Accuracy:  {accuracy:.4f}")
    print(f"  Precision: {precision:.4f}")
    print(f"  Recall:    {recall:.4f}")
    print(f"  F1-Score:  {f1:.4f}")
    print("=" * 60)


# ============================================================================
# LOGGING UTILITIES
# ============================================================================

def log_training_run(
    model_name: str,
    metrics: Dict[str, float],
    hyperparameters: Dict[str, Any]
):
    """
    Log training run details to file.

    Args:
        model_name: Name of trained model
        metrics: Dictionary of evaluation metrics
        hyperparameters: Dictionary of model hyperparameters
    """
    log_file = LOGS_DIR / "training_history.jsonl"

    log_entry = {
        'timestamp': datetime.now().isoformat(),
        'model_name': model_name,
        'metrics': metrics,
        'hyperparameters': hyperparameters,
    }

    # Append to log file (JSONL format - one JSON per line)
    with open(log_file, 'a') as f:
        f.write(json.dumps(log_entry) + '\n')

    print(f"✓ Training run logged to: {log_file}")


# ============================================================================
# MODULE TESTING
# ============================================================================

if __name__ == "__main__":
    """
    Test utility functions.

    Run this file directly to test utilities:
        python app/ml/utils.py
    """
    print("\nTesting ML Utilities Module\n")

    # Test data
    print("1. Testing train_test_split_temporal:")
    X = np.random.rand(100, 5)
    y = np.random.randint(0, 2, 100)
    X_train, X_test, y_train, y_test = train_test_split_temporal(X, y, test_size=0.2)

    print("\n2. Testing calculate_class_weights:")
    # Simulate imbalanced data: 95% class 0, 5% class 1
    y_imbalanced = np.array([0] * 95 + [1] * 5)
    weights = calculate_class_weights(y_imbalanced)

    print("\n3. Testing prediction utilities:")
    # Simulate probabilities
    probs = np.array([0.9, 0.1, 0.5, 0.7, 0.3])
    confidence = get_prediction_confidence(probs)
    print(f"Probabilities: {probs}")
    print(f"Confidence: {confidence}")

    print("\n4. Testing classification report:")
    y_true = np.array([0, 0, 0, 1, 1, 1, 0, 1])
    y_pred = np.array([0, 0, 1, 1, 1, 0, 0, 1])
    print_classification_report(y_true, y_pred)

    print("\n✓ Utilities module test complete!\n")
```

---

## Installation & Environment Setup

### Step 1: Update requirements.txt

Add ML packages to your requirements file:

```bash
cd /home/user/guide-sentinel

# Backup current requirements
cp requirements.txt requirements.txt.backup

# Add ML packages
cat >> requirements.txt << 'EOF'

# Day 8 - Machine Learning Packages
scikit-learn==1.3.2
xgboost==2.0.3
pandas==2.1.4
numpy==1.26.2
joblib==1.3.2
EOF
```

**What each package does:**

```
scikit-learn (sklearn)
  - Standard ML library for Python
  - Provides: train_test_split, metrics, preprocessing
  - Used by: Almost every ML project

xgboost
  - Extreme Gradient Boosting library
  - Our main algorithm for fraud detection
  - Why: Best performance on tabular data

pandas
  - Data manipulation and analysis
  - Used for: Loading, cleaning, transforming data
  - Essential for: Feature engineering

numpy
  - Numerical computing library
  - Foundation for: scikit-learn, XGBoost, pandas
  - Provides: Fast array operations

joblib
  - Model serialization (save/load models)
  - More efficient than pickle for large numpy arrays
  - Used for: Saving trained models
```

### Step 2: Install ML Packages

```bash
# Make sure virtual environment is activated
# You should see (venv) in your prompt

# Install new packages
pip install -r requirements.txt

# This will install:
# - scikit-learn and dependencies
# - xgboost
# - pandas
# - numpy (if not already installed)
# - joblib
#
# Installation time: 2-3 minutes
# Download size: ~150 MB
```

**Expected output:**
```
Collecting scikit-learn==1.3.2
  Downloading scikit_learn-1.3.2-cp39-cp39-linux_x86_64.whl (10.8 MB)
Collecting xgboost==2.0.3
  Downloading xgboost-2.0.3-py3-none-manylinux2014_x86_64.whl (297.6 MB)
Collecting pandas==2.1.4
  Downloading pandas-2.1.4-cp39-cp39-linux_x86_64.whl (12.3 MB)
...
Successfully installed joblib-1.3.2 numpy-1.26.2 pandas-2.1.4 scikit-learn-1.3.2 scipy-1.11.4 threadpoolctl-3.2.0 xgboost-2.0.3
```

### Step 3: Verify Installation

```bash
# Test scikit-learn
python -c "import sklearn; print(f'scikit-learn version: {sklearn.__version__}')"
# Expected: scikit-learn version: 1.3.2

# Test XGBoost
python -c "import xgboost; print(f'XGBoost version: {xgboost.__version__}')"
# Expected: XGBoost version: 2.0.3

# Test pandas
python -c "import pandas; print(f'pandas version: {pandas.__version__}')"
# Expected: pandas version: 2.1.4

# Test numpy
python -c "import numpy; print(f'numpy version: {numpy.__version__}')"
# Expected: numpy version: 1.26.2

# Test all imports together
python -c "import sklearn, xgboost, pandas, numpy, joblib; print('✓ All ML packages installed successfully')"
# Expected: ✓ All ML packages installed successfully
```

### Step 4: Create ML Package Structure

```bash
# Create ml directory
mkdir -p app/ml

# Create module files
touch app/ml/__init__.py
touch app/ml/config.py
touch app/ml/utils.py

# Create models directory (for saving trained models)
mkdir -p models

# Create data directory (for datasets)
mkdir -p data

# Create logs directory (for training logs)
mkdir -p logs/ml

# Verify structure
ls -la app/ml/
# Expected: __init__.py, config.py, utils.py

ls -la
# Expected: models/, data/, logs/ directories visible
```

### Step 5: Add Code to Files

Now copy the complete code from the "Complete Code Implementation" section above into each file:

1. Copy `app/ml/__init__.py` content
2. Copy `app/ml/config.py` content
3. Copy `app/ml/utils.py` content

### Step 6: Test ML Modules

```bash
# Test config module
python app/ml/config.py
# Expected: Prints ML configuration

# Test utils module
python app/ml/utils.py
# Expected: Runs utility function tests

# Test imports from main app
python -c "from app.ml.config import MLConfig; MLConfig.print_config()"
# Expected: Prints configuration
```

---

## Simple XGBoost Examples

### Example 1: Iris Classification (Classic ML Problem)

This example uses the famous Iris dataset to demonstrate XGBoost basics.

Create a new file: `test_xgboost_iris.py`

```python
"""
Simple XGBoost Example - Iris Classification

This script demonstrates XGBoost basics using the Iris dataset:
- 150 samples of iris flowers
- 4 features: sepal length, sepal width, petal length, petal width
- 3 classes: setosa, versicolor, virginica

This is NOT fraud detection - just learning XGBoost fundamentals.

Run with:
    python test_xgboost_iris.py
"""

import numpy as np
import xgboost as xgb
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, classification_report


# ============================================================================
# LOAD DATA
# ============================================================================

print("=" * 70)
print("XGBoost Example: Iris Classification")
print("=" * 70)

# Load iris dataset
iris = load_iris()
X = iris.data  # Features: sepal length, sepal width, petal length, petal width
y = iris.target  # Classes: 0=setosa, 1=versicolor, 2=virginica

print(f"\nDataset Info:")
print(f"  Samples: {len(X)}")
print(f"  Features: {X.shape[1]}")
print(f"  Classes: {len(np.unique(y))}")
print(f"\nFeature names:")
for i, name in enumerate(iris.feature_names):
    print(f"  {i}: {name}")


# ============================================================================
# TRAIN/TEST SPLIT
# ============================================================================

# Split: 80% training, 20% testing
X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,
    random_state=42
)

print(f"\nData Split:")
print(f"  Training samples: {len(X_train)}")
print(f"  Test samples: {len(X_test)}")


# ============================================================================
# TRAIN MODEL
# ============================================================================

print(f"\nTraining XGBoost model...")

# Create XGBoost classifier
model = xgb.XGBClassifier(
    max_depth=3,              # Shallow trees (simple dataset)
    learning_rate=0.1,        # Step size
    n_estimators=100,         # Number of trees
    objective='multi:softmax',  # Multi-class classification
    random_state=42,
    verbosity=0,              # Quiet training
)

# Train model
model.fit(X_train, y_train)

print(f"✓ Training complete!")


# ============================================================================
# MAKE PREDICTIONS
# ============================================================================

# Predict on test set
y_pred = model.predict(X_test)

# Calculate accuracy
accuracy = accuracy_score(y_test, y_pred)

print(f"\nResults:")
print(f"  Accuracy: {accuracy:.4f} ({accuracy*100:.2f}%)")


# ============================================================================
# DETAILED EVALUATION
# ============================================================================

print(f"\nDetailed Classification Report:")
print(classification_report(
    y_test,
    y_pred,
    target_names=iris.target_names
))


# ============================================================================
# FEATURE IMPORTANCE
# ============================================================================

print(f"\nFeature Importance:")
feature_importance = model.feature_importances_

for i, importance in enumerate(feature_importance):
    print(f"  {iris.feature_names[i]}: {importance:.4f}")

# Which feature matters most?
most_important_idx = np.argmax(feature_importance)
print(f"\nMost important feature: {iris.feature_names[most_important_idx]}")


# ============================================================================
# EXAMPLE PREDICTIONS
# ============================================================================

print(f"\nExample Predictions:")
print(f"{'Actual':<15} {'Predicted':<15} {'Match'}")
print("-" * 45)

for i in range(min(10, len(y_test))):
    actual = iris.target_names[y_test[i]]
    predicted = iris.target_names[y_pred[i]]
    match = "✓" if y_test[i] == y_pred[i] else "✗"
    print(f"{actual:<15} {predicted:<15} {match}")

print("=" * 70)
print("✓ Example complete!")
print("=" * 70)
```

**Run the example:**

```bash
python test_xgboost_iris.py
```

**Expected output:**
```
======================================================================
XGBoost Example: Iris Classification
======================================================================

Dataset Info:
  Samples: 150
  Features: 4
  Classes: 3

Feature names:
  0: sepal length (cm)
  1: sepal width (cm)
  2: petal length (cm)
  3: petal width (cm)

Data Split:
  Training samples: 120
  Test samples: 30

Training XGBoost model...
✓ Training complete!

Results:
  Accuracy: 1.0000 (100.00%)

Detailed Classification Report:
              precision    recall  f1-score   support

      setosa       1.00      1.00      1.00        10
  versicolor       1.00      1.00      1.00         9
   virginica       1.00      1.00      1.00        11

    accuracy                           1.00        30
   macro avg       1.00      1.00      1.00        30
weighted avg       1.00      1.00      1.00        30

Feature Importance:
  sepal length (cm): 0.0123
  sepal width (cm): 0.0147
  petal length (cm): 0.4859
  petal width (cm): 0.4871

Most important feature: petal width (cm)

Example Predictions:
Actual          Predicted       Match
---------------------------------------------
setosa          setosa          ✓
setosa          setosa          ✓
setosa          setosa          ✓
versicolor      versicolor      ✓
versicolor      versicolor      ✓
virginica       virginica       ✓
...
======================================================================
✓ Example complete!
======================================================================
```

### Example 2: Binary Classification with Synthetic Data

This example creates synthetic fraud-like data to demonstrate binary classification.

Create a new file: `test_xgboost_binary.py`

```python
"""
XGBoost Binary Classification Example

This demonstrates binary classification (0 or 1) using synthetic data
that mimics fraud detection patterns.

Run with:
    python test_xgboost_binary.py
"""

import numpy as np
import xgboost as xgb
from sklearn.model_selection import train_test_split
from sklearn.metrics import (
    accuracy_score,
    precision_score,
    recall_score,
    f1_score,
    confusion_matrix,
    roc_auc_score
)


# ============================================================================
# CREATE SYNTHETIC DATA
# ============================================================================

print("=" * 70)
print("XGBoost Binary Classification Example")
print("=" * 70)

# Set random seed for reproducibility
np.random.seed(42)

# Generate synthetic transaction-like data
n_samples = 1000

# Features:
# - amount: transaction amount ($0-$10000)
# - velocity: transactions in last hour (0-20)
# - hour: hour of day (0-23)
# - day: day of week (0-6)

amount = np.random.exponential(scale=100, size=n_samples)  # Most transactions small
velocity = np.random.poisson(lam=2, size=n_samples)  # Most users have low velocity
hour = np.random.randint(0, 24, size=n_samples)
day = np.random.randint(0, 7, size=n_samples)

# Combine features
X = np.column_stack([amount, velocity, hour, day])

# Create labels (fraud or not)
# Fraud if: high amount (>$1000) AND high velocity (>10) OR late night (hour 2-5)
fraud_rule = (
    ((amount > 1000) & (velocity > 10)) |  # High amount + high velocity
    ((amount > 2000) & (hour >= 2) & (hour <= 5))  # High amount + late night
)

# Add some noise (5% random flips)
noise = np.random.random(n_samples) < 0.05
y = (fraud_rule ^ noise).astype(int)  # XOR for random flips

print(f"\nSynthetic Dataset:")
print(f"  Total samples: {len(X)}")
print(f"  Features: {X.shape[1]}")
print(f"  Fraud samples: {sum(y)} ({sum(y)/len(y)*100:.2f}%)")
print(f"  Legitimate samples: {sum(y==0)} ({sum(y==0)/len(y)*100:.2f}%)")


# ============================================================================
# TRAIN/TEST SPLIT
# ============================================================================

X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,
    random_state=42,
    stratify=y  # Maintain class balance in split
)

print(f"\nData Split:")
print(f"  Training: {len(X_train)} samples")
print(f"  Test: {len(X_test)} samples")


# ============================================================================
# TRAIN MODEL
# ============================================================================

print(f"\nTraining XGBoost model...")

# Calculate scale_pos_weight for imbalanced data
scale_pos_weight = sum(y_train == 0) / sum(y_train == 1)
print(f"  scale_pos_weight: {scale_pos_weight:.2f}")

model = xgb.XGBClassifier(
    max_depth=4,
    learning_rate=0.1,
    n_estimators=100,
    objective='binary:logistic',  # Binary classification
    scale_pos_weight=scale_pos_weight,  # Handle imbalance
    random_state=42,
    verbosity=0,
)

model.fit(X_train, y_train)

print(f"✓ Training complete!")


# ============================================================================
# PREDICTIONS
# ============================================================================

# Predict probabilities
y_proba = model.predict_proba(X_test)[:, 1]  # Probability of fraud

# Predict classes (default threshold = 0.5)
y_pred = model.predict(X_test)


# ============================================================================
# EVALUATION
# ============================================================================

print(f"\nEvaluation Metrics:")

accuracy = accuracy_score(y_test, y_pred)
precision = precision_score(y_test, y_pred)
recall = recall_score(y_test, y_pred)
f1 = f1_score(y_test, y_pred)
auc = roc_auc_score(y_test, y_proba)

print(f"  Accuracy:  {accuracy:.4f}")
print(f"  Precision: {precision:.4f} (of flagged transactions, % truly fraud)")
print(f"  Recall:    {recall:.4f} (of actual fraud, % we caught)")
print(f"  F1-Score:  {f1:.4f} (harmonic mean of precision & recall)")
print(f"  ROC AUC:   {auc:.4f} (area under ROC curve)")

# Confusion matrix
cm = confusion_matrix(y_test, y_pred)
tn, fp, fn, tp = cm.ravel()

print(f"\nConfusion Matrix:")
print(f"                Predicted")
print(f"              Not Fraud  Fraud")
print(f"Actual Not    {tn:8d}  {fp:5d}  (True Neg, False Pos)")
print(f"       Fraud  {fn:8d}  {tp:5d}  (False Neg, True Pos)")

print(f"\nInterpretation:")
print(f"  True Positives (TP):  {tp} - Correctly caught fraud")
print(f"  False Positives (FP): {fp} - Incorrectly flagged legitimate")
print(f"  False Negatives (FN): {fn} - Missed fraud")
print(f"  True Negatives (TN):  {tn} - Correctly approved legitimate")


# ============================================================================
# FEATURE IMPORTANCE
# ============================================================================

feature_names = ['amount', 'velocity', 'hour', 'day']
print(f"\nFeature Importance:")

for name, importance in zip(feature_names, model.feature_importances_):
    print(f"  {name}: {importance:.4f}")


# ============================================================================
# EXAMPLE PREDICTIONS
# ============================================================================

print(f"\nExample Predictions (first 10 test samples):")
print(f"{'Actual':<10} {'Predicted':<10} {'Probability':<12} {'Match'}")
print("-" * 50)

for i in range(min(10, len(y_test))):
    actual = "Fraud" if y_test[i] == 1 else "Not Fraud"
    predicted = "Fraud" if y_pred[i] == 1 else "Not Fraud"
    probability = y_proba[i]
    match = "✓" if y_test[i] == y_pred[i] else "✗"

    print(f"{actual:<10} {predicted:<10} {probability:.4f}      {match}")

print("=" * 70)
print("✓ Binary classification example complete!")
print("=" * 70)
```

**Run the example:**

```bash
python test_xgboost_binary.py
```

---

## Understanding Model Evaluation

### Evaluation Metrics Explained

For fraud detection, we care about several metrics:

#### 1. Confusion Matrix

```
                    Predicted
              Not Fraud    Fraud
Actual Not       TN         FP      TN = True Negative (correct)
       Fraud     FN         TP      FP = False Positive (incorrect)
                                    FN = False Negative (incorrect)
                                    TP = True Positive (correct)
```

**Real-world implications:**

```
True Positive (TP): Caught actual fraud ✓
  - Good: Prevented fraud
  - Impact: Saved money, protected customer

False Positive (FP): Flagged legitimate transaction ✗
  - Bad: Annoyed customer
  - Impact: Customer calls support, possible lost sale

False Negative (FN): Missed actual fraud ✗✗
  - Very bad: Fraud succeeded
  - Impact: Lost money, potential chargebacks

True Negative (TN): Approved legitimate transaction ✓
  - Good: Smooth customer experience
  - Impact: Happy customer, completed sale
```

#### 2. Accuracy

```
Accuracy = (TP + TN) / (TP + TN + FP + FN)
```

**Interpretation:** Percentage of correct predictions (both fraud and not fraud)

**Problem for Fraud Detection:**
```
If only 0.5% of transactions are fraud:
- Predicting "not fraud" for everything → 99.5% accuracy!
- But we caught ZERO fraud!

Accuracy is MISLEADING for imbalanced datasets.
```

#### 3. Precision

```
Precision = TP / (TP + FP)
```

**Interpretation:** Of all transactions we flagged as fraud, what percentage were actually fraud?

**High precision means:** Few false alarms

**Trade-off:**
```
High precision (be very sure before flagging):
  - Flag only obvious fraud
  - Fewer false positives (happy customers)
  - But miss some fraud (more false negatives)

Example: Precision = 0.90
  - 90% of flagged transactions are truly fraud
  - 10% are false alarms
```

#### 4. Recall (Sensitivity)

```
Recall = TP / (TP + FN)
```

**Interpretation:** Of all actual fraud, what percentage did we catch?

**High recall means:** Catch most fraud

**Trade-off:**
```
High recall (catch all fraud):
  - Flag anything suspicious
  - Catch most fraud (fewer false negatives)
  - But many false alarms (more false positives)

Example: Recall = 0.95
  - We caught 95% of all fraud
  - 5% of fraud slipped through
```

#### 5. F1-Score

```
F1 = 2 * (Precision * Recall) / (Precision + Recall)
```

**Interpretation:** Harmonic mean of precision and recall

**Use case:** When you care equally about precision and recall

```
Good F1-score means:
  - Balanced performance
  - Not sacrificing one metric for the other

Example:
  Precision = 0.80, Recall = 0.90 → F1 = 0.85
  Precision = 0.95, Recall = 0.40 → F1 = 0.56 (unbalanced!)
```

#### 6. ROC Curve and AUC

**ROC (Receiver Operating Characteristic) Curve:**
```
Plots:
  X-axis: False Positive Rate (FPR)
  Y-axis: True Positive Rate (TPR = Recall)

Shows trade-off between catching fraud (TPR) and false alarms (FPR)
```

**AUC (Area Under Curve):**
```
Range: 0.5 to 1.0
- 0.5 = Random guessing
- 0.7 = Fair model
- 0.8 = Good model
- 0.9 = Excellent model
- 1.0 = Perfect model (unrealistic)

For fraud detection:
  AUC > 0.85 is considered good
  AUC > 0.90 is excellent
```

### Choosing the Right Metric

Different business goals require different metrics:

**Scenario 1: Low-Value Transactions (e-commerce)**
```
Goal: Minimize false positives (don't annoy customers)
Metric: Maximize precision
Threshold: High (0.7-0.9)
Result: Only flag very suspicious transactions
```

**Scenario 2: High-Value Transactions (banking)**
```
Goal: Catch all fraud (money is important)
Metric: Maximize recall
Threshold: Low (0.3-0.5)
Result: Flag anything remotely suspicious, manual review
```

**Scenario 3: Balanced Approach**
```
Goal: Balance customer experience and fraud prevention
Metric: Maximize F1-score or AUC
Threshold: Medium (0.5)
Result: Reasonable trade-off
```

### Threshold Tuning

The classification threshold (default 0.5) can be adjusted:

```python
# Model predicts probabilities
probability = model.predict_proba(X)[:, 1]

# Different thresholds
conservative = probability > 0.8  # Few flags, high precision
balanced = probability > 0.5      # Default
aggressive = probability > 0.3    # Many flags, high recall

# Business decision:
# "We can manually review 100 transactions per day"
# → Set threshold such that ~100 transactions flagged
```

**Threshold Impact:**

```
Threshold = 0.3 (Low):
  Precision: 0.60 (many false positives)
  Recall: 0.95 (catch almost all fraud)
  Use when: Fraud is very costly

Threshold = 0.5 (Medium):
  Precision: 0.80 (balanced)
  Recall: 0.85 (balanced)
  Use when: Balanced approach

Threshold = 0.7 (High):
  Precision: 0.92 (few false positives)
  Recall: 0.65 (miss some fraud)
  Use when: Customer experience is priority
```

---

## Testing & Verification

### Test 1: Verify ML Package Import

```bash
# Test basic imports
python -c "
from app.ml.config import MLConfig
from app.ml.utils import save_model, load_model

print('✓ ML package imports successful')
"
```

### Test 2: Test Configuration

```bash
# Print configuration
python -c "
from app.ml.config import MLConfig

MLConfig.print_config()
"
```

### Test 3: Run Iris Example

```bash
# Run iris classification example
python test_xgboost_iris.py

# Should see:
# - Dataset info
# - Training progress
# - Accuracy close to 100%
# - Feature importance
```

### Test 4: Run Binary Classification Example

```bash
# Run binary classification example
python test_xgboost_binary.py

# Should see:
# - Synthetic data generation
# - Training with imbalanced data
# - Evaluation metrics
# - Confusion matrix
```

### Test 5: Verify Package Versions

```bash
# Create version check script
cat > check_ml_versions.py << 'EOF'
"""Check ML package versions."""

import sys

packages = {
    'scikit-learn': '1.3.2',
    'xgboost': '2.0.3',
    'pandas': '2.1.4',
    'numpy': '1.26.2',
    'joblib': '1.3.2',
}

print("Checking ML package versions:\n")

all_good = True
for package_name, expected_version in packages.items():
    try:
        if package_name == 'scikit-learn':
            import sklearn
            actual_version = sklearn.__version__
            import_name = 'sklearn'
        else:
            module = __import__(package_name.replace('-', '_'))
            actual_version = module.__version__
            import_name = package_name

        status = "✓" if actual_version == expected_version else "✗"
        print(f"{status} {package_name}: {actual_version} (expected {expected_version})")

        if actual_version != expected_version:
            all_good = False

    except ImportError as e:
        print(f"✗ {package_name}: NOT INSTALLED")
        all_good = False

print()
if all_good:
    print("✓ All ML packages installed correctly!")
    sys.exit(0)
else:
    print("✗ Some packages have issues")
    sys.exit(1)
EOF

python check_ml_versions.py
```

---

## Next Steps

### What You Accomplished Today

Congratulations! You've completed Day 8. Here's what you learned:

**✓ Machine Learning Fundamentals:**
- Supervised vs unsupervised vs reinforcement learning
- Classification vs regression
- Features vs labels
- Training vs inference
- Overfitting vs underfitting

**✓ XGBoost Deep Dive:**
- Decision trees and ensemble methods
- Gradient boosting algorithm
- XGBoost optimizations
- Hyperparameter tuning
- When to use XGBoost

**✓ Project Setup:**
- Created `app/ml/` package
- Installed ML packages (scikit-learn, XGBoost, pandas)
- Set up configuration system
- Built utility functions
- Created model storage structure

**✓ Practical Examples:**
- Iris classification (multi-class)
- Binary classification (fraud-like)
- Feature importance
- Model evaluation metrics

**✓ Evaluation Metrics:**
- Confusion matrix
- Precision, recall, F1-score
- ROC curve and AUC
- Threshold tuning

### Tomorrow (Day 9): ML Training for Fraud Detection

**On Day 9, you'll:**
- Feature engineering for fraud detection
- Extract features from transaction database
- Handle imbalanced datasets
- Train XGBoost model on real fraud data
- Evaluate model performance
- Save and load trained models
- Create prediction API endpoint

**Preparation:**

1. **Review your transaction data:**
   ```bash
   # Connect to database and check transaction count
   # Make sure you have transaction data from Day 2
   ```

2. **Understand your data:**
   - How many transactions do you have?
   - What's the fraud rate?
   - What features are available?

3. **Read about feature engineering:**
   - Velocity features (transaction count over time)
   - Aggregation features (average amount per user)
   - Time-based features (hour, day of week)
   - Location features (distance traveled)

### Additional Learning Resources

**Books:**
- "Hands-On Machine Learning" by Aurélien Géron
- "The Hundred-Page Machine Learning Book" by Andriy Burkov

**Courses:**
- Fast.ai: Practical Deep Learning (free)
- Coursera: Machine Learning by Andrew Ng

**XGBoost:**
- Official docs: https://xgboost.readthedocs.io/
- Parameters guide: https://xgboost.readthedocs.io/en/stable/parameter.html

**Kaggle:**
- Practice on fraud detection datasets
- Learn from competition winners
- Kaggle Learn: Free micro-courses

### Practice Exercises

1. **Modify the iris example:**
   - Try different max_depth values (2, 4, 6, 8)
   - Observe how accuracy changes
   - Plot feature importance

2. **Improve binary classification:**
   - Add more features to synthetic data
   - Try different scale_pos_weight values
   - Experiment with threshold tuning

3. **Create your own dataset:**
   - Generate synthetic fraud data
   - Train XGBoost model
   - Evaluate with different metrics

4. **Explore XGBoost parameters:**
   - Change learning_rate (0.01, 0.1, 0.3)
   - Change n_estimators (50, 100, 500)
   - Observe training time and accuracy

---

## Troubleshooting

### Issue 1: XGBoost Import Error

**Error:**
```
ImportError: cannot import name 'XGBClassifier' from 'xgboost'
```

**Solution:**
```bash
# Reinstall XGBoost
pip uninstall xgboost
pip install xgboost==2.0.3

# Verify
python -c "import xgboost; print(xgboost.__version__)"
```

### Issue 2: Scikit-learn Version Mismatch

**Error:**
```
AttributeError: module 'sklearn' has no attribute 'XXX'
```

**Solution:**
```bash
# Check version
python -c "import sklearn; print(sklearn.__version__)"

# If wrong version, reinstall
pip install --upgrade scikit-learn==1.3.2
```

### Issue 3: NumPy Version Conflict

**Error:**
```
ValueError: numpy.ndarray size changed
```

**Solution:**
```bash
# Reinstall all packages
pip uninstall numpy pandas scikit-learn xgboost -y
pip install numpy==1.26.2
pip install pandas==2.1.4
pip install scikit-learn==1.3.2
pip install xgboost==2.0.3
```

### Issue 4: Model Save/Load Errors

**Error:**
```
FileNotFoundError: models/ directory not found
```

**Solution:**
```bash
# Create missing directories
mkdir -p models data logs/ml

# Verify
ls -la models/
```

---

**Navigation:** [← Previous: Day 7](README-DAY-007.md) | [Main Guide](README.md) | [Next: Day 9 →](README-DAY-009.md)

---

**Last Updated:** 2024-01-20
**Author:** Sentinel Team
**Version:** 1.0.0
**Estimated Completion Time:** 3-4 hours

---

*End of Day 8 Guide*
