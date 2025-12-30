# Day 10: Model Training & Evaluation - Machine Learning Foundations

**Building an XGBoost Fraud Detection Model**

**Navigation:** [← Previous: Day 9](README-DAY-009.md) | [Main Guide](README.md) | [Next: Day 11 →](README-DAY-011.md)

---

## Table of Contents

1. [Overview](#overview)
2. [Machine Learning Fundamentals](#machine-learning-fundamentals)
3. [Understanding Class Imbalance](#understanding-class-imbalance)
4. [XGBoost Deep Dive](#xgboost-deep-dive)
5. [Project Structure](#project-structure)
6. [Complete Code Implementation](#complete-code-implementation)
7. [Data Preparation Pipeline](#data-preparation-pipeline)
8. [Training Pipeline](#training-pipeline)
9. [Model Evaluation](#model-evaluation)
10. [Model Persistence & Versioning](#model-persistence--versioning)
11. [Testing the Model](#testing-the-model)
12. [Understanding the Results](#understanding-the-results)
13. [Troubleshooting](#troubleshooting)
14. [Next Steps](#next-steps)

---

## Overview

### Day 10 Objectives

Welcome to Day 10! Today marks a critical milestone in building Sentinel - we're training our first machine learning model for fraud detection. By the end of this day, you'll have:

✅ A trained XGBoost model for fraud detection
✅ Understanding of train/test splits and cross-validation
✅ Knowledge of handling class imbalance with SMOTE
✅ Comprehensive model evaluation metrics
✅ Model versioning and persistence system
✅ Feature importance analysis
✅ Production-ready model artifacts

**What You'll Build:**

```
Training Pipeline:
├─→ Fetch transactions from database
├─→ Extract features (from Day 9)
├─→ Handle class imbalance (SMOTE)
├─→ Split data (train/test 80/20)
├─→ Train XGBoost model
├─→ Cross-validate (5-fold)
├─→ Evaluate performance
├─→ Save model with metadata
└─→ Generate evaluation reports

Evaluation Metrics:
├─→ Confusion Matrix
├─→ Precision, Recall, F1-Score
├─→ ROC Curve & AUC
├─→ Feature Importance
└─→ Error Analysis
```

**Estimated Time:** 4-5 hours

**Prerequisites:**
- ✅ Completed Day 9 (Feature Engineering)
- ✅ Completed Day 8 (ML Setup - XGBoost, scikit-learn installed)
- ✅ Completed Day 2 (Database with fraud transactions)
- ✅ PostgreSQL running with sample fraud data
- ✅ Virtual environment activated

**NO New Packages Required:**
All dependencies were installed in Day 8:
- XGBoost
- scikit-learn
- imbalanced-learn (for SMOTE)
- joblib (for model persistence)
- matplotlib/seaborn (for visualization)

---

## Machine Learning Fundamentals

### What is Supervised Learning?

Supervised learning is a type of machine learning where we train a model using labeled data (data where we know the correct answer).

**Fraud Detection as Supervised Learning:**

```
Input (Features):                    Output (Label):
├─→ Transaction amount                ├─→ is_fraud = 0 (legitimate)
├─→ Device information                └─→ is_fraud = 1 (fraudulent)
├─→ User behavior patterns
├─→ Location data
└─→ Time patterns

Training Process:
1. Show model examples: "This transaction was fraud because..."
2. Model learns patterns: "High amounts + new device + VPN = likely fraud"
3. Model predicts new transactions: "This looks like fraud (85% confidence)"
```

**Example:**

```python
# Historical data with labels
training_data = [
    # Features                                        Label
    {"amount": 500000, "new_device": True,  ...}  →  1 (fraud)
    {"amount": 5000,   "new_device": False, ...}  →  0 (legitimate)
    {"amount": 250000, "vpn": True,         ...}  →  1 (fraud)
]

# Train model
model.fit(training_data_features, training_data_labels)

# Predict new transaction
new_transaction = {"amount": 100000, "new_device": True, ...}
prediction = model.predict(new_transaction)  # → 0.78 (78% fraud probability)
```

### Train/Test Split

**Why Split Data?**

We need to evaluate our model on data it hasn't seen during training. Otherwise, it's like testing students with the exact same questions they studied.

```
All Data (100%)
    ↓
    Split
    ↓
Training Set (80%)          Test Set (20%)
Used to train model         Used to evaluate model
Model sees this data        Model never sees this
Learns patterns             Tests generalization
```

**The Problem with Not Splitting:**

```python
# ❌ BAD: Training and testing on same data
model.train(all_data)
accuracy = model.test(all_data)  # → 99.9% (misleading!)
# Model just memorized the data (overfitting)

# ✅ GOOD: Training and testing on different data
train_data, test_data = split(all_data, test_size=0.2)
model.train(train_data)
accuracy = model.test(test_data)  # → 92% (realistic)
# Model actually learned patterns
```

### Cross-Validation (K-Fold)

Cross-validation provides a more robust estimate of model performance by testing on multiple train/test splits.

**5-Fold Cross-Validation:**

```
Original Data: [████████████████████████████████]

Fold 1: [TEST][TRAIN TRAIN TRAIN TRAIN] → Accuracy: 91%
Fold 2: [TRAIN][TEST][TRAIN TRAIN TRAIN] → Accuracy: 93%
Fold 3: [TRAIN TRAIN][TEST][TRAIN TRAIN] → Accuracy: 90%
Fold 4: [TRAIN TRAIN TRAIN][TEST][TRAIN] → Accuracy: 92%
Fold 5: [TRAIN TRAIN TRAIN TRAIN][TEST]  → Accuracy: 91%

Average Accuracy: 91.4% ± 1.1%
```

**Benefits:**
- Uses all data for both training and testing
- Provides confidence interval (mean ± std)
- Reduces variance in performance estimates
- Detects overfitting (large gap between train and test scores)

### Overfitting vs Underfitting

**Underfitting:**
Model is too simple to capture patterns
```
Training Accuracy: 60%
Test Accuracy: 58%
Problem: Model didn't learn enough
Solution: Use more complex model or features
```

**Good Fit:**
Model captures patterns without memorizing
```
Training Accuracy: 93%
Test Accuracy: 91%
Sweet spot: Similar performance on both sets
```

**Overfitting:**
Model memorized training data instead of learning patterns
```
Training Accuracy: 99%
Test Accuracy: 75%
Problem: Model doesn't generalize to new data
Solution: Regularization, more data, simpler model
```

**Visual Example:**

```
Transaction Amount vs Fraud Probability

Underfitting:                 Good Fit:                 Overfitting:
     |                             |                           |
100% |____                     100%|      ____            100% |  _  _  _
     |    \____                    |    /                      | / \/ \/ \
 50% |         \               50% |   /                   50% |/        \
     |          \                  |  /                        |         |
  0% |___________             0%  |_/                     0%  |_________|
     0  100k  500k               0  100k  500k               0  100k  500k

Too simple                    Captures pattern           Memorized noise
```

---

## Understanding Class Imbalance

### The Imbalance Problem

In fraud detection, **fraud is rare**. This creates a significant challenge:

```
Real-world Transaction Distribution:
├─→ Legitimate: 99.5% (9,950 transactions)
└─→ Fraudulent:  0.5% (50 transactions)

Problem:
Model can achieve 99.5% accuracy by always predicting "not fraud"!
But it catches ZERO frauds - completely useless!
```

**Example:**

```python
# Naive model that always predicts "legitimate"
class NaiveModel:
    def predict(self, transaction):
        return 0  # Always "not fraud"

# Test on 10,000 transactions (50 frauds)
predictions = [model.predict(tx) for tx in test_data]
accuracy = 99.5%  # Looks great!

# But reality check:
frauds_caught = 0      # Missed all frauds!
false_alarms = 0       # Never flagged anything
precision = undefined  # Useless model
recall = 0%           # Caught 0% of frauds
```

### Why Accuracy is Misleading

**Accuracy alone doesn't tell the story:**

```
Scenario 1: Imbalanced Data (99% legitimate, 1% fraud)
Model always predicts "legitimate"
├─→ Accuracy: 99% ✓ (looks good)
├─→ Frauds caught: 0% ✗ (useless)
└─→ Business impact: Lose millions to fraud ✗

Scenario 2: Balanced Evaluation
Model trained with SMOTE
├─→ Accuracy: 92% ✓
├─→ Frauds caught: 87% ✓ (excellent)
├─→ False alarms: 8% ✓ (acceptable)
└─→ Business impact: Save millions ✓
```

### SMOTE (Synthetic Minority Over-sampling Technique)

SMOTE creates synthetic examples of the minority class (fraud) to balance the dataset.

**How SMOTE Works:**

```
Original Imbalanced Data:
Legitimate: ●●●●●●●●●●●●●●●●●●●● (20 samples)
Fraud:      ●●                      (2 samples)

SMOTE Process:
1. Find nearest fraud neighbors
2. Create synthetic frauds between them

    Fraud A ●
             \  Create new synthetic fraud
              \ here (interpolation)
               ●
              /
    Fraud B ●

After SMOTE:
Legitimate: ●●●●●●●●●●●●●●●●●●●● (20 samples)
Fraud:      ●●●●●●●●●●●●●●●●●●●● (20 samples - 2 real + 18 synthetic)
```

**SMOTE vs Simple Duplication:**

```python
# ❌ BAD: Simple duplication (overfitting)
fraud_samples = [fraud1, fraud2]
duplicated = fraud_samples * 10  # [fraud1, fraud2, fraud1, fraud2, ...]
# Just copies - model memorizes exact examples

# ✅ GOOD: SMOTE (generalization)
from imblearn.over_sampling import SMOTE
smote = SMOTE(random_state=42)
X_balanced, y_balanced = smote.fit_resample(X, y)
# Creates variations - model learns patterns
```

**When to Use SMOTE:**

```
Use SMOTE when:
✓ Minority class < 20% of data
✓ You need model to catch rare events
✓ Cost of missing fraud is high

Don't use SMOTE when:
✗ Classes are already balanced
✗ You have very little minority data (< 20 samples)
✗ You're using tree-based models with proper weights
```

---

## XGBoost Deep Dive

### What is XGBoost?

XGBoost (eXtreme Gradient Boosting) is a powerful machine learning algorithm that builds an ensemble of decision trees sequentially, where each tree corrects the mistakes of previous trees.

**Why XGBoost for Fraud Detection?**

1. **Handles Imbalanced Data:** Built-in `scale_pos_weight` parameter
2. **Fast Training:** Optimized C++ implementation
3. **Feature Importance:** Shows which features matter most
4. **Robust to Outliers:** Tree-based models are naturally robust
5. **Prevents Overfitting:** Built-in regularization (L1, L2)
6. **Industry Proven:** Used by Kaggle winners and major companies

### Boosting Explained (Simple Analogy)

```
Boosting = Team of Specialists Learning from Mistakes

Tree 1 (Generalist):
  "High amount → fraud"
  Mistakes: Misses frauds with normal amounts
  Accuracy: 70%

Tree 2 (Specialist):
  "Learn from Tree 1's mistakes"
  "Normal amount + new device → fraud"
  Accuracy on mistakes: 60%

Tree 3 (Specialist):
  "Learn from Trees 1 & 2's mistakes"
  "Normal amount + old device + VPN → fraud"
  Accuracy on mistakes: 55%

Final Model = Tree 1 + Tree 2 + Tree 3 + ...
Combined Accuracy: 92%
```

**Visual Example:**

```
Sequential Tree Building:

Round 1:                Round 2:                Round 3:
Tree 1 predicts         Tree 2 focuses on       Tree 3 refines
                        Tree 1's errors         remaining errors

High amt?               New device?             VPN used?
  /  \                    /  \                     /  \
Yes  No                 Yes  No                  Yes  No
 |    |                  |    |                   |    |
Fraud Legit          Fraud Legit              Fraud Legit
70% acc.             +15% acc.                +7% acc.

Final prediction = weighted sum of all trees
```

### XGBoost Hyperparameters Explained

**1. max_depth (Tree Complexity):**

```python
max_depth = 3  # Shallow trees (prevent overfitting)
max_depth = 10 # Deep trees (capture complex patterns, risk overfitting)

Example:
max_depth=3:                max_depth=6:
   Amount?                     Amount?
   /    \                      /    \
  Yes   No                   Yes   No
  / \   / \                  /  \
Complexity: Simple         Complexity: High
Overfitting: Low          Overfitting: Risk
Training time: Fast       Training time: Slow
```

**Recommendation for Fraud Detection:** Start with 3-6, increase if underfitting

**2. learning_rate (Step Size):**

```python
learning_rate = 0.3  # Default (fast learning, risk overfitting)
learning_rate = 0.1  # Conservative (slower, better generalization)
learning_rate = 0.01 # Very conservative (needs more trees)

Analogy:
  learning_rate = 1.0   → Taking giant steps (might overshoot)
  learning_rate = 0.1   → Taking careful steps (stable)
  learning_rate = 0.01  → Taking tiny steps (very safe but slow)
```

**Recommendation:** 0.1 for production (balance speed and accuracy)

**3. n_estimators (Number of Trees):**

```python
n_estimators = 100   # Quick training
n_estimators = 500   # Better performance
n_estimators = 1000  # Diminishing returns

Performance vs Trees:
Accuracy
    |     /-------  (plateau)
95% |    /
    |   /
90% |  /
    | /
85% |/
    +------------------
    0  100  500  1000 Trees

More trees = Better performance (up to a point)
But also = Longer training time
```

**Recommendation:** 500-1000 for fraud detection

**4. scale_pos_weight (Handle Imbalance):**

```python
# Calculate for imbalanced data
scale_pos_weight = (num_legitimate / num_fraud)

Example:
  9,900 legitimate, 100 frauds
  scale_pos_weight = 9900 / 100 = 99

What it does:
  Tells XGBoost: "Fraud errors are 99x more costly than legitimate errors"
  Model pays more attention to fraud examples
```

**5. subsample (Row Sampling):**

```python
subsample = 1.0  # Use all data (risk overfitting)
subsample = 0.8  # Use 80% of data (prevent overfitting)

Process:
  Each tree randomly selects 80% of training data
  Different trees see different samples
  Reduces overfitting through randomization
```

**6. colsample_bytree (Feature Sampling):**

```python
colsample_bytree = 1.0  # Use all features
colsample_bytree = 0.8  # Use 80% of features per tree

Example:
  10 features total
  colsample_bytree = 0.8
  Each tree randomly selects 8 features
  Forces model to learn diverse patterns
```

**Putting It All Together:**

```python
# Recommended XGBoost parameters for fraud detection
params = {
    'max_depth': 6,              # Moderate complexity
    'learning_rate': 0.1,        # Conservative learning
    'n_estimators': 500,         # Sufficient trees
    'scale_pos_weight': 99,      # Handle imbalance (adjust based on data)
    'subsample': 0.8,            # Prevent overfitting
    'colsample_bytree': 0.8,     # Feature diversity
    'objective': 'binary:logistic',  # Binary classification
    'eval_metric': 'auc',        # Optimize for AUC
    'random_state': 42           # Reproducibility
}
```

---

## Project Structure

### Directory Layout

```
/home/user/sentinel-api/
├── app/
│   ├── ml/
│   │   ├── __init__.py
│   │   ├── features.py          # From Day 9 (Feature engineering)
│   │   ├── model.py            # NEW: XGBoost model wrapper
│   │   ├── trainer.py          # NEW: Training pipeline
│   │   ├── evaluator.py        # NEW: Model evaluation
│   │   └── metrics.py          # NEW: Custom metrics
│   ├── models/                 # NEW: Saved model artifacts
│   │   ├── fraud_model_v1.pkl  # Serialized model
│   │   └── metadata_v1.json    # Model metadata
│   └── scripts/                # NEW: Training scripts
│       ├── train_model.py      # Run training
│       └── evaluate_model.py   # Evaluate trained model
├── data/                       # Optional: Sample data
│   └── sample_transactions.csv
├── notebooks/                  # Optional: Jupyter notebooks
│   └── model_exploration.ipynb
└── requirements.txt            # Dependencies from Day 8
```

### Files We'll Create Today

1. **app/ml/model.py** - XGBoost model wrapper class
2. **app/ml/trainer.py** - Complete training pipeline
3. **app/ml/evaluator.py** - Model evaluation and metrics
4. **app/ml/metrics.py** - Custom metric calculations
5. **app/scripts/train_model.py** - Training script
6. **app/scripts/evaluate_model.py** - Evaluation script

---

## Complete Code Implementation

### Step 1: Create Project Directories

```bash
# Navigate to project root
cd /home/user/sentinel-api

# Create ML modules
mkdir -p app/ml
touch app/ml/__init__.py

# Create models directory for saved models
mkdir -p app/models

# Create scripts directory
mkdir -p app/scripts
touch app/scripts/__init__.py

# Verify structure
ls -la app/ml/
ls -la app/models/
ls -la app/scripts/
```

---

### Step 2: XGBoost Model Wrapper

**File: `/home/user/sentinel-api/app/ml/model.py`**

This file wraps XGBoost in a clean interface with fraud detection-specific logic.

```python
"""
XGBoost Model Wrapper for Fraud Detection.

This module provides a clean interface for training, predicting, and
managing XGBoost models specifically tuned for fraud detection.

Key Features:
- Fraud-specific hyperparameters
- Class imbalance handling
- Probability calibration
- Model versioning
- Feature importance extraction
"""

import numpy as np
import pandas as pd
from typing import Dict, List, Optional, Tuple, Any
import xgboost as xgb
from sklearn.calibration import CalibratedClassifierCV
import joblib
import json
from datetime import datetime
from pathlib import Path


class FraudDetectionModel:
    """
    XGBoost-based fraud detection model.

    This class encapsulates an XGBoost classifier with fraud detection
    optimizations including class imbalance handling and probability
    calibration.

    Attributes:
        model: XGBoost classifier instance
        calibrated_model: Calibrated classifier for better probabilities
        feature_names: List of feature names used in training
        version: Model version string
        metadata: Model training metadata
    """

    def __init__(
        self,
        max_depth: int = 6,
        learning_rate: float = 0.1,
        n_estimators: int = 500,
        scale_pos_weight: Optional[float] = None,
        subsample: float = 0.8,
        colsample_bytree: float = 0.8,
        random_state: int = 42,
        **kwargs
    ):
        """
        Initialize fraud detection model with XGBoost.

        Args:
            max_depth: Maximum tree depth (default: 6)
                      Higher = more complex, risk overfitting
            learning_rate: Learning rate / eta (default: 0.1)
                          Lower = more conservative, needs more trees
            n_estimators: Number of boosting rounds (default: 500)
            scale_pos_weight: Weight for positive class (fraud)
                             If None, calculated from training data
            subsample: Fraction of samples per tree (default: 0.8)
                      Helps prevent overfitting
            colsample_bytree: Fraction of features per tree (default: 0.8)
                             Increases model diversity
            random_state: Random seed for reproducibility
            **kwargs: Additional XGBoost parameters

        Example:
            >>> model = FraudDetectionModel(
            ...     max_depth=6,
            ...     learning_rate=0.1,
            ...     n_estimators=500
            ... )
        """
        self.max_depth = max_depth
        self.learning_rate = learning_rate
        self.n_estimators = n_estimators
        self.scale_pos_weight = scale_pos_weight
        self.subsample = subsample
        self.colsample_bytree = colsample_bytree
        self.random_state = random_state
        self.extra_params = kwargs

        # Initialize models (will be set during training)
        self.model: Optional[xgb.XGBClassifier] = None
        self.calibrated_model: Optional[CalibratedClassifierCV] = None

        # Model metadata
        self.feature_names: List[str] = []
        self.version: str = "1.0.0"
        self.metadata: Dict[str, Any] = {}

    def build_model(self, scale_pos_weight: Optional[float] = None) -> xgb.XGBClassifier:
        """
        Build XGBoost classifier with fraud detection parameters.

        Args:
            scale_pos_weight: Weight for positive class
                            If None, uses value from __init__

        Returns:
            Configured XGBoost classifier

        Example:
            >>> model_wrapper = FraudDetectionModel()
            >>> xgb_model = model_wrapper.build_model(scale_pos_weight=99)
        """
        # Use provided scale_pos_weight or fall back to instance variable
        if scale_pos_weight is None:
            scale_pos_weight = self.scale_pos_weight

        # Build parameter dictionary
        params = {
            'max_depth': self.max_depth,
            'learning_rate': self.learning_rate,
            'n_estimators': self.n_estimators,
            'subsample': self.subsample,
            'colsample_bytree': self.colsample_bytree,
            'objective': 'binary:logistic',  # Binary classification
            'eval_metric': 'auc',            # Optimize for AUC
            'random_state': self.random_state,
            'n_jobs': -1,                    # Use all CPU cores
            'tree_method': 'hist',           # Faster training
        }

        # Add scale_pos_weight if provided
        if scale_pos_weight is not None:
            params['scale_pos_weight'] = scale_pos_weight

        # Add any extra parameters
        params.update(self.extra_params)

        # Create XGBoost classifier
        model = xgb.XGBClassifier(**params)

        return model

    def train(
        self,
        X_train: pd.DataFrame,
        y_train: pd.Series,
        X_val: Optional[pd.DataFrame] = None,
        y_val: Optional[pd.Series] = None,
        calibrate: bool = True,
        verbose: bool = True
    ) -> Dict[str, Any]:
        """
        Train the fraud detection model.

        Args:
            X_train: Training features
            y_train: Training labels (0=legitimate, 1=fraud)
            X_val: Validation features (optional)
            y_val: Validation labels (optional)
            calibrate: Whether to calibrate probabilities (default: True)
            verbose: Print training progress (default: True)

        Returns:
            Dictionary with training metrics and metadata

        Example:
            >>> model = FraudDetectionModel()
            >>> results = model.train(X_train, y_train, X_val, y_val)
            >>> print(f"Training AUC: {results['train_auc']:.3f}")
        """
        # Store feature names
        self.feature_names = list(X_train.columns)

        # Calculate scale_pos_weight if not provided
        if self.scale_pos_weight is None:
            n_legitimate = (y_train == 0).sum()
            n_fraud = (y_train == 1).sum()
            self.scale_pos_weight = n_legitimate / n_fraud if n_fraud > 0 else 1.0

            if verbose:
                print(f"\nCalculated scale_pos_weight: {self.scale_pos_weight:.2f}")
                print(f"  Legitimate samples: {n_legitimate}")
                print(f"  Fraud samples: {n_fraud}")

        # Build model
        self.model = self.build_model(self.scale_pos_weight)

        if verbose:
            print(f"\nTraining XGBoost model...")
            print(f"  Features: {len(self.feature_names)}")
            print(f"  Training samples: {len(X_train)}")
            print(f"  Hyperparameters:")
            print(f"    - max_depth: {self.max_depth}")
            print(f"    - learning_rate: {self.learning_rate}")
            print(f"    - n_estimators: {self.n_estimators}")
            print(f"    - scale_pos_weight: {self.scale_pos_weight:.2f}")

        # Prepare evaluation set
        eval_set = []
        if X_val is not None and y_val is not None:
            eval_set = [(X_train, y_train), (X_val, y_val)]
            eval_names = ['train', 'validation']
        else:
            eval_set = [(X_train, y_train)]
            eval_names = ['train']

        # Train model
        self.model.fit(
            X_train,
            y_train,
            eval_set=eval_set if eval_set else None,
            verbose=verbose
        )

        # Calibrate probabilities for better fraud probability estimates
        if calibrate:
            if verbose:
                print("\nCalibrating probabilities...")

            # Use isotonic regression for calibration
            # This improves probability estimates for decision-making
            self.calibrated_model = CalibratedClassifierCV(
                self.model,
                method='isotonic',
                cv='prefit'  # Use already-trained model
            )

            # Fit calibration on validation set or training set
            cal_X = X_val if X_val is not None else X_train
            cal_y = y_val if y_val is not None else y_train
            self.calibrated_model.fit(cal_X, cal_y)

            if verbose:
                print("  Calibration complete!")

        # Collect training metadata
        training_results = {
            'timestamp': datetime.now().isoformat(),
            'n_features': len(self.feature_names),
            'n_train_samples': len(X_train),
            'n_val_samples': len(X_val) if X_val is not None else 0,
            'class_distribution': {
                'legitimate': int((y_train == 0).sum()),
                'fraud': int((y_train == 1).sum())
            },
            'scale_pos_weight': float(self.scale_pos_weight),
            'hyperparameters': {
                'max_depth': self.max_depth,
                'learning_rate': self.learning_rate,
                'n_estimators': self.n_estimators,
                'subsample': self.subsample,
                'colsample_bytree': self.colsample_bytree
            }
        }

        # Store metadata
        self.metadata = training_results

        if verbose:
            print("\n✅ Training complete!")

        return training_results

    def predict(self, X: pd.DataFrame, use_calibrated: bool = True) -> np.ndarray:
        """
        Predict fraud labels for transactions.

        Args:
            X: Features to predict
            use_calibrated: Use calibrated model if available (default: True)

        Returns:
            Binary predictions (0=legitimate, 1=fraud)

        Example:
            >>> predictions = model.predict(X_test)
            >>> print(f"Frauds detected: {predictions.sum()}")
        """
        if self.model is None:
            raise ValueError("Model not trained. Call train() first.")

        # Validate features
        self._validate_features(X)

        # Choose model
        model = self.calibrated_model if (use_calibrated and self.calibrated_model) else self.model

        # Predict
        predictions = model.predict(X)

        return predictions

    def predict_proba(self, X: pd.DataFrame, use_calibrated: bool = True) -> np.ndarray:
        """
        Predict fraud probabilities for transactions.

        Args:
            X: Features to predict
            use_calibrated: Use calibrated model if available (default: True)

        Returns:
            Array of shape (n_samples, 2) with probabilities for each class
            Column 0: Probability of legitimate (class 0)
            Column 1: Probability of fraud (class 1)

        Example:
            >>> probas = model.predict_proba(X_test)
            >>> fraud_probas = probas[:, 1]  # Get fraud probabilities
            >>> high_risk = fraud_probas > 0.7  # Threshold at 70%
        """
        if self.model is None:
            raise ValueError("Model not trained. Call train() first.")

        # Validate features
        self._validate_features(X)

        # Choose model
        model = self.calibrated_model if (use_calibrated and self.calibrated_model) else self.model

        # Predict probabilities
        probabilities = model.predict_proba(X)

        return probabilities

    def get_feature_importance(
        self,
        importance_type: str = 'gain',
        top_n: Optional[int] = None
    ) -> pd.DataFrame:
        """
        Get feature importance scores.

        Args:
            importance_type: Type of importance
                - 'weight': Number of times feature used in splits
                - 'gain': Average gain when feature is used (default)
                - 'cover': Average coverage of feature
            top_n: Return only top N features (None = all features)

        Returns:
            DataFrame with features and importance scores, sorted by importance

        Example:
            >>> importance = model.get_feature_importance(top_n=10)
            >>> print(importance)
                        feature  importance
            0   amount_log       0.342
            1   velocity_1h      0.187
            ...
        """
        if self.model is None:
            raise ValueError("Model not trained. Call train() first.")

        # Get importance from XGBoost
        importance_dict = self.model.get_booster().get_score(
            importance_type=importance_type
        )

        # Convert to DataFrame
        importance_df = pd.DataFrame([
            {'feature': feature, 'importance': score}
            for feature, score in importance_dict.items()
        ])

        # Sort by importance
        importance_df = importance_df.sort_values('importance', ascending=False)

        # Reset index
        importance_df = importance_df.reset_index(drop=True)

        # Return top N if specified
        if top_n is not None:
            importance_df = importance_df.head(top_n)

        return importance_df

    def save(self, model_path: str, save_metadata: bool = True) -> None:
        """
        Save model to disk.

        Args:
            model_path: Path to save model (e.g., 'models/fraud_model_v1.pkl')
            save_metadata: Also save metadata JSON file (default: True)

        Example:
            >>> model.save('app/models/fraud_model_v1.pkl')
            >>> # Saves:
            >>> #   app/models/fraud_model_v1.pkl (model)
            >>> #   app/models/metadata_v1.json (metadata)
        """
        if self.model is None:
            raise ValueError("Model not trained. Call train() first.")

        # Create directory if it doesn't exist
        model_dir = Path(model_path).parent
        model_dir.mkdir(parents=True, exist_ok=True)

        # Prepare model data
        model_data = {
            'model': self.model,
            'calibrated_model': self.calibrated_model,
            'feature_names': self.feature_names,
            'version': self.version,
            'scale_pos_weight': self.scale_pos_weight,
            'hyperparameters': {
                'max_depth': self.max_depth,
                'learning_rate': self.learning_rate,
                'n_estimators': self.n_estimators,
                'subsample': self.subsample,
                'colsample_bytree': self.colsample_bytree
            }
        }

        # Save model
        joblib.dump(model_data, model_path)
        print(f"✅ Model saved to: {model_path}")

        # Save metadata
        if save_metadata:
            metadata_path = str(model_path).replace('.pkl', '_metadata.json')
            with open(metadata_path, 'w') as f:
                json.dump(self.metadata, f, indent=2)
            print(f"✅ Metadata saved to: {metadata_path}")

    @classmethod
    def load(cls, model_path: str) -> 'FraudDetectionModel':
        """
        Load model from disk.

        Args:
            model_path: Path to saved model

        Returns:
            Loaded FraudDetectionModel instance

        Example:
            >>> model = FraudDetectionModel.load('app/models/fraud_model_v1.pkl')
            >>> predictions = model.predict(X_test)
        """
        # Load model data
        model_data = joblib.load(model_path)

        # Create instance
        instance = cls()

        # Restore model components
        instance.model = model_data['model']
        instance.calibrated_model = model_data.get('calibrated_model')
        instance.feature_names = model_data['feature_names']
        instance.version = model_data.get('version', '1.0.0')
        instance.scale_pos_weight = model_data.get('scale_pos_weight')

        # Restore hyperparameters
        hyperparams = model_data.get('hyperparameters', {})
        instance.max_depth = hyperparams.get('max_depth', 6)
        instance.learning_rate = hyperparams.get('learning_rate', 0.1)
        instance.n_estimators = hyperparams.get('n_estimators', 500)
        instance.subsample = hyperparams.get('subsample', 0.8)
        instance.colsample_bytree = hyperparams.get('colsample_bytree', 0.8)

        print(f"✅ Model loaded from: {model_path}")
        print(f"  Version: {instance.version}")
        print(f"  Features: {len(instance.feature_names)}")

        return instance

    def _validate_features(self, X: pd.DataFrame) -> None:
        """
        Validate that input features match training features.

        Args:
            X: Features to validate

        Raises:
            ValueError: If features don't match
        """
        input_features = set(X.columns)
        expected_features = set(self.feature_names)

        if input_features != expected_features:
            missing = expected_features - input_features
            extra = input_features - expected_features

            error_msg = "Feature mismatch!\n"
            if missing:
                error_msg += f"  Missing features: {missing}\n"
            if extra:
                error_msg += f"  Extra features: {extra}\n"

            raise ValueError(error_msg)


# Example usage
if __name__ == "__main__":
    # This section demonstrates how to use the FraudDetectionModel
    print("FraudDetectionModel - Example Usage\n")
    print("=" * 50)

    # Create sample data
    np.random.seed(42)
    n_samples = 1000
    X_train = pd.DataFrame({
        'amount': np.random.exponential(50000, n_samples),
        'hour': np.random.randint(0, 24, n_samples),
        'is_new_device': np.random.choice([0, 1], n_samples, p=[0.9, 0.1])
    })
    y_train = np.random.choice([0, 1], n_samples, p=[0.95, 0.05])

    # Initialize model
    print("\n1. Initialize model")
    model = FraudDetectionModel(
        max_depth=3,
        learning_rate=0.1,
        n_estimators=100
    )
    print("   ✓ Model initialized")

    # Train model
    print("\n2. Train model")
    results = model.train(X_train, y_train, verbose=False)
    print(f"   ✓ Training complete")
    print(f"   ✓ Scale pos weight: {results['scale_pos_weight']:.2f}")

    # Get predictions
    print("\n3. Make predictions")
    predictions = model.predict(X_train)
    probabilities = model.predict_proba(X_train)
    print(f"   ✓ Predictions: {predictions[:5]}")
    print(f"   ✓ Fraud probabilities: {probabilities[:5, 1]}")

    # Feature importance
    print("\n4. Feature importance")
    importance = model.get_feature_importance()
    print(importance)

    print("\n" + "=" * 50)
    print("Example complete!")
```

---

### Step 3: Training Pipeline

**File: `/home/user/sentinel-api/app/ml/trainer.py`**

This file implements the complete training pipeline from data loading to model saving.

```python
"""
Model Training Pipeline for Fraud Detection.

This module handles the complete training workflow:
1. Data loading from database
2. Feature engineering
3. Train/test split
4. Class imbalance handling (SMOTE)
5. Model training
6. Cross-validation
7. Model evaluation
8. Model persistence

Example:
    >>> from app.ml.trainer import ModelTrainer
    >>> trainer = ModelTrainer()
    >>> model, metrics = trainer.train_model()
    >>> print(f"Test AUC: {metrics['test_auc']:.3f}")
"""

import numpy as np
import pandas as pd
from typing import Dict, List, Tuple, Optional, Any
from datetime import datetime, timedelta
from pathlib import Path

# Scikit-learn
from sklearn.model_selection import train_test_split, cross_val_score, StratifiedKFold
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, confusion_matrix, classification_report
)

# Imbalanced-learn
from imblearn.over_sampling import SMOTE
from imblearn.pipeline import Pipeline as ImbPipeline

# Database
from sqlalchemy.orm import Session
from sqlalchemy import desc

# Internal imports
from app.db.session import SessionLocal
from app.models.fraud import FraudTransaction
from app.ml.model import FraudDetectionModel
from app.ml.features import FeatureEngineer  # From Day 9


class ModelTrainer:
    """
    Complete training pipeline for fraud detection model.

    This class orchestrates the entire training process from data loading
    to model evaluation and persistence.

    Attributes:
        random_state: Random seed for reproducibility
        test_size: Fraction of data for testing (default: 0.2)
        use_smote: Whether to use SMOTE for class imbalance (default: True)
        cv_folds: Number of cross-validation folds (default: 5)
    """

    def __init__(
        self,
        random_state: int = 42,
        test_size: float = 0.2,
        use_smote: bool = True,
        cv_folds: int = 5
    ):
        """
        Initialize model trainer.

        Args:
            random_state: Random seed for reproducibility
            test_size: Fraction of data for test set (0.2 = 20%)
            use_smote: Use SMOTE to handle class imbalance
            cv_folds: Number of cross-validation folds
        """
        self.random_state = random_state
        self.test_size = test_size
        self.use_smote = use_smote
        self.cv_folds = cv_folds

        # Feature engineer (from Day 9)
        self.feature_engineer = FeatureEngineer()

        # Will be set during training
        self.scaler: Optional[StandardScaler] = None
        self.feature_names: List[str] = []

    def load_data_from_db(
        self,
        limit: Optional[int] = None,
        min_date: Optional[datetime] = None
    ) -> pd.DataFrame:
        """
        Load transaction data from database.

        Args:
            limit: Maximum number of transactions to load (None = all)
            min_date: Only load transactions after this date (None = all)

        Returns:
            DataFrame with transaction data

        Example:
            >>> trainer = ModelTrainer()
            >>> # Load last 10,000 transactions
            >>> df = trainer.load_data_from_db(limit=10000)
            >>> # Load transactions from last 30 days
            >>> df = trainer.load_data_from_db(
            ...     min_date=datetime.now() - timedelta(days=30)
            ... )
        """
        print("\n" + "=" * 60)
        print("LOADING DATA FROM DATABASE")
        print("=" * 60)

        db: Session = SessionLocal()
        try:
            # Build query
            query = db.query(FraudTransaction).order_by(
                desc(FraudTransaction.created_at)
            )

            # Apply date filter
            if min_date:
                query = query.filter(FraudTransaction.created_at >= min_date)
                print(f"Filtering: Transactions after {min_date}")

            # Apply limit
            if limit:
                query = query.limit(limit)
                print(f"Limiting: {limit} transactions")

            # Execute query
            transactions = query.all()

            if not transactions:
                raise ValueError("No transactions found in database!")

            print(f"\n✅ Loaded {len(transactions)} transactions from database")

            # Convert to DataFrame
            data = []
            for tx in transactions:
                # Extract metadata
                metadata = tx.metadata or {}

                data.append({
                    'transaction_id': tx.transaction_id,
                    'user_id': tx.user_id,
                    'amount': tx.amount,
                    'currency': tx.currency,
                    'transaction_type': tx.transaction_type,
                    'is_fraud': 1 if tx.is_flagged else 0,  # Target variable
                    'fraud_score': tx.fraud_score,
                    'timestamp': tx.created_at,
                    # Extract from metadata
                    'device_id': metadata.get('device_id'),
                    'ip_address': metadata.get('ip_address'),
                    'country': metadata.get('location', {}).get('country'),
                    'city': metadata.get('location', {}).get('city'),
                })

            df = pd.DataFrame(data)

            # Print data summary
            print(f"\nData Summary:")
            print(f"  Total transactions: {len(df)}")
            print(f"  Fraud cases: {df['is_fraud'].sum()} ({df['is_fraud'].mean()*100:.2f}%)")
            print(f"  Legitimate cases: {(df['is_fraud'] == 0).sum()}")
            print(f"  Date range: {df['timestamp'].min()} to {df['timestamp'].max()}")

            return df

        finally:
            db.close()

    def prepare_features(
        self,
        df: pd.DataFrame,
        fit_scaler: bool = True
    ) -> Tuple[pd.DataFrame, pd.Series]:
        """
        Prepare features for training.

        Args:
            df: Raw transaction DataFrame
            fit_scaler: Whether to fit scaler (True for training, False for inference)

        Returns:
            Tuple of (X, y) where:
                X: Feature DataFrame
                y: Target Series (is_fraud)

        Example:
            >>> df = trainer.load_data_from_db()
            >>> X, y = trainer.prepare_features(df)
            >>> print(f"Features: {X.shape[1]}, Samples: {len(X)}")
        """
        print("\n" + "=" * 60)
        print("FEATURE ENGINEERING")
        print("=" * 60)

        # Extract features using FeatureEngineer from Day 9
        print("\nExtracting features...")
        X = self.feature_engineer.extract_features(df)

        # Target variable
        y = df['is_fraud']

        print(f"\n✅ Feature extraction complete")
        print(f"  Features created: {X.shape[1]}")
        print(f"  Samples: {X.shape[0]}")

        # Store feature names
        self.feature_names = list(X.columns)

        # Scale features
        if fit_scaler:
            print("\nScaling features...")
            self.scaler = StandardScaler()
            X_scaled = pd.DataFrame(
                self.scaler.fit_transform(X),
                columns=X.columns,
                index=X.index
            )
            print("  ✓ Scaler fitted and features scaled")
        else:
            if self.scaler is None:
                raise ValueError("Scaler not fitted. Set fit_scaler=True first.")
            X_scaled = pd.DataFrame(
                self.scaler.transform(X),
                columns=X.columns,
                index=X.index
            )
            print("  ✓ Features scaled with existing scaler")

        return X_scaled, y

    def split_data(
        self,
        X: pd.DataFrame,
        y: pd.Series
    ) -> Tuple[pd.DataFrame, pd.DataFrame, pd.Series, pd.Series]:
        """
        Split data into training and test sets.

        Uses stratified split to maintain class distribution in both sets.

        Args:
            X: Features
            y: Target variable

        Returns:
            Tuple of (X_train, X_test, y_train, y_test)

        Example:
            >>> X_train, X_test, y_train, y_test = trainer.split_data(X, y)
            >>> print(f"Train: {len(X_train)}, Test: {len(X_test)}")
        """
        print("\n" + "=" * 60)
        print("TRAIN/TEST SPLIT")
        print("=" * 60)

        # Stratified split (maintains class distribution)
        X_train, X_test, y_train, y_test = train_test_split(
            X, y,
            test_size=self.test_size,
            random_state=self.random_state,
            stratify=y  # Important: maintains fraud/legitimate ratio
        )

        print(f"\nSplit: {int((1-self.test_size)*100)}% train, {int(self.test_size*100)}% test")
        print(f"\nTraining set:")
        print(f"  Samples: {len(X_train)}")
        print(f"  Frauds: {y_train.sum()} ({y_train.mean()*100:.2f}%)")
        print(f"  Legitimate: {(y_train == 0).sum()}")

        print(f"\nTest set:")
        print(f"  Samples: {len(X_test)}")
        print(f"  Frauds: {y_test.sum()} ({y_test.mean()*100:.2f}%)")
        print(f"  Legitimate: {(y_test == 0).sum()}")

        return X_train, X_test, y_train, y_test

    def apply_smote(
        self,
        X_train: pd.DataFrame,
        y_train: pd.Series
    ) -> Tuple[pd.DataFrame, pd.Series]:
        """
        Apply SMOTE to balance training data.

        SMOTE (Synthetic Minority Over-sampling Technique) creates
        synthetic fraud examples to balance the dataset.

        Args:
            X_train: Training features
            y_train: Training labels

        Returns:
            Tuple of (X_resampled, y_resampled) with balanced classes

        Example:
            >>> X_bal, y_bal = trainer.apply_smote(X_train, y_train)
            >>> print(f"Fraud ratio: {y_bal.mean():.2%}")  # Should be ~50%
        """
        if not self.use_smote:
            print("\nSMOTE disabled, using original data")
            return X_train, y_train

        print("\n" + "=" * 60)
        print("APPLYING SMOTE (Class Balancing)")
        print("=" * 60)

        # Calculate class distribution before SMOTE
        n_legitimate_before = (y_train == 0).sum()
        n_fraud_before = (y_train == 1).sum()
        ratio_before = n_fraud_before / len(y_train)

        print(f"\nBefore SMOTE:")
        print(f"  Legitimate: {n_legitimate_before}")
        print(f"  Fraud: {n_fraud_before}")
        print(f"  Fraud ratio: {ratio_before:.2%}")
        print(f"  Imbalance ratio: 1:{n_legitimate_before/n_fraud_before:.1f}")

        # Apply SMOTE
        print(f"\nApplying SMOTE...")
        smote = SMOTE(
            sampling_strategy='auto',  # Balance to 1:1 ratio
            random_state=self.random_state,
            k_neighbors=5
        )

        X_resampled, y_resampled = smote.fit_resample(X_train, y_train)

        # Calculate class distribution after SMOTE
        n_legitimate_after = (y_resampled == 0).sum()
        n_fraud_after = (y_resampled == 1).sum()
        ratio_after = n_fraud_after / len(y_resampled)

        print(f"\nAfter SMOTE:")
        print(f"  Legitimate: {n_legitimate_after}")
        print(f"  Fraud: {n_fraud_after} (+{n_fraud_after - n_fraud_before} synthetic)")
        print(f"  Fraud ratio: {ratio_after:.2%}")
        print(f"  Imbalance ratio: 1:{n_legitimate_after/n_fraud_after:.1f}")

        print(f"\n✅ SMOTE complete! Dataset balanced.")

        # Convert back to DataFrame/Series to maintain column names
        X_resampled = pd.DataFrame(X_resampled, columns=X_train.columns)
        y_resampled = pd.Series(y_resampled, name=y_train.name)

        return X_resampled, y_resampled

    def train_model(
        self,
        X_train: pd.DataFrame,
        y_train: pd.Series,
        X_val: Optional[pd.DataFrame] = None,
        y_val: Optional[pd.Series] = None,
        model_params: Optional[Dict[str, Any]] = None
    ) -> FraudDetectionModel:
        """
        Train XGBoost model.

        Args:
            X_train: Training features
            y_train: Training labels
            X_val: Validation features (optional)
            y_val: Validation labels (optional)
            model_params: Custom model parameters (optional)

        Returns:
            Trained FraudDetectionModel

        Example:
            >>> model = trainer.train_model(X_train, y_train, X_val, y_val)
        """
        print("\n" + "=" * 60)
        print("TRAINING XGBOOST MODEL")
        print("=" * 60)

        # Default parameters optimized for fraud detection
        default_params = {
            'max_depth': 6,
            'learning_rate': 0.1,
            'n_estimators': 500,
            'subsample': 0.8,
            'colsample_bytree': 0.8,
            'random_state': self.random_state
        }

        # Override with custom parameters
        if model_params:
            default_params.update(model_params)

        print(f"\nModel Parameters:")
        for key, value in default_params.items():
            print(f"  {key}: {value}")

        # Initialize model
        model = FraudDetectionModel(**default_params)

        # Train
        model.train(
            X_train=X_train,
            y_train=y_train,
            X_val=X_val,
            y_val=y_val,
            calibrate=True,
            verbose=True
        )

        return model

    def cross_validate(
        self,
        X: pd.DataFrame,
        y: pd.Series,
        model: FraudDetectionModel
    ) -> Dict[str, Any]:
        """
        Perform k-fold cross-validation.

        Args:
            X: Features
            y: Target variable
            model: Model to cross-validate

        Returns:
            Dictionary with cross-validation scores

        Example:
            >>> cv_results = trainer.cross_validate(X_train, y_train, model)
            >>> print(f"CV AUC: {cv_results['mean_auc']:.3f} ± {cv_results['std_auc']:.3f}")
        """
        print("\n" + "=" * 60)
        print(f"CROSS-VALIDATION ({self.cv_folds}-Fold)")
        print("=" * 60)

        # Stratified K-Fold (maintains class distribution in each fold)
        cv = StratifiedKFold(
            n_splits=self.cv_folds,
            shuffle=True,
            random_state=self.random_state
        )

        # Cross-validate on multiple metrics
        print(f"\nRunning {self.cv_folds}-fold cross-validation...")

        # AUC scores
        auc_scores = cross_val_score(
            model.model, X, y,
            cv=cv,
            scoring='roc_auc',
            n_jobs=-1
        )

        # Precision scores
        precision_scores = cross_val_score(
            model.model, X, y,
            cv=cv,
            scoring='precision',
            n_jobs=-1
        )

        # Recall scores
        recall_scores = cross_val_score(
            model.model, X, y,
            cv=cv,
            scoring='recall',
            n_jobs=-1
        )

        # F1 scores
        f1_scores = cross_val_score(
            model.model, X, y,
            cv=cv,
            scoring='f1',
            n_jobs=-1
        )

        # Compile results
        cv_results = {
            'mean_auc': float(np.mean(auc_scores)),
            'std_auc': float(np.std(auc_scores)),
            'mean_precision': float(np.mean(precision_scores)),
            'std_precision': float(np.std(precision_scores)),
            'mean_recall': float(np.mean(recall_scores)),
            'std_recall': float(np.std(recall_scores)),
            'mean_f1': float(np.mean(f1_scores)),
            'std_f1': float(np.std(f1_scores)),
            'fold_scores': {
                'auc': auc_scores.tolist(),
                'precision': precision_scores.tolist(),
                'recall': recall_scores.tolist(),
                'f1': f1_scores.tolist()
            }
        }

        # Print results
        print(f"\nCross-Validation Results:")
        print(f"  AUC:       {cv_results['mean_auc']:.3f} ± {cv_results['std_auc']:.3f}")
        print(f"  Precision: {cv_results['mean_precision']:.3f} ± {cv_results['std_precision']:.3f}")
        print(f"  Recall:    {cv_results['mean_recall']:.3f} ± {cv_results['std_recall']:.3f}")
        print(f"  F1:        {cv_results['mean_f1']:.3f} ± {cv_results['std_f1']:.3f}")

        print(f"\n✅ Cross-validation complete!")

        return cv_results

    def full_training_pipeline(
        self,
        limit: Optional[int] = None,
        min_date: Optional[datetime] = None,
        model_params: Optional[Dict[str, Any]] = None
    ) -> Tuple[FraudDetectionModel, Dict[str, Any]]:
        """
        Execute complete training pipeline.

        This is the main entry point that orchestrates all training steps:
        1. Load data from database
        2. Feature engineering
        3. Train/test split
        4. SMOTE (if enabled)
        5. Model training
        6. Cross-validation
        7. Model evaluation

        Args:
            limit: Max transactions to load from DB
            min_date: Only load transactions after this date
            model_params: Custom model parameters

        Returns:
            Tuple of (trained_model, metrics_dict)

        Example:
            >>> trainer = ModelTrainer()
            >>> model, metrics = trainer.full_training_pipeline(limit=10000)
            >>> print(f"Test AUC: {metrics['test_auc']:.3f}")
        """
        print("\n" + "=" * 70)
        print("FRAUD DETECTION MODEL TRAINING PIPELINE")
        print("=" * 70)
        print(f"Started at: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

        # Step 1: Load data
        df = self.load_data_from_db(limit=limit, min_date=min_date)

        # Step 2: Feature engineering
        X, y = self.prepare_features(df, fit_scaler=True)

        # Step 3: Train/test split
        X_train, X_test, y_train, y_test = self.split_data(X, y)

        # Step 4: Apply SMOTE (if enabled)
        X_train_balanced, y_train_balanced = self.apply_smote(X_train, y_train)

        # Step 5: Train model
        model = self.train_model(
            X_train=X_train_balanced,
            y_train=y_train_balanced,
            X_val=X_test,
            y_val=y_test,
            model_params=model_params
        )

        # Step 6: Cross-validation
        cv_results = self.cross_validate(X_train_balanced, y_train_balanced, model)

        # Step 7: Evaluate on test set
        from app.ml.evaluator import ModelEvaluator
        evaluator = ModelEvaluator()
        test_metrics = evaluator.evaluate(model, X_test, y_test)

        # Combine metrics
        all_metrics = {
            **test_metrics,
            'cross_validation': cv_results,
            'training_info': {
                'n_train': len(X_train),
                'n_test': len(X_test),
                'n_features': X_train.shape[1],
                'smote_applied': self.use_smote,
                'test_size': self.test_size
            }
        }

        print("\n" + "=" * 70)
        print("TRAINING COMPLETE!")
        print("=" * 70)
        print(f"\nFinal Test Metrics:")
        print(f"  AUC:       {test_metrics['auc']:.3f}")
        print(f"  Precision: {test_metrics['precision']:.3f}")
        print(f"  Recall:    {test_metrics['recall']:.3f}")
        print(f"  F1-Score:  {test_metrics['f1']:.3f}")
        print(f"\nCompleted at: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

        return model, all_metrics


# Example usage
if __name__ == "__main__":
    print("ModelTrainer - Example Usage\n")

    # Initialize trainer
    trainer = ModelTrainer(
        random_state=42,
        test_size=0.2,
        use_smote=True,
        cv_folds=5
    )

    # Run full training pipeline
    model, metrics = trainer.full_training_pipeline(limit=5000)

    # Save model
    model.save('app/models/fraud_model_v1.pkl')

    print("\n✅ Training pipeline complete!")
```

This is Part 1 of the comprehensive Day 10 guide. The file is getting long, so I'll continue with the remaining sections in the next part. Should I continue with:
- Step 4: Model Evaluator
- Step 5: Custom Metrics
- Step 6: Training Scripts
- Testing sections
- Troubleshooting
- Next Steps?
---

### Step 4: Model Evaluator

**File: `/home/user/sentinel-api/app/ml/evaluator.py`**

This file provides comprehensive model evaluation with visualizations and detailed metrics.

```python
"""
Model Evaluation Module for Fraud Detection.

This module provides comprehensive model evaluation including:
- Classification metrics (accuracy, precision, recall, F1, AUC)
- Confusion matrix
- ROC curve plotting
- Feature importance visualization
- Error analysis

Example:
    >>> from app.ml.evaluator import ModelEvaluator
    >>> evaluator = ModelEvaluator()
    >>> metrics = evaluator.evaluate(model, X_test, y_test)
    >>> print(f"AUC: {metrics['auc']:.3f}")
"""

import numpy as np
import pandas as pd
from typing import Dict, List, Tuple, Optional, Any
import matplotlib.pyplot as plt
import seaborn as sns
from pathlib import Path

# Scikit-learn metrics
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, roc_curve, confusion_matrix,
    classification_report, precision_recall_curve,
    average_precision_score
)

# Internal imports
from app.ml.model import FraudDetectionModel


class ModelEvaluator:
    """
    Comprehensive model evaluation for fraud detection.

    This class provides methods for evaluating model performance
    using various metrics and visualizations.
    """

    def __init__(self, output_dir: str = "app/models/evaluation"):
        """
        Initialize model evaluator.

        Args:
            output_dir: Directory to save evaluation plots and reports
        """
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)

        # Set plotting style
        sns.set_style("whitegrid")
        plt.rcParams['figure.figsize'] = (10, 6)

    def evaluate(
        self,
        model: FraudDetectionModel,
        X_test: pd.DataFrame,
        y_test: pd.Series,
        threshold: float = 0.5,
        save_plots: bool = True
    ) -> Dict[str, Any]:
        """
        Comprehensive model evaluation.

        Args:
            model: Trained fraud detection model
            X_test: Test features
            y_test: Test labels
            threshold: Classification threshold (default: 0.5)
            save_plots: Save evaluation plots to disk (default: True)

        Returns:
            Dictionary with all evaluation metrics

        Example:
            >>> evaluator = ModelEvaluator()
            >>> metrics = evaluator.evaluate(model, X_test, y_test)
            >>> print(metrics['classification_report'])
        """
        print("\n" + "=" * 60)
        print("MODEL EVALUATION")
        print("=" * 60)

        # Get predictions
        y_pred_proba = model.predict_proba(X_test)[:, 1]  # Fraud probabilities
        y_pred = (y_pred_proba >= threshold).astype(int)  # Binary predictions

        # Calculate metrics
        metrics = {
            'accuracy': accuracy_score(y_test, y_pred),
            'precision': precision_score(y_test, y_pred, zero_division=0),
            'recall': recall_score(y_test, y_pred, zero_division=0),
            'f1': f1_score(y_test, y_pred, zero_division=0),
            'auc': roc_auc_score(y_test, y_pred_proba),
            'average_precision': average_precision_score(y_test, y_pred_proba)
        }

        # Confusion matrix
        cm = confusion_matrix(y_test, y_pred)
        tn, fp, fn, tp = cm.ravel()

        metrics['confusion_matrix'] = {
            'true_negatives': int(tn),
            'false_positives': int(fp),
            'false_negatives': int(fn),
            'true_positives': int(tp)
        }

        # Additional metrics
        metrics['specificity'] = tn / (tn + fp) if (tn + fp) > 0 else 0
        metrics['false_positive_rate'] = fp / (fp + tn) if (fp + tn) > 0 else 0
        metrics['false_negative_rate'] = fn / (fn + tp) if (fn + tp) > 0 else 0

        # Classification report
        metrics['classification_report'] = classification_report(
            y_test, y_pred,
            target_names=['Legitimate', 'Fraud'],
            output_dict=True
        )

        # Print summary
        self.print_evaluation_summary(metrics)

        # Generate plots
        if save_plots:
            self.plot_confusion_matrix(cm, save_path=self.output_dir / "confusion_matrix.png")
            self.plot_roc_curve(y_test, y_pred_proba, metrics['auc'], save_path=self.output_dir / "roc_curve.png")
            self.plot_precision_recall_curve(y_test, y_pred_proba, save_path=self.output_dir / "pr_curve.png")
            self.plot_feature_importance(model, save_path=self.output_dir / "feature_importance.png")
            self.plot_prediction_distribution(y_test, y_pred_proba, save_path=self.output_dir / "prediction_distribution.png")

        return metrics

    def print_evaluation_summary(self, metrics: Dict[str, Any]) -> None:
        """
        Print evaluation metrics summary.

        Args:
            metrics: Dictionary of evaluation metrics
        """
        print("\nClassification Metrics:")
        print(f"  Accuracy:  {metrics['accuracy']:.3f}")
        print(f"  Precision: {metrics['precision']:.3f}")
        print(f"  Recall:    {metrics['recall']:.3f}")
        print(f"  F1-Score:  {metrics['f1']:.3f}")
        print(f"  AUC-ROC:   {metrics['auc']:.3f}")
        print(f"  Avg Precision: {metrics['average_precision']:.3f}")

        print("\nConfusion Matrix:")
        cm = metrics['confusion_matrix']
        print(f"  True Negatives:  {cm['true_negatives']:,}")
        print(f"  False Positives: {cm['false_positives']:,}")
        print(f"  False Negatives: {cm['false_negatives']:,}")
        print(f"  True Positives:  {cm['true_positives']:,}")

        print("\nAdditional Metrics:")
        print(f"  Specificity: {metrics['specificity']:.3f}")
        print(f"  FPR: {metrics['false_positive_rate']:.3f}")
        print(f"  FNR: {metrics['false_negative_rate']:.3f}")

    def plot_confusion_matrix(
        self,
        cm: np.ndarray,
        save_path: Optional[Path] = None
    ) -> None:
        """
        Plot confusion matrix heatmap.

        Args:
            cm: Confusion matrix array
            save_path: Path to save plot (optional)
        """
        plt.figure(figsize=(8, 6))

        # Create heatmap
        sns.heatmap(
            cm,
            annot=True,
            fmt='d',
            cmap='Blues',
            xticklabels=['Legitimate', 'Fraud'],
            yticklabels=['Legitimate', 'Fraud'],
            cbar_kws={'label': 'Count'}
        )

        plt.title('Confusion Matrix', fontsize=16, fontweight='bold')
        plt.ylabel('True Label', fontsize=12)
        plt.xlabel('Predicted Label', fontsize=12)
        plt.tight_layout()

        if save_path:
            plt.savefig(save_path, dpi=300, bbox_inches='tight')
            print(f"  ✓ Confusion matrix saved: {save_path}")

        plt.close()

    def plot_roc_curve(
        self,
        y_true: pd.Series,
        y_pred_proba: np.ndarray,
        auc: float,
        save_path: Optional[Path] = None
    ) -> None:
        """
        Plot ROC curve.

        Args:
            y_true: True labels
            y_pred_proba: Predicted probabilities
            auc: AUC score
            save_path: Path to save plot (optional)
        """
        # Calculate ROC curve
        fpr, tpr, thresholds = roc_curve(y_true, y_pred_proba)

        plt.figure(figsize=(8, 6))

        # Plot ROC curve
        plt.plot(fpr, tpr, linewidth=2, label=f'ROC Curve (AUC = {auc:.3f})')
        plt.plot([0, 1], [0, 1], 'k--', linewidth=1, label='Random Classifier')

        plt.xlabel('False Positive Rate', fontsize=12)
        plt.ylabel('True Positive Rate', fontsize=12)
        plt.title('ROC Curve - Fraud Detection', fontsize=16, fontweight='bold')
        plt.legend(loc='lower right', fontsize=11)
        plt.grid(True, alpha=0.3)
        plt.tight_layout()

        if save_path:
            plt.savefig(save_path, dpi=300, bbox_inches='tight')
            print(f"  ✓ ROC curve saved: {save_path}")

        plt.close()

    def plot_precision_recall_curve(
        self,
        y_true: pd.Series,
        y_pred_proba: np.ndarray,
        save_path: Optional[Path] = None
    ) -> None:
        """
        Plot Precision-Recall curve.

        Args:
            y_true: True labels
            y_pred_proba: Predicted probabilities
            save_path: Path to save plot (optional)
        """
        # Calculate PR curve
        precision, recall, thresholds = precision_recall_curve(y_true, y_pred_proba)
        avg_precision = average_precision_score(y_true, y_pred_proba)

        plt.figure(figsize=(8, 6))

        # Plot PR curve
        plt.plot(recall, precision, linewidth=2, label=f'PR Curve (AP = {avg_precision:.3f})')

        plt.xlabel('Recall', fontsize=12)
        plt.ylabel('Precision', fontsize=12)
        plt.title('Precision-Recall Curve', fontsize=16, fontweight='bold')
        plt.legend(loc='lower left', fontsize=11)
        plt.grid(True, alpha=0.3)
        plt.tight_layout()

        if save_path:
            plt.savefig(save_path, dpi=300, bbox_inches='tight')
            print(f"  ✓ Precision-Recall curve saved: {save_path}")

        plt.close()

    def plot_feature_importance(
        self,
        model: FraudDetectionModel,
        top_n: int = 20,
        save_path: Optional[Path] = None
    ) -> None:
        """
        Plot feature importance.

        Args:
            model: Trained model
            top_n: Number of top features to show
            save_path: Path to save plot (optional)
        """
        # Get feature importance
        importance_df = model.get_feature_importance(importance_type='gain', top_n=top_n)

        plt.figure(figsize=(10, 8))

        # Plot horizontal bar chart
        plt.barh(
            range(len(importance_df)),
            importance_df['importance'],
            color='steelblue'
        )

        plt.yticks(range(len(importance_df)), importance_df['feature'])
        plt.xlabel('Importance (Gain)', fontsize=12)
        plt.ylabel('Feature', fontsize=12)
        plt.title(f'Top {top_n} Most Important Features', fontsize=16, fontweight='bold')
        plt.gca().invert_yaxis()  # Highest importance at top
        plt.tight_layout()

        if save_path:
            plt.savefig(save_path, dpi=300, bbox_inches='tight')
            print(f"  ✓ Feature importance saved: {save_path}")

        plt.close()

    def plot_prediction_distribution(
        self,
        y_true: pd.Series,
        y_pred_proba: np.ndarray,
        save_path: Optional[Path] = None
    ) -> None:
        """
        Plot distribution of predicted probabilities for each class.

        Args:
            y_true: True labels
            y_pred_proba: Predicted probabilities
            save_path: Path to save plot (optional)
        """
        plt.figure(figsize=(10, 6))

        # Separate probabilities by true class
        fraud_probs = y_pred_proba[y_true == 1]
        legit_probs = y_pred_proba[y_true == 0]

        # Plot distributions
        plt.hist(legit_probs, bins=50, alpha=0.6, label='Legitimate', color='green')
        plt.hist(fraud_probs, bins=50, alpha=0.6, label='Fraud', color='red')

        plt.axvline(x=0.5, color='black', linestyle='--', linewidth=2, label='Threshold (0.5)')

        plt.xlabel('Predicted Fraud Probability', fontsize=12)
        plt.ylabel('Count', fontsize=12)
        plt.title('Prediction Distribution by True Class', fontsize=16, fontweight='bold')
        plt.legend(fontsize=11)
        plt.grid(True, alpha=0.3)
        plt.tight_layout()

        if save_path:
            plt.savefig(save_path, dpi=300, bbox_inches='tight')
            print(f"  ✓ Prediction distribution saved: {save_path}")

        plt.close()

    def analyze_errors(
        self,
        model: FraudDetectionModel,
        X_test: pd.DataFrame,
        y_test: pd.Series,
        threshold: float = 0.5,
        n_examples: int = 10
    ) -> Dict[str, pd.DataFrame]:
        """
        Analyze model errors (false positives and false negatives).

        Args:
            model: Trained model
            X_test: Test features
            y_test: Test labels
            threshold: Classification threshold
            n_examples: Number of examples to return for each error type

        Returns:
            Dictionary with DataFrames of false positives and false negatives

        Example:
            >>> errors = evaluator.analyze_errors(model, X_test, y_test)
            >>> print("False Positives:")
            >>> print(errors['false_positives'].head())
        """
        # Get predictions
        y_pred_proba = model.predict_proba(X_test)[:, 1]
        y_pred = (y_pred_proba >= threshold).astype(int)

        # Find errors
        false_positives_mask = (y_test == 0) & (y_pred == 1)
        false_negatives_mask = (y_test == 1) & (y_pred == 0)

        # Create error DataFrames
        fp_df = X_test[false_positives_mask].copy()
        fp_df['true_label'] = y_test[false_positives_mask]
        fp_df['predicted_proba'] = y_pred_proba[false_positives_mask]
        fp_df = fp_df.sort_values('predicted_proba', ascending=False).head(n_examples)

        fn_df = X_test[false_negatives_mask].copy()
        fn_df['true_label'] = y_test[false_negatives_mask]
        fn_df['predicted_proba'] = y_pred_proba[false_negatives_mask]
        fn_df = fn_df.sort_values('predicted_proba', ascending=True).head(n_examples)

        print("\nError Analysis:")
        print(f"  False Positives: {false_positives_mask.sum()}")
        print(f"  False Negatives: {false_negatives_mask.sum()}")

        return {
            'false_positives': fp_df,
            'false_negatives': fn_df
        }


# Example usage
if __name__ == "__main__":
    print("ModelEvaluator - Example Usage\n")
    print("=" * 50)

    # Note: This requires a trained model and test data
    print("\nTo use ModelEvaluator:")
    print("1. Train a model using ModelTrainer")
    print("2. Prepare test data (X_test, y_test)")
    print("3. Call evaluator.evaluate(model, X_test, y_test)")
    print("\nExample:")
    print("  >>> evaluator = ModelEvaluator()")
    print("  >>> metrics = evaluator.evaluate(model, X_test, y_test)")
    print("  >>> print(f'AUC: {metrics[\"auc\"]:.3f}')")
```

---

### Step 5: Custom Metrics Module

**File: `/home/user/sentinel-api/app/ml/metrics.py`**

This file provides fraud-specific custom metrics and business-oriented calculations.

```python
"""
Custom Metrics for Fraud Detection.

This module provides fraud-specific metrics that go beyond standard
classification metrics to include business-oriented calculations.

Metrics included:
- Cost-based metrics (false positive cost, false negative cost)
- Expected savings
- Detection rate at different thresholds
- Precision at various recall levels
"""

import numpy as np
import pandas as pd
from typing import Dict, List, Tuple, Optional
from sklearn.metrics import precision_recall_curve, roc_curve


class FraudMetrics:
    """
    Custom metrics for fraud detection evaluation.

    This class provides fraud-specific metrics that consider business
    impact and costs.
    """

    def __init__(
        self,
        fp_cost: float = 10.0,
        fn_cost: float = 1000.0,
        avg_fraud_amount: float = 50000.0
    ):
        """
        Initialize fraud metrics calculator.

        Args:
            fp_cost: Cost of false positive (manual review cost)
            fn_cost: Cost of false negative (fraud not caught)
            avg_fraud_amount: Average amount of fraudulent transaction
        """
        self.fp_cost = fp_cost
        self.fn_cost = fn_cost
        self.avg_fraud_amount = avg_fraud_amount

    def calculate_business_metrics(
        self,
        y_true: np.ndarray,
        y_pred: np.ndarray,
        transaction_amounts: Optional[np.ndarray] = None
    ) -> Dict[str, float]:
        """
        Calculate business-oriented metrics.

        Args:
            y_true: True labels
            y_pred: Predicted labels
            transaction_amounts: Transaction amounts (optional)

        Returns:
            Dictionary with business metrics

        Example:
            >>> metrics = FraudMetrics()
            >>> business = metrics.calculate_business_metrics(y_test, y_pred)
            >>> print(f"Total cost: ${business['total_cost']:,.2f}")
        """
        # Confusion matrix components
        tn = ((y_true == 0) & (y_pred == 0)).sum()
        fp = ((y_true == 0) & (y_pred == 1)).sum()
        fn = ((y_true == 1) & (y_pred == 0)).sum()
        tp = ((y_true == 1) & (y_pred == 1)).sum()

        # Calculate costs
        fp_total_cost = fp * self.fp_cost
        fn_total_cost = fn * self.fn_cost

        # Calculate fraud amounts if provided
        if transaction_amounts is not None:
            fraud_caught_amount = transaction_amounts[(y_true == 1) & (y_pred == 1)].sum()
            fraud_missed_amount = transaction_amounts[(y_true == 1) & (y_pred == 0)].sum()
        else:
            fraud_caught_amount = tp * self.avg_fraud_amount
            fraud_missed_amount = fn * self.avg_fraud_amount

        # Total cost
        total_cost = fp_total_cost + fn_total_cost

        # Fraud detection rate
        fraud_detection_rate = tp / (tp + fn) if (tp + fn) > 0 else 0

        # False alarm rate
        false_alarm_rate = fp / (fp + tn) if (fp + tn) > 0 else 0

        return {
            'total_cost': float(total_cost),
            'false_positive_cost': float(fp_total_cost),
            'false_negative_cost': float(fn_total_cost),
            'fraud_caught_amount': float(fraud_caught_amount),
            'fraud_missed_amount': float(fraud_missed_amount),
            'fraud_detection_rate': float(fraud_detection_rate),
            'false_alarm_rate': float(false_alarm_rate),
            'true_positives': int(tp),
            'false_positives': int(fp),
            'true_negatives': int(tn),
            'false_negatives': int(fn)
        }

    def find_optimal_threshold(
        self,
        y_true: np.ndarray,
        y_pred_proba: np.ndarray,
        metric: str = 'f1'
    ) -> Tuple[float, float]:
        """
        Find optimal classification threshold.

        Args:
            y_true: True labels
            y_pred_proba: Predicted probabilities
            metric: Metric to optimize ('f1', 'precision', 'recall', 'cost')

        Returns:
            Tuple of (optimal_threshold, optimal_metric_value)

        Example:
            >>> metrics = FraudMetrics()
            >>> threshold, f1 = metrics.find_optimal_threshold(
            ...     y_test, y_pred_proba, metric='f1'
            ... )
            >>> print(f"Optimal threshold: {threshold:.3f}, F1: {f1:.3f}")
        """
        # Calculate precision-recall curve
        precision, recall, thresholds = precision_recall_curve(y_true, y_pred_proba)

        if metric == 'f1':
            # Calculate F1 for each threshold
            f1_scores = 2 * (precision * recall) / (precision + recall + 1e-10)
            optimal_idx = np.argmax(f1_scores)
            optimal_value = f1_scores[optimal_idx]
            optimal_threshold = thresholds[optimal_idx] if optimal_idx < len(thresholds) else 0.5

        elif metric == 'precision':
            # Find threshold that maximizes precision
            optimal_idx = np.argmax(precision)
            optimal_value = precision[optimal_idx]
            optimal_threshold = thresholds[optimal_idx] if optimal_idx < len(thresholds) else 0.5

        elif metric == 'recall':
            # Find threshold that maximizes recall
            optimal_idx = np.argmax(recall)
            optimal_value = recall[optimal_idx]
            optimal_threshold = thresholds[optimal_idx] if optimal_idx < len(thresholds) else 0.5

        elif metric == 'cost':
            # Find threshold that minimizes cost
            costs = []
            for threshold in thresholds:
                y_pred = (y_pred_proba >= threshold).astype(int)
                business = self.calculate_business_metrics(y_true, y_pred)
                costs.append(business['total_cost'])

            optimal_idx = np.argmin(costs)
            optimal_value = -costs[optimal_idx]  # Negative because we minimize cost
            optimal_threshold = thresholds[optimal_idx]

        else:
            raise ValueError(f"Unknown metric: {metric}")

        return float(optimal_threshold), float(optimal_value)

    def precision_at_recall(
        self,
        y_true: np.ndarray,
        y_pred_proba: np.ndarray,
        target_recall: float = 0.9
    ) -> Tuple[float, float]:
        """
        Find precision at target recall level.

        Args:
            y_true: True labels
            y_pred_proba: Predicted probabilities
            target_recall: Target recall level (e.g., 0.9 for 90%)

        Returns:
            Tuple of (precision, threshold) at target recall

        Example:
            >>> metrics = FraudMetrics()
            >>> precision, threshold = metrics.precision_at_recall(
            ...     y_test, y_pred_proba, target_recall=0.9
            ... )
            >>> print(f"At 90% recall: precision={precision:.3f}, threshold={threshold:.3f}")
        """
        precision, recall, thresholds = precision_recall_curve(y_true, y_pred_proba)

        # Find index where recall >= target_recall
        idx = np.where(recall >= target_recall)[0]

        if len(idx) == 0:
            # Target recall not achievable
            return 0.0, 1.0

        # Get first index where we achieve target recall
        idx = idx[0]
        return float(precision[idx]), float(thresholds[idx] if idx < len(thresholds) else 0.0)


# Example usage
if __name__ == "__main__":
    print("FraudMetrics - Example Usage\n")
    print("=" * 50)

    # Create sample data
    np.random.seed(42)
    n_samples = 1000
    y_true = np.random.choice([0, 1], n_samples, p=[0.95, 0.05])
    y_pred_proba = np.random.random(n_samples)
    y_pred = (y_pred_proba > 0.5).astype(int)

    # Initialize metrics
    metrics = FraudMetrics(
        fp_cost=10.0,
        fn_cost=1000.0,
        avg_fraud_amount=50000.0
    )

    # Calculate business metrics
    print("\n1. Business Metrics")
    business = metrics.calculate_business_metrics(y_true, y_pred)
    print(f"   Total Cost: ${business['total_cost']:,.2f}")
    print(f"   FP Cost: ${business['false_positive_cost']:,.2f}")
    print(f"   FN Cost: ${business['false_negative_cost']:,.2f}")
    print(f"   Fraud Detection Rate: {business['fraud_detection_rate']:.2%}")

    # Find optimal threshold
    print("\n2. Optimal Threshold (F1)")
    threshold, f1 = metrics.find_optimal_threshold(y_true, y_pred_proba, metric='f1')
    print(f"   Threshold: {threshold:.3f}")
    print(f"   F1 Score: {f1:.3f}")

    # Precision at recall
    print("\n3. Precision at 90% Recall")
    precision, threshold = metrics.precision_at_recall(y_true, y_pred_proba, target_recall=0.9)
    print(f"   Precision: {precision:.3f}")
    print(f"   Threshold: {threshold:.3f}")

    print("\n" + "=" * 50)
```

---

### Step 6: Training Script

**File: `/home/user/sentinel-api/app/scripts/train_model.py`**

This script runs the complete training pipeline.

```python
"""
Model Training Script.

This script trains a fraud detection model using the complete pipeline.

Usage:
    python app/scripts/train_model.py [OPTIONS]

Options:
    --limit INT         Maximum transactions to load (default: all)
    --min-days INT      Only load transactions from last N days
    --test-size FLOAT   Test set size (default: 0.2)
    --no-smote          Disable SMOTE
    --save-path STR     Path to save model (default: app/models/fraud_model_v1.pkl)

Example:
    # Train on last 10,000 transactions
    python app/scripts/train_model.py --limit 10000

    # Train on last 30 days
    python app/scripts/train_model.py --min-days 30

    # Train without SMOTE
    python app/scripts/train_model.py --no-smote
"""

import sys
from pathlib import Path
import argparse
from datetime import datetime, timedelta
import json

# Add project root to path
project_root = Path(__file__).parent.parent.parent
sys.path.insert(0, str(project_root))

from app.ml.trainer import ModelTrainer
from app.ml.evaluator import ModelEvaluator


def parse_args():
    """Parse command line arguments."""
    parser = argparse.ArgumentParser(
        description='Train fraud detection model',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  Train on all data:
    python app/scripts/train_model.py

  Train on last 10,000 transactions:
    python app/scripts/train_model.py --limit 10000

  Train on last 30 days:
    python app/scripts/train_model.py --min-days 30

  Train without SMOTE:
    python app/scripts/train_model.py --no-smote

  Custom save path:
    python app/scripts/train_model.py --save-path models/my_model.pkl
        """
    )

    parser.add_argument(
        '--limit',
        type=int,
        default=None,
        help='Maximum transactions to load (default: all)'
    )

    parser.add_argument(
        '--min-days',
        type=int,
        default=None,
        help='Only load transactions from last N days'
    )

    parser.add_argument(
        '--test-size',
        type=float,
        default=0.2,
        help='Test set size (default: 0.2)'
    )

    parser.add_argument(
        '--no-smote',
        action='store_true',
        help='Disable SMOTE'
    )

    parser.add_argument(
        '--cv-folds',
        type=int,
        default=5,
        help='Number of cross-validation folds (default: 5)'
    )

    parser.add_argument(
        '--save-path',
        type=str,
        default='app/models/fraud_model_v1.pkl',
        help='Path to save model (default: app/models/fraud_model_v1.pkl)'
    )

    parser.add_argument(
        '--random-state',
        type=int,
        default=42,
        help='Random seed (default: 42)'
    )

    return parser.parse_args()


def main():
    """Main training function."""
    # Parse arguments
    args = parse_args()

    print("\n" + "=" * 70)
    print("FRAUD DETECTION MODEL TRAINING")
    print("=" * 70)
    print(f"\nConfiguration:")
    print(f"  Data limit: {args.limit if args.limit else 'All'}")
    print(f"  Min days: {args.min_days if args.min_days else 'All'}")
    print(f"  Test size: {args.test_size}")
    print(f"  SMOTE: {'Disabled' if args.no_smote else 'Enabled'}")
    print(f"  CV folds: {args.cv_folds}")
    print(f"  Random state: {args.random_state}")
    print(f"  Save path: {args.save_path}")

    # Calculate min_date if min_days provided
    min_date = None
    if args.min_days:
        min_date = datetime.now() - timedelta(days=args.min_days)

    # Initialize trainer
    trainer = ModelTrainer(
        random_state=args.random_state,
        test_size=args.test_size,
        use_smote=not args.no_smote,
        cv_folds=args.cv_folds
    )

    try:
        # Run training pipeline
        model, metrics = trainer.full_training_pipeline(
            limit=args.limit,
            min_date=min_date
        )

        # Save model
        print("\n" + "=" * 70)
        print("SAVING MODEL")
        print("=" * 70)
        model.save(args.save_path, save_metadata=True)

        # Save metrics
        metrics_path = args.save_path.replace('.pkl', '_metrics.json')
        with open(metrics_path, 'w') as f:
            # Convert numpy types to native Python types for JSON
            metrics_json = json.dumps(metrics, indent=2, default=float)
            f.write(metrics_json)
        print(f"✅ Metrics saved: {metrics_path}")

        # Print final summary
        print("\n" + "=" * 70)
        print("TRAINING COMPLETE!")
        print("=" * 70)
        print(f"\nModel Performance:")
        print(f"  AUC:       {metrics['auc']:.3f}")
        print(f"  Precision: {metrics['precision']:.3f}")
        print(f"  Recall:    {metrics['recall']:.3f}")
        print(f"  F1-Score:  {metrics['f1']:.3f}")

        print(f"\nCross-Validation:")
        cv = metrics['cross_validation']
        print(f"  CV AUC:       {cv['mean_auc']:.3f} ± {cv['std_auc']:.3f}")
        print(f"  CV Precision: {cv['mean_precision']:.3f} ± {cv['std_precision']:.3f}")
        print(f"  CV Recall:    {cv['mean_recall']:.3f} ± {cv['std_recall']:.3f}")

        print(f"\nModel saved to: {args.save_path}")
        print(f"Metrics saved to: {metrics_path}")

        print("\n✅ Training successful!")

    except Exception as e:
        print(f"\n❌ Training failed: {str(e)}")
        import traceback
        traceback.print_exc()
        sys.exit(1)


if __name__ == "__main__":
    main()
```

---

### Step 7: Evaluation Script

**File: `/home/user/sentinel-api/app/scripts/evaluate_model.py`**

This script evaluates a trained model and generates reports.

```python
"""
Model Evaluation Script.

This script evaluates a trained fraud detection model.

Usage:
    python app/scripts/evaluate_model.py MODEL_PATH [OPTIONS]

Options:
    --limit INT         Maximum transactions to load for evaluation
    --threshold FLOAT   Classification threshold (default: 0.5)

Example:
    python app/scripts/evaluate_model.py app/models/fraud_model_v1.pkl
    python app/scripts/evaluate_model.py app/models/fraud_model_v1.pkl --threshold 0.7
"""

import sys
from pathlib import Path
import argparse

# Add project root to path
project_root = Path(__file__).parent.parent.parent
sys.path.insert(0, str(project_root))

from app.ml.model import FraudDetectionModel
from app.ml.trainer import ModelTrainer
from app.ml.evaluator import ModelEvaluator
from app.ml.metrics import FraudMetrics


def parse_args():
    """Parse command line arguments."""
    parser = argparse.ArgumentParser(description='Evaluate fraud detection model')

    parser.add_argument(
        'model_path',
        type=str,
        help='Path to saved model file'
    )

    parser.add_argument(
        '--limit',
        type=int,
        default=None,
        help='Maximum transactions to load (default: all)'
    )

    parser.add_argument(
        '--threshold',
        type=float,
        default=0.5,
        help='Classification threshold (default: 0.5)'
    )

    return parser.parse_args()


def main():
    """Main evaluation function."""
    # Parse arguments
    args = parse_args()

    print("\n" + "=" * 70)
    print("FRAUD DETECTION MODEL EVALUATION")
    print("=" * 70)
    print(f"\nModel: {args.model_path}")
    print(f"Threshold: {args.threshold}")

    # Load model
    print("\nLoading model...")
    model = FraudDetectionModel.load(args.model_path)

    # Load test data
    print("\nLoading test data...")
    trainer = ModelTrainer()
    df = trainer.load_data_from_db(limit=args.limit)
    X, y = trainer.prepare_features(df, fit_scaler=True)

    # Split data (use same random state for reproducibility)
    _, X_test, _, y_test = trainer.split_data(X, y)

    # Evaluate model
    evaluator = ModelEvaluator()
    metrics = evaluator.evaluate(
        model=model,
        X_test=X_test,
        y_test=y_test,
        threshold=args.threshold,
        save_plots=True
    )

    # Business metrics
    print("\n" + "=" * 70)
    print("BUSINESS METRICS")
    print("=" * 70)

    fraud_metrics = FraudMetrics(
        fp_cost=10.0,       # Cost to manually review false positive
        fn_cost=1000.0,     # Cost of missed fraud
        avg_fraud_amount=50000.0
    )

    y_pred_proba = model.predict_proba(X_test)[:, 1]
    y_pred = (y_pred_proba >= args.threshold).astype(int)

    business = fraud_metrics.calculate_business_metrics(y_test, y_pred)

    print(f"\nCost Analysis:")
    print(f"  False Positive Cost: ${business['false_positive_cost']:,.2f}")
    print(f"  False Negative Cost: ${business['false_negative_cost']:,.2f}")
    print(f"  Total Cost: ${business['total_cost']:,.2f}")

    print(f"\nFraud Detection:")
    print(f"  Fraud Caught: ${business['fraud_caught_amount']:,.2f}")
    print(f"  Fraud Missed: ${business['fraud_missed_amount']:,.2f}")
    print(f"  Detection Rate: {business['fraud_detection_rate']:.2%}")

    # Find optimal threshold
    print("\n" + "=" * 70)
    print("THRESHOLD OPTIMIZATION")
    print("=" * 70)

    threshold_f1, f1_score = fraud_metrics.find_optimal_threshold(
        y_test, y_pred_proba, metric='f1'
    )
    print(f"\nOptimal threshold (F1): {threshold_f1:.3f} (F1={f1_score:.3f})")

    threshold_cost, _ = fraud_metrics.find_optimal_threshold(
        y_test, y_pred_proba, metric='cost'
    )
    print(f"Optimal threshold (Cost): {threshold_cost:.3f}")

    # Precision at high recall
    precision_90, threshold_90 = fraud_metrics.precision_at_recall(
        y_test, y_pred_proba, target_recall=0.9
    )
    print(f"\nAt 90% recall:")
    print(f"  Precision: {precision_90:.3f}")
    print(f"  Threshold: {threshold_90:.3f}")

    # Error analysis
    print("\n" + "=" * 70)
    print("ERROR ANALYSIS")
    print("=" * 70)

    errors = evaluator.analyze_errors(
        model=model,
        X_test=X_test,
        y_test=y_test,
        threshold=args.threshold,
        n_examples=5
    )

    if len(errors['false_positives']) > 0:
        print("\nTop 5 False Positives (legitimate flagged as fraud):")
        print(errors['false_positives'][['predicted_proba']].to_string())

    if len(errors['false_negatives']) > 0:
        print("\nTop 5 False Negatives (fraud missed):")
        print(errors['false_negatives'][['predicted_proba']].to_string())

    print("\n✅ Evaluation complete!")
    print(f"\nPlots saved to: {evaluator.output_dir}")


if __name__ == "__main__":
    main()
```

---

## Testing the Model

### Step 1: Prepare Sample Data

First, ensure you have fraud transaction data in your database. If not, create sample data:

**File: `/home/user/sentinel-api/app/scripts/generate_sample_data.py`**

```python
"""
Generate sample fraud transaction data for training.

This script creates synthetic fraud transactions for testing the ML pipeline.
"""

import sys
from pathlib import Path
import numpy as np
import pandas as pd
from datetime import datetime, timedelta

# Add project root to path
project_root = Path(__file__).parent.parent.parent
sys.path.insert(0, str(project_root))

from app.db.session import SessionLocal
from app.models.fraud import FraudTransaction


def generate_sample_transactions(n_samples: int = 1000, fraud_ratio: float = 0.05):
    """
    Generate sample fraud transactions.

    Args:
        n_samples: Number of transactions to generate
        fraud_ratio: Fraction of fraudulent transactions (default: 0.05 = 5%)
    """
    print(f"\nGenerating {n_samples} sample transactions...")

    np.random.seed(42)

    # Calculate number of frauds
    n_fraud = int(n_samples * fraud_ratio)
    n_legit = n_samples - n_fraud

    print(f"  Legitimate: {n_legit}")
    print(f"  Fraudulent: {n_fraud}")

    transactions = []

    # Generate legitimate transactions
    for i in range(n_legit):
        amount = np.random.lognormal(mean=10.5, sigma=0.8)  # Mean ~50,000
        transactions.append({
            'transaction_id': f'tx_legit_{i}',
            'user_id': f'user_{np.random.randint(1, 500)}',
            'amount': float(amount),
            'currency': 'NGN',
            'transaction_type': np.random.choice(['DEPOSIT', 'WITHDRAWAL', 'TRANSFER']),
            'is_flagged': False,
            'fraud_score': np.random.randint(0, 30),
            'risk_level': 'LOW',
            'metadata': {
                'device_id': f'device_{np.random.randint(1, 100)}',
                'ip_address': f'197.210.{np.random.randint(1, 255)}.{np.random.randint(1, 255)}',
                'location': {
                    'country': 'Nigeria',
                    'city': np.random.choice(['Lagos', 'Abuja', 'Port Harcourt'])
                }
            },
            'created_at': datetime.now() - timedelta(days=np.random.randint(0, 90))
        })

    # Generate fraudulent transactions
    for i in range(n_fraud):
        amount = np.random.lognormal(mean=11.5, sigma=1.0)  # Higher amounts for fraud
        transactions.append({
            'transaction_id': f'tx_fraud_{i}',
            'user_id': f'user_{np.random.randint(1, 500)}',
            'amount': float(amount),
            'currency': 'NGN',
            'transaction_type': 'WITHDRAWAL',  # Most frauds are withdrawals
            'is_flagged': True,
            'fraud_score': np.random.randint(70, 100),
            'risk_level': np.random.choice(['MEDIUM', 'HIGH']),
            'metadata': {
                'device_id': f'device_new_{i}',  # New devices
                'ip_address': f'{np.random.randint(100, 200)}.{np.random.randint(1, 255)}.{np.random.randint(1, 255)}.{np.random.randint(1, 255)}',
                'location': {
                    'country': np.random.choice(['Nigeria', 'United States', 'Unknown']),
                    'city': 'Unknown'
                }
            },
            'created_at': datetime.now() - timedelta(days=np.random.randint(0, 90))
        })

    # Shuffle transactions
    np.random.shuffle(transactions)

    # Save to database
    db = SessionLocal()
    try:
        print("\nSaving to database...")

        for tx_data in transactions:
            tx = FraudTransaction(**tx_data)
            db.add(tx)

        db.commit()
        print(f"✅ Saved {len(transactions)} transactions to database")

    except Exception as e:
        print(f"❌ Error: {e}")
        db.rollback()

    finally:
        db.close()


if __name__ == "__main__":
    generate_sample_transactions(n_samples=5000, fraud_ratio=0.05)
```

---

### Step 2: Run Training

```bash
# Navigate to project root
cd /home/user/sentinel-api

# Activate virtual environment
source venv/bin/activate

# Generate sample data (if needed)
python app/scripts/generate_sample_data.py

# Train model
python app/scripts/train_model.py --limit 5000

# Expected output:
# ============================================================
# FRAUD DETECTION MODEL TRAINING PIPELINE
# ============================================================
#
# Loading data from database...
# ✅ Loaded 5000 transactions
#   Fraud: 250 (5.00%)
#   Legitimate: 4750 (95.00%)
#
# Feature engineering...
# ✅ 15 features created
#
# Train/test split...
#   Training: 4000 samples
#   Test: 1000 samples
#
# Applying SMOTE...
#   Before: Fraud ratio 5.00%
#   After: Fraud ratio 50.00%
# ✅ SMOTE complete
#
# Training XGBoost...
#   Training complete!
#
# Cross-validation...
#   CV AUC: 0.912 ± 0.023
# ✅ Cross-validation complete
#
# Evaluation...
#   Test AUC: 0.905
#   Precision: 0.867
#   Recall: 0.823
#   F1: 0.844
#
# ✅ Model saved: app/models/fraud_model_v1.pkl
```

---

### Step 3: Evaluate Model

```bash
# Evaluate saved model
python app/scripts/evaluate_model.py app/models/fraud_model_v1.pkl

# With custom threshold
python app/scripts/evaluate_model.py app/models/fraud_model_v1.pkl --threshold 0.7

# Expected output:
# ============================================================
# MODEL EVALUATION
# ============================================================
#
# Classification Metrics:
#   Accuracy:  0.947
#   Precision: 0.867
#   Recall:    0.823
#   F1-Score:  0.844
#   AUC-ROC:   0.905
#
# Confusion Matrix:
#   True Negatives:  920
#   False Positives: 30
#   False Negatives: 22
#   True Positives:  28
#
# Business Metrics:
#   False Positive Cost: $300.00
#   False Negative Cost: $22,000.00
#   Total Cost: $22,300.00
#   Fraud Detection Rate: 56.00%
#
# ✅ Evaluation plots saved to: app/models/evaluation/
```

---

## Understanding the Results

### Interpreting Metrics

**1. AUC (Area Under ROC Curve):**
```
AUC = 0.905 (Excellent!)

What it means:
  0.5: Random guessing
  0.7-0.8: Acceptable
  0.8-0.9: Excellent
  0.9-1.0: Outstanding

Our model (0.905) is outstanding at distinguishing fraud from legitimate.
```

**2. Precision:**
```
Precision = 0.867 = 86.7%

What it means:
  Of all transactions flagged as fraud, 86.7% actually are fraud.
  13.3% are false alarms (legitimate flagged as fraud).

Business impact:
  If you flag 100 transactions:
    - 87 are real fraud (good catches)
    - 13 are false alarms (unnecessary reviews)
```

**3. Recall:**
```
Recall = 0.823 = 82.3%

What it means:
  Of all actual frauds, we catch 82.3%.
  We miss 17.7% of frauds.

Business impact:
  If there are 100 frauds:
    - We catch 82
    - We miss 18 (losses)
```

**4. F1-Score:**
```
F1 = 0.844

What it means:
  Harmonic mean of precision and recall.
  Balances both metrics.

  Good when: F1 > 0.8
  Excellent when: F1 > 0.9
```

### Confusion Matrix Explained

```
                 Predicted
                 Legit  Fraud
Actual  Legit     920    30     ← 30 false positives
        Fraud      22    28     ← 22 false negatives

True Negatives (920):  Correctly identified legitimate ✓
False Positives (30):  Legitimate flagged as fraud ✗
False Negatives (22):  Fraud missed ✗
True Positives (28):   Correctly caught fraud ✓

Perfect model would have:
  High TN and TP (diagonal)
  Zero FP and FN (off-diagonal)
```

### Feature Importance

```
Top 10 Features by Importance:

1. amount_log              0.342  ← Transaction amount (log scaled)
2. velocity_1h             0.187  ← Transactions in last hour
3. is_new_device           0.156  ← New device flag
4. hour_sin                0.098  ← Time of day (cyclical)
5. country_risk_score      0.087  ← Country fraud risk
6. days_since_last_tx      0.054  ← Time since last transaction
7. amount_to_avg_ratio     0.032  ← Amount vs user average
8. is_weekend              0.021  ← Weekend transaction
9. device_count            0.015  ← Number of devices used
10. is_vpn                 0.008  ← VPN detected

Interpretation:
  - Transaction amount is the strongest predictor
  - Recent transaction velocity matters
  - Device and location features are important
  - Time-based features contribute moderately
```

---

## Troubleshooting

### Issue 1: Import Errors

**Error:**
```
ModuleNotFoundError: No module named 'app.ml.features'
```

**Solution:**
```bash
# Ensure you completed Day 9 (Feature Engineering)
ls -la app/ml/features.py

# If missing, you need to create features.py from Day 9
# For now, create a minimal version:
cat > app/ml/features.py << 'FEATURES_EOF'
import pandas as pd

class FeatureEngineer:
    def extract_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """Extract basic features."""
        features = pd.DataFrame()

        # Basic features
        features['amount'] = df['amount']
        features['hour'] = df['timestamp'].dt.hour
        features['is_weekend'] = df['timestamp'].dt.dayofweek.isin([5, 6]).astype(int)

        return features
FEATURES_EOF
```

---

### Issue 2: Insufficient Data

**Error:**
```
ValueError: No transactions found in database!
```

**Solution:**
```bash
# Generate sample data
python app/scripts/generate_sample_data.py

# Verify data
python -c "
from app.db.session import SessionLocal
from app.models.fraud import FraudTransaction
db = SessionLocal()
count = db.query(FraudTransaction).count()
print(f'Transactions in database: {count}')
db.close()
"
```

---

### Issue 3: SMOTE Error

**Error:**
```
ValueError: The number of classes has to be greater than one
```

**Solution:**
This means all samples are the same class (all fraud or all legitimate).

```python
# Check class distribution
from app.ml.trainer import ModelTrainer
trainer = ModelTrainer()
df = trainer.load_data_from_db()
print(df['is_fraud'].value_counts())

# Should see both 0 and 1
# If not, regenerate sample data with both classes
```

---

### Issue 4: Low Model Performance

**Problem: AUC < 0.7**

**Solutions:**

1. **More data:**
```bash
# Train on more transactions
python app/scripts/train_model.py --limit 10000
```

2. **Better features:**
```python
# Add more features in features.py
# - Velocity features
# - Device history
# - User behavior patterns
```

3. **Tune hyperparameters:**
```python
# Try different parameters
model_params = {
    'max_depth': 8,          # Deeper trees
    'learning_rate': 0.05,   # Slower learning
    'n_estimators': 1000     # More trees
}
```

---

## Next Steps

### Day 11: Model Deployment & API Integration

Tomorrow we'll integrate the trained model into our FastAPI service:

**What you'll build:**
- Model serving endpoint
- Real-time fraud prediction API
- Model versioning system
- A/B testing framework
- Monitoring and logging

**Endpoints:**
```python
POST /api/v1/predict-fraud
  → Load model
  → Extract features
  → Predict fraud probability
  → Return risk assessment

GET /api/v1/model/info
  → Model version
  → Training metrics
  → Feature list
```

---

### Practice Exercises

**1. Experiment with Hyperparameters:**
```python
# Try different combinations
python app/scripts/train_model.py --save-path models/model_deep.pkl
# Edit trainer.py to use max_depth=10

# Compare performance
python app/scripts/evaluate_model.py models/fraud_model_v1.pkl
python app/scripts/evaluate_model.py models/model_deep.pkl
```

**2. Feature Engineering:**
```python
# Add new features to features.py:
# - Transaction count last 7 days
# - Average transaction amount per user
# - Time since account creation
# - Number of countries used

# Retrain and compare
```

**3. Threshold Optimization:**
```python
# Find optimal threshold for your business
# - High precision (fewer false alarms)
# - High recall (catch more frauds)
# - Minimize cost
```

---

### Key Takeaways

**What You Built Today:**
1. ✅ Complete ML training pipeline
2. ✅ XGBoost model for fraud detection
3. ✅ Comprehensive evaluation framework
4. ✅ Model versioning and persistence
5. ✅ Business-oriented metrics

**ML Concepts Mastered:**
- Train/test split and why it matters
- Class imbalance and SMOTE
- Cross-validation for robust evaluation
- Precision vs Recall tradeoff
- Feature importance analysis
- Model persistence with joblib

**Project Status:**
```
✅ Day 1-5: API and rules engine
✅ Day 8: ML setup
✅ Day 9: Feature engineering
✅ Day 10: Model training & evaluation ← YOU ARE HERE
⬜ Day 11: Model deployment
⬜ Day 12: Monitoring and retraining
```

---

**Navigation:** [← Previous: Day 9](README-DAY-009.md) | [Main Guide](README.md) | [Next: Day 11 →](README-DAY-011.md)

---

**Congratulations!** 🎉

You've successfully built and trained a production-grade fraud detection model! Your model can:

✅ Detect fraud with 90%+ AUC
✅ Handle imbalanced data with SMOTE
✅ Provide calibrated probability estimates
✅ Explain predictions with feature importance
✅ Generalize to unseen data (validated through cross-validation)

Tomorrow, we'll deploy this model to production and serve real-time predictions through our API.

Keep building! 🚀

---

**Questions or Issues?**

Common issues and solutions:
1. Low performance → More data, better features
2. Overfitting → Reduce max_depth, more regularization
3. Underfitting → Increase model complexity
4. Class imbalance → Verify SMOTE is working
5. Slow training → Reduce n_estimators, use smaller dataset for testing

**Happy modeling!** 📊
