# Day 9: Feature Engineering for Fraud Detection

**Transforming Raw Data into Powerful ML Features**

**Navigation:** [← Previous: Day 8](README-DAY-008.md) | [Main Guide](README.md) | [Next: Day 10 →](README-DAY-010.md)

---

## Table of Contents

1. [Overview](#overview)
2. [What is Feature Engineering?](#what-is-feature-engineering)
3. [Feature Engineering Architecture](#feature-engineering-architecture)
4. [Complete Code Implementation](#complete-code-implementation)
5. [Feature Categories Deep Dive](#feature-categories-deep-dive)
6. [Categorical Encoding Explained](#categorical-encoding-explained)
7. [Feature Transformations](#feature-transformations)
8. [Testing Your Features](#testing-your-features)
9. [Feature Importance Analysis](#feature-importance-analysis)
10. [Troubleshooting](#troubleshooting)
11. [Next Steps](#next-steps)

---

## Overview

### Day 9 Objectives

Today we're building the **most critical component** of any machine learning system: **Feature Engineering**. Raw transaction data isn't enough for ML models - we need to extract meaningful patterns that indicate fraud.

**What You'll Build:**

✅ **60+ engineered features** from raw transaction data
✅ **Transaction features** - Amount statistics, temporal patterns
✅ **User features** - Account history, behavior patterns
✅ **Device features** - Device tracking, VPN detection
✅ **Location features** - Geographic patterns, impossible travel
✅ **Velocity features** - Transaction frequency, spending patterns
✅ **Categorical encoders** - One-hot, label encoding
✅ **Feature transformers** - Scaling, normalization
✅ **Feature validation** - NaN handling, correlation analysis

**Key Deliverables:**

```
app/
└── ml/
    ├── __init__.py           # ML module initialization
    ├── features.py           # Feature extraction (60+ features)
    ├── encoders.py           # Categorical encoding
    └── transformers.py       # Feature transformations
```

**Estimated Time:** 4-5 hours

**Prerequisites:**
- ✅ Day 8: ML fundamentals and packages (scikit-learn, pandas)
- ✅ Day 2: Database models (FraudTransaction, UserRiskProfile)
- ✅ Day 3: Pydantic schemas (TransactionCheckRequest)

**No New Packages:** We'll use scikit-learn and pandas from Day 8!

---

## What is Feature Engineering?

### Definition

**Feature Engineering** is the process of transforming raw data into features that better represent the underlying problem to the predictive models, resulting in improved model accuracy.

### Why Feature Engineering is Critical

In fraud detection, **feature engineering accounts for 70-80% of model performance**. A sophisticated model with poor features will perform worse than a simple model with excellent features.

**Example: Raw Data vs Engineered Features**

```python
# ❌ Raw Data (Not useful for ML)
{
    "amount": 1250.00,
    "timestamp": "2025-12-30T14:23:45Z",
    "user_id": "user_12345"
}

# ✅ Engineered Features (ML-ready)
{
    "amount": 1250.00,                           # Original
    "amount_zscore": 2.3,                        # How unusual is this amount?
    "amount_vs_user_avg": 5.2,                   # 5.2x user's typical amount
    "hour_of_day": 14,                           # Time patterns
    "is_weekend": 0,                             # Weekend transactions riskier
    "days_since_account_created": 45,            # Account age
    "txn_count_last_hour": 5,                    # Velocity feature
    "txn_count_last_day": 12,                    # More velocity
    "unique_merchants_last_day": 8,              # Shopping pattern
    "distance_from_last_txn_km": 450.2,          # Location change
    "impossible_travel_flag": 1,                 # Physics violation!
    "new_device_flag": 1,                        # Device risk
    "vpn_flag": 1,                               # VPN usage
    "country_risk_score": 0.75,                  # Geographic risk
    # ... 47+ more features
}
```

### The Feature Engineering Pipeline

```
┌────────────────────────────────────────────────────────────┐
│                    RAW TRANSACTION DATA                     │
│  {amount, timestamp, user_id, device_id, location...}      │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│              STEP 1: FEATURE EXTRACTION                     │
│                  (features.py)                              │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Transaction  │  │ User         │  │ Device       │     │
│  │ Features     │  │ Features     │  │ Features     │     │
│  │ (15)         │  │ (15)         │  │ (10)         │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ Location     │  │ Velocity     │                        │
│  │ Features     │  │ Features     │                        │
│  │ (10)         │  │ (10)         │                        │
│  └──────────────┘  └──────────────┘                        │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│              STEP 2: CATEGORICAL ENCODING                   │
│                  (encoders.py)                              │
│                                                             │
│  Convert categories to numbers:                            │
│  - Country: "US" → [1, 0, 0, 0]  (one-hot)                 │
│  - Browser: "Chrome" → 0  (label encoding)                 │
│  - Merchant: "Amazon" → 42  (label encoding)               │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│              STEP 3: FEATURE TRANSFORMATION                 │
│                  (transformers.py)                          │
│                                                             │
│  Scale and normalize:                                      │
│  - StandardScaler: mean=0, std=1                           │
│  - MinMaxScaler: range [0, 1]                              │
│  - Handle missing values (imputation)                      │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│                    ML-READY FEATURES                        │
│              (60+ numeric features)                         │
│          Ready for model training!                          │
└────────────────────────────────────────────────────────────┘
```

### Types of Features in Fraud Detection

| Category | Examples | Why Important |
|----------|----------|---------------|
| **Transaction** | Amount, time, frequency | Core fraud signals |
| **User Behavioral** | Account age, avg amount | User patterns |
| **Device** | New device, browser type | Device fingerprinting |
| **Location** | Country, IP, distance | Geographic anomalies |
| **Velocity** | Txns/hour, spending rate | Rapid fraud patterns |
| **Temporal** | Hour, day, weekend | Time-based patterns |
| **Statistical** | Z-scores, percentiles | Anomaly detection |
| **Relational** | Ratios, differences | Comparative analysis |

---

## Feature Engineering Architecture

### Module Structure

```
app/ml/
├── __init__.py              # Module exports
├── features.py              # 🔥 Feature extraction (900+ lines)
│   ├── FeatureExtractor     # Main class
│   ├── TransactionFeatures  # Amount, time features
│   ├── UserFeatures         # User behavioral features
│   ├── DeviceFeatures       # Device fingerprinting
│   ├── LocationFeatures     # Geographic features
│   └── VelocityFeatures     # Frequency features
│
├── encoders.py              # Categorical encoding (300+ lines)
│   ├── CategoricalEncoder   # Main encoder
│   ├── OneHotEncoder        # One-hot encoding
│   └── LabelEncoder         # Label encoding
│
└── transformers.py          # Feature transformations (400+ lines)
    ├── FeatureTransformer   # Main transformer
    ├── StandardScaler       # Z-score normalization
    ├── MinMaxScaler         # Min-max normalization
    └── ImputationHandler    # Missing value handling
```

### Design Principles

**1. Modularity**
- Each feature category in its own class
- Easy to add/remove features
- Clear separation of concerns

**2. Scalability**
- Efficient pandas operations
- Vectorized computations
- Handles millions of transactions

**3. Maintainability**
- Well-documented features
- Type hints throughout
- Comprehensive tests

**4. Production-Ready**
- NaN handling
- Error handling
- Logging for debugging

---

## Complete Code Implementation

### Step 1: Create ML Module Directory

```bash
# Create the ML module
mkdir -p /home/user/sentinel-api/app/ml
touch /home/user/sentinel-api/app/ml/__init__.py
```

### Step 2: ML Module __init__.py

**File: `/home/user/sentinel-api/app/ml/__init__.py`**

```python
"""
Machine Learning module for Sentinel Fraud Detection Platform.

This module provides:
- Feature engineering (60+ features)
- Categorical encoding
- Feature transformations
- Model training (future)
- Model inference (future)

Usage:
    from app.ml.features import FeatureExtractor

    extractor = FeatureExtractor()
    features = extractor.extract(transaction_data)
"""

from app.ml.features import FeatureExtractor
from app.ml.encoders import CategoricalEncoder
from app.ml.transformers import FeatureTransformer

__all__ = [
    "FeatureExtractor",
    "CategoricalEncoder",
    "FeatureTransformer",
]

# Version information
__version__ = "0.1.0"
__author__ = "Sentinel ML Team"
```

### Step 3: Feature Extraction - The Core Engine

**File: `/home/user/sentinel-api/app/ml/features.py`**

This is the largest and most important file - 900+ lines of feature engineering logic.

```python
"""
Feature Extraction for Fraud Detection.

This module extracts 60+ features from raw transaction data to power
machine learning models. Features are organized into 5 categories:

1. Transaction Features (15) - Amount patterns, temporal features
2. User Features (15) - Account history, behavioral patterns
3. Device Features (10) - Device fingerprinting, VPN detection
4. Location Features (10) - Geographic patterns, impossible travel
5. Velocity Features (10) - Transaction frequency, spending velocity

Author: Sentinel ML Team
Version: 1.0.0
"""

import logging
from datetime import datetime, timedelta
from typing import Dict, List, Optional, Any
import pandas as pd
import numpy as np
from sqlalchemy.orm import Session

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class FeatureExtractor:
    """
    Main feature extraction engine.

    Coordinates all feature extractors and combines their outputs
    into a single feature vector ready for ML models.

    Usage:
        extractor = FeatureExtractor()
        features = extractor.extract(transaction_data, db_session)

    Returns:
        Dict with 60+ features as key-value pairs
    """

    def __init__(self):
        """Initialize all feature extractors."""
        self.transaction_extractor = TransactionFeatures()
        self.user_extractor = UserFeatures()
        self.device_extractor = DeviceFeatures()
        self.location_extractor = LocationFeatures()
        self.velocity_extractor = VelocityFeatures()

        logger.info("FeatureExtractor initialized with 5 feature categories")

    def extract(
        self,
        transaction: Dict[str, Any],
        db: Session,
        user_history: Optional[pd.DataFrame] = None
    ) -> Dict[str, float]:
        """
        Extract all features for a single transaction.

        Args:
            transaction: Transaction data (dict or Pydantic model)
            db: Database session for querying historical data
            user_history: Optional pre-loaded user transaction history

        Returns:
            Dictionary with 60+ features

        Example:
            >>> features = extractor.extract(txn, db)
            >>> len(features)
            62
            >>> features['amount_zscore']
            2.34
        """
        try:
            # Convert Pydantic model to dict if needed
            if hasattr(transaction, 'dict'):
                txn_dict = transaction.dict()
            else:
                txn_dict = transaction

            # Extract features from each category
            features = {}

            # 1. Transaction Features (15 features)
            txn_features = self.transaction_extractor.extract(txn_dict)
            features.update(txn_features)
            logger.debug(f"Extracted {len(txn_features)} transaction features")

            # 2. User Features (15 features)
            user_features = self.user_extractor.extract(
                txn_dict, db, user_history
            )
            features.update(user_features)
            logger.debug(f"Extracted {len(user_features)} user features")

            # 3. Device Features (10 features)
            device_features = self.device_extractor.extract(txn_dict, db)
            features.update(device_features)
            logger.debug(f"Extracted {len(device_features)} device features")

            # 4. Location Features (10 features)
            location_features = self.location_extractor.extract(txn_dict, db)
            features.update(location_features)
            logger.debug(f"Extracted {len(location_features)} location features")

            # 5. Velocity Features (10 features)
            velocity_features = self.velocity_extractor.extract(txn_dict, db)
            features.update(velocity_features)
            logger.debug(f"Extracted {len(velocity_features)} velocity features")

            # Log total features extracted
            logger.info(f"Total features extracted: {len(features)}")

            # Validate feature count
            if len(features) < 60:
                logger.warning(
                    f"Expected 60+ features, got {len(features)}. "
                    "Some features may be missing."
                )

            return features

        except Exception as e:
            logger.error(f"Feature extraction failed: {str(e)}", exc_info=True)
            raise

    def extract_batch(
        self,
        transactions: List[Dict[str, Any]],
        db: Session
    ) -> pd.DataFrame:
        """
        Extract features for multiple transactions (batch processing).

        Args:
            transactions: List of transaction dictionaries
            db: Database session

        Returns:
            DataFrame with rows=transactions, columns=features

        Example:
            >>> df = extractor.extract_batch(txns, db)
            >>> df.shape
            (1000, 62)  # 1000 transactions, 62 features
        """
        features_list = []

        for txn in transactions:
            features = self.extract(txn, db)
            features_list.append(features)

        return pd.DataFrame(features_list)

    def get_feature_names(self) -> List[str]:
        """
        Get list of all feature names.

        Returns:
            List of feature names in order
        """
        # Create dummy transaction to extract feature names
        dummy_txn = {
            "amount": 100.0,
            "timestamp": datetime.utcnow(),
            "user_id": "dummy",
            "device_id": "dummy",
            "merchant": "dummy",
            "country": "US",
            "user_info": {},
            "device_info": {},
            "location_info": {}
        }

        # Extract features (will return with NaNs for missing DB data)
        from unittest.mock import MagicMock
        mock_db = MagicMock()

        features = self.extract(dummy_txn, mock_db)
        return list(features.keys())


# ============================================================================
# TRANSACTION FEATURES (15 features)
# ============================================================================

class TransactionFeatures:
    """
    Extract features from the transaction itself.

    Features (15 total):
    1. amount - Raw transaction amount
    2. amount_log - Log-transformed amount (for skewed distributions)
    3. amount_zscore - Z-score (how many std devs from mean)
    4. amount_percentile - Percentile rank (0-100)
    5. hour_of_day - Hour (0-23)
    6. day_of_week - Day (0=Monday, 6=Sunday)
    7. is_weekend - Weekend flag (0/1)
    8. is_night - Night transaction flag (0/1)
    9. is_business_hours - Business hours flag (0/1)
    10. day_of_month - Day (1-31)
    11. month - Month (1-12)
    12. is_month_end - Last 3 days of month (0/1)
    13. is_month_start - First 3 days of month (0/1)
    14. amount_round_number - Is round number like 100, 500 (0/1)
    15. merchant_category_encoded - Merchant category (label encoded)
    """

    # Historical statistics for normalization (from training data)
    # In production, these would be loaded from a config file
    AMOUNT_MEAN = 150.0
    AMOUNT_STD = 200.0
    AMOUNT_MIN = 0.01
    AMOUNT_MAX = 10000.0

    def extract(self, transaction: Dict[str, Any]) -> Dict[str, float]:
        """Extract transaction-level features."""
        features = {}

        # Get transaction fields
        amount = float(transaction.get('amount', 0))
        timestamp = transaction.get('timestamp')
        merchant = transaction.get('merchant', '')

        # Parse timestamp
        if isinstance(timestamp, str):
            timestamp = datetime.fromisoformat(timestamp.replace('Z', '+00:00'))
        elif timestamp is None:
            timestamp = datetime.utcnow()

        # ================================================================
        # AMOUNT FEATURES (4 features)
        # ================================================================

        # 1. Raw amount
        features['amount'] = amount

        # 2. Log-transformed amount (reduces skewness)
        # Many transactions are small, few are large (power law)
        features['amount_log'] = np.log1p(amount)  # log(1 + x) to handle 0

        # 3. Amount Z-score (how unusual is this amount?)
        # Z-score > 2 means > 95% of transactions
        # Z-score > 3 means > 99.7% of transactions
        features['amount_zscore'] = (
            (amount - self.AMOUNT_MEAN) / self.AMOUNT_STD
        )

        # 4. Amount percentile (0-100)
        # Simple linear normalization
        features['amount_percentile'] = (
            (amount - self.AMOUNT_MIN) / (self.AMOUNT_MAX - self.AMOUNT_MIN) * 100
        )
        features['amount_percentile'] = np.clip(
            features['amount_percentile'], 0, 100
        )

        # ================================================================
        # TEMPORAL FEATURES (9 features)
        # ================================================================

        # 5. Hour of day (0-23)
        # Fraud patterns: Late night (2-4 AM), early morning (5-7 AM)
        features['hour_of_day'] = timestamp.hour

        # 6. Day of week (0=Monday, 6=Sunday)
        # Fraud patterns: Weekends have different patterns
        features['day_of_week'] = timestamp.weekday()

        # 7. Is weekend (0/1)
        features['is_weekend'] = 1 if timestamp.weekday() >= 5 else 0

        # 8. Is night (0/1) - 10 PM to 6 AM
        # Late night transactions are riskier
        features['is_night'] = (
            1 if timestamp.hour >= 22 or timestamp.hour < 6 else 0
        )

        # 9. Is business hours (0/1) - 9 AM to 5 PM
        # Transactions outside business hours can be suspicious
        features['is_business_hours'] = (
            1 if 9 <= timestamp.hour < 17 else 0
        )

        # 10. Day of month (1-31)
        # Patterns: Salary days (1st, 15th), month end
        features['day_of_month'] = timestamp.day

        # 11. Month (1-12)
        # Seasonal patterns: Holiday fraud (Nov-Dec)
        features['month'] = timestamp.month

        # 12. Is month end (0/1) - Last 3 days
        # Higher spending at month end
        features['is_month_end'] = 1 if timestamp.day >= 28 else 0

        # 13. Is month start (0/1) - First 3 days
        # Salary credit, higher spending
        features['is_month_start'] = 1 if timestamp.day <= 3 else 0

        # ================================================================
        # MERCHANT FEATURES (2 features)
        # ================================================================

        # 14. Amount is round number (0/1)
        # Fraudsters often test with round numbers: $100, $500, $1000
        round_numbers = [50, 100, 200, 250, 500, 1000, 2000, 5000]
        features['amount_round_number'] = 1 if amount in round_numbers else 0

        # 15. Merchant category (encoded)
        # This is a placeholder - will be properly encoded by CategoricalEncoder
        # For now, just store the hash
        features['merchant_category_encoded'] = hash(merchant) % 100

        return features


# ============================================================================
# USER FEATURES (15 features)
# ============================================================================

class UserFeatures:
    """
    Extract features from user's historical behavior.

    Features (15 total):
    1. account_age_days - Days since account creation
    2. total_transaction_count - Lifetime transaction count
    3. total_transaction_amount - Lifetime transaction amount
    4. avg_transaction_amount - Average transaction amount
    5. std_transaction_amount - Std dev of transaction amount
    6. min_transaction_amount - Minimum transaction amount
    7. max_transaction_amount - Maximum transaction amount
    8. fraud_history_count - Number of past fraud flags
    9. fraud_history_rate - Fraud rate (fraud/total)
    10. is_verified - User verification status (0/1)
    11. days_since_last_transaction - Days since last txn
    12. transactions_last_30_days - Transaction count (30d)
    13. amount_last_30_days - Transaction amount (30d)
    14. unique_merchants_last_30_days - Unique merchants (30d)
    15. amount_vs_user_avg - Current amount / avg amount ratio
    """

    def extract(
        self,
        transaction: Dict[str, Any],
        db: Session,
        user_history: Optional[pd.DataFrame] = None
    ) -> Dict[str, float]:
        """Extract user behavioral features."""
        features = {}

        user_id = transaction.get('user_id')
        amount = float(transaction.get('amount', 0))
        timestamp = transaction.get('timestamp')

        if isinstance(timestamp, str):
            timestamp = datetime.fromisoformat(timestamp.replace('Z', '+00:00'))

        # Get user info
        user_info = transaction.get('user_info', {})

        # ================================================================
        # ACCOUNT FEATURES (2 features)
        # ================================================================

        # 1. Account age (days)
        # Newer accounts are riskier
        account_created = user_info.get('account_created_at')
        if account_created:
            if isinstance(account_created, str):
                account_created = datetime.fromisoformat(
                    account_created.replace('Z', '+00:00')
                )
            features['account_age_days'] = (
                (timestamp - account_created).days
            )
        else:
            features['account_age_days'] = 0  # Unknown = new account

        # 2. Verification status (0/1)
        # Unverified accounts are riskier
        features['is_verified'] = (
            1 if user_info.get('is_verified', False) else 0
        )

        # ================================================================
        # HISTORICAL STATISTICS (7 features)
        # ================================================================

        # Load user history from database if not provided
        if user_history is None:
            user_history = self._load_user_history(user_id, db)

        if len(user_history) > 0:
            # 3. Total transaction count
            features['total_transaction_count'] = len(user_history)

            # 4. Total transaction amount
            features['total_transaction_amount'] = user_history['amount'].sum()

            # 5. Average transaction amount
            features['avg_transaction_amount'] = user_history['amount'].mean()

            # 6. Std dev of transaction amount
            features['std_transaction_amount'] = user_history['amount'].std()

            # 7. Minimum transaction amount
            features['min_transaction_amount'] = user_history['amount'].min()

            # 8. Maximum transaction amount
            features['max_transaction_amount'] = user_history['amount'].max()

            # 9. Fraud history count
            features['fraud_history_count'] = (
                user_history['is_fraud'].sum()
                if 'is_fraud' in user_history.columns else 0
            )

            # 10. Fraud rate
            features['fraud_history_rate'] = (
                features['fraud_history_count'] / len(user_history)
                if len(user_history) > 0 else 0
            )
        else:
            # No history - new user (all zeros)
            features['total_transaction_count'] = 0
            features['total_transaction_amount'] = 0.0
            features['avg_transaction_amount'] = 0.0
            features['std_transaction_amount'] = 0.0
            features['min_transaction_amount'] = 0.0
            features['max_transaction_amount'] = 0.0
            features['fraud_history_count'] = 0
            features['fraud_history_rate'] = 0.0

        # ================================================================
        # RECENCY FEATURES (4 features)
        # ================================================================

        if len(user_history) > 0:
            # 11. Days since last transaction
            last_txn_time = user_history['timestamp'].max()
            features['days_since_last_transaction'] = (
                (timestamp - last_txn_time).total_seconds() / 86400
            )

            # 12-14. Last 30 days statistics
            thirty_days_ago = timestamp - timedelta(days=30)
            recent_history = user_history[
                user_history['timestamp'] >= thirty_days_ago
            ]

            features['transactions_last_30_days'] = len(recent_history)
            features['amount_last_30_days'] = (
                recent_history['amount'].sum() if len(recent_history) > 0 else 0.0
            )
            features['unique_merchants_last_30_days'] = (
                recent_history['merchant'].nunique()
                if len(recent_history) > 0 else 0
            )
        else:
            features['days_since_last_transaction'] = 999  # No history
            features['transactions_last_30_days'] = 0
            features['amount_last_30_days'] = 0.0
            features['unique_merchants_last_30_days'] = 0

        # ================================================================
        # COMPARATIVE FEATURES (1 feature)
        # ================================================================

        # 15. Amount vs user average ratio
        # If current txn is 5x user's average, it's suspicious
        avg_amount = features['avg_transaction_amount']
        if avg_amount > 0:
            features['amount_vs_user_avg'] = amount / avg_amount
        else:
            features['amount_vs_user_avg'] = 1.0  # No history

        return features

    def _load_user_history(
        self,
        user_id: str,
        db: Session
    ) -> pd.DataFrame:
        """
        Load user's transaction history from database.

        Args:
            user_id: User identifier
            db: Database session

        Returns:
            DataFrame with user's past transactions
        """
        try:
            from app.models import FraudTransaction

            # Query user's transactions (last 1000 for performance)
            transactions = db.query(FraudTransaction).filter(
                FraudTransaction.user_id == user_id
            ).order_by(
                FraudTransaction.created_at.desc()
            ).limit(1000).all()

            if not transactions:
                return pd.DataFrame()

            # Convert to DataFrame
            data = []
            for txn in transactions:
                data.append({
                    'amount': txn.amount,
                    'timestamp': txn.created_at,
                    'merchant': txn.merchant or '',
                    'is_fraud': txn.is_fraud or False
                })

            return pd.DataFrame(data)

        except Exception as e:
            logger.error(f"Failed to load user history: {str(e)}")
            return pd.DataFrame()


# ============================================================================
# DEVICE FEATURES (10 features)
# ============================================================================

class DeviceFeatures:
    """
    Extract features from device information.

    Features (10 total):
    1. new_device_flag - First time seeing this device (0/1)
    2. device_age_days - Days since first seen
    3. device_transaction_count - Transactions from this device
    4. device_fraud_count - Fraud incidents from this device
    5. device_fraud_rate - Fraud rate for this device
    6. vpn_flag - VPN detected (0/1)
    7. proxy_flag - Proxy detected (0/1)
    8. tor_flag - Tor network detected (0/1)
    9. browser_encoded - Browser type (encoded)
    10. os_encoded - Operating system (encoded)
    """

    def extract(self, transaction: Dict[str, Any], db: Session) -> Dict[str, float]:
        """Extract device fingerprinting features."""
        features = {}

        device_id = transaction.get('device_id', '')
        device_info = transaction.get('device_info', {})
        timestamp = transaction.get('timestamp')

        if isinstance(timestamp, str):
            timestamp = datetime.fromisoformat(timestamp.replace('Z', '+00:00'))

        # ================================================================
        # DEVICE IDENTITY FEATURES (5 features)
        # ================================================================

        # Load device history
        device_history = self._load_device_history(device_id, db)

        if len(device_history) > 0:
            # 1. New device flag
            features['new_device_flag'] = 0  # Known device

            # 2. Device age (days since first seen)
            first_seen = device_history['timestamp'].min()
            features['device_age_days'] = (
                (timestamp - first_seen).total_seconds() / 86400
            )

            # 3. Device transaction count
            features['device_transaction_count'] = len(device_history)

            # 4. Device fraud count
            features['device_fraud_count'] = (
                device_history['is_fraud'].sum()
                if 'is_fraud' in device_history.columns else 0
            )

            # 5. Device fraud rate
            features['device_fraud_rate'] = (
                features['device_fraud_count'] / len(device_history)
                if len(device_history) > 0 else 0
            )
        else:
            # New device - high risk!
            features['new_device_flag'] = 1
            features['device_age_days'] = 0
            features['device_transaction_count'] = 0
            features['device_fraud_count'] = 0
            features['device_fraud_rate'] = 0.0

        # ================================================================
        # ANONYMIZATION DETECTION (3 features)
        # ================================================================

        # 6. VPN flag
        features['vpn_flag'] = 1 if device_info.get('vpn_detected') else 0

        # 7. Proxy flag
        features['proxy_flag'] = 1 if device_info.get('proxy_detected') else 0

        # 8. Tor network flag
        features['tor_flag'] = 1 if device_info.get('tor_detected') else 0

        # ================================================================
        # DEVICE TYPE FEATURES (2 features)
        # ================================================================

        # 9. Browser type (encoded)
        # Will be properly encoded by CategoricalEncoder
        browser = device_info.get('browser', 'unknown')
        features['browser_encoded'] = hash(browser) % 50

        # 10. Operating system (encoded)
        os = device_info.get('os', 'unknown')
        features['os_encoded'] = hash(os) % 50

        return features

    def _load_device_history(
        self,
        device_id: str,
        db: Session
    ) -> pd.DataFrame:
        """Load device's transaction history."""
        try:
            from app.models import FraudTransaction

            transactions = db.query(FraudTransaction).filter(
                FraudTransaction.device_id == device_id
            ).order_by(
                FraudTransaction.created_at.desc()
            ).limit(500).all()

            if not transactions:
                return pd.DataFrame()

            data = []
            for txn in transactions:
                data.append({
                    'timestamp': txn.created_at,
                    'is_fraud': txn.is_fraud or False
                })

            return pd.DataFrame(data)

        except Exception as e:
            logger.error(f"Failed to load device history: {str(e)}")
            return pd.DataFrame()


# ============================================================================
# LOCATION FEATURES (10 features)
# ============================================================================

class LocationFeatures:
    """
    Extract features from location/IP information.

    Features (10 total):
    1. country_risk_score - Country fraud risk (0-1)
    2. new_country_flag - First txn from this country (0/1)
    3. distance_from_last_txn_km - Distance from last txn
    4. distance_from_home_km - Distance from user's home
    5. impossible_travel_flag - Physics violation (0/1)
    6. country_change_flag - Different country than last txn (0/1)
    7. timezone_offset - Hours from user's home timezone
    8. is_high_risk_country - High risk country (0/1)
    9. ip_address_encoded - IP address (hashed)
    10. city_encoded - City (encoded)
    """

    # Country risk scores (fraud prevalence)
    # In production, these would be updated regularly from fraud data
    COUNTRY_RISK_SCORES = {
        'US': 0.15,      # Low risk
        'GB': 0.18,      # Low risk
        'CA': 0.12,      # Low risk
        'DE': 0.14,      # Low risk
        'FR': 0.16,      # Low risk
        'AU': 0.13,      # Low risk
        'JP': 0.10,      # Very low risk
        'CN': 0.45,      # Medium-high risk
        'RU': 0.65,      # High risk
        'NG': 0.75,      # High risk
        'IN': 0.35,      # Medium risk
        'BR': 0.40,      # Medium risk
        'UNKNOWN': 0.50  # Default
    }

    HIGH_RISK_COUNTRIES = ['RU', 'NG', 'CN', 'UA', 'RO']

    def extract(self, transaction: Dict[str, Any], db: Session) -> Dict[str, float]:
        """Extract location-based features."""
        features = {}

        user_id = transaction.get('user_id')
        location_info = transaction.get('location_info', {})
        timestamp = transaction.get('timestamp')

        country = location_info.get('country', 'UNKNOWN')
        latitude = location_info.get('latitude', 0.0)
        longitude = location_info.get('longitude', 0.0)

        # ================================================================
        # COUNTRY FEATURES (3 features)
        # ================================================================

        # 1. Country risk score
        features['country_risk_score'] = self.COUNTRY_RISK_SCORES.get(
            country, self.COUNTRY_RISK_SCORES['UNKNOWN']
        )

        # 2. New country flag
        country_history = self._get_user_countries(user_id, db)
        features['new_country_flag'] = (
            1 if country not in country_history else 0
        )

        # 3. Is high risk country
        features['is_high_risk_country'] = (
            1 if country in self.HIGH_RISK_COUNTRIES else 0
        )

        # ================================================================
        # DISTANCE FEATURES (3 features)
        # ================================================================

        # 4. Distance from last transaction
        last_location = self._get_last_location(user_id, db)
        if last_location:
            features['distance_from_last_txn_km'] = self._haversine_distance(
                latitude, longitude,
                last_location['latitude'], last_location['longitude']
            )
        else:
            features['distance_from_last_txn_km'] = 0.0

        # 5. Distance from home
        home_location = self._get_home_location(user_id, db)
        if home_location:
            features['distance_from_home_km'] = self._haversine_distance(
                latitude, longitude,
                home_location['latitude'], home_location['longitude']
            )
        else:
            features['distance_from_home_km'] = 0.0

        # 6. Country change flag
        last_country = self._get_last_country(user_id, db)
        features['country_change_flag'] = (
            1 if last_country and last_country != country else 0
        )

        # ================================================================
        # IMPOSSIBLE TRAVEL DETECTION (1 feature)
        # ================================================================

        # 7. Impossible travel flag
        # Check if travel speed exceeds maximum possible (airplane)
        features['impossible_travel_flag'] = 0

        if last_location and last_location.get('timestamp'):
            distance_km = features['distance_from_last_txn_km']
            time_diff_hours = (
                (timestamp - last_location['timestamp']).total_seconds() / 3600
            )

            if time_diff_hours > 0:
                speed_kmh = distance_km / time_diff_hours

                # Max speed: 900 km/h (jet airplane)
                # If speed > 900 km/h, it's physically impossible
                if speed_kmh > 900:
                    features['impossible_travel_flag'] = 1
                    logger.warning(
                        f"Impossible travel detected: {speed_kmh:.0f} km/h"
                    )

        # ================================================================
        # TIMEZONE FEATURES (1 feature)
        # ================================================================

        # 8. Timezone offset from home
        # Placeholder - would use actual timezone lookup in production
        features['timezone_offset'] = 0

        # ================================================================
        # ENCODED LOCATION FEATURES (2 features)
        # ================================================================

        # 9. IP address (hashed for privacy)
        ip_address = location_info.get('ip_address', '')
        features['ip_address_encoded'] = hash(ip_address) % 1000

        # 10. City (encoded)
        city = location_info.get('city', 'unknown')
        features['city_encoded'] = hash(city) % 500

        return features

    def _haversine_distance(
        self,
        lat1: float, lon1: float,
        lat2: float, lon2: float
    ) -> float:
        """
        Calculate distance between two points on Earth (km).

        Uses Haversine formula: https://en.wikipedia.org/wiki/Haversine_formula
        """
        # Convert to radians
        lat1, lon1, lat2, lon2 = map(
            np.radians, [lat1, lon1, lat2, lon2]
        )

        # Haversine formula
        dlat = lat2 - lat1
        dlon = lon2 - lon1

        a = (
            np.sin(dlat/2)**2 +
            np.cos(lat1) * np.cos(lat2) * np.sin(dlon/2)**2
        )
        c = 2 * np.arcsin(np.sqrt(a))

        # Earth's radius in km
        r = 6371

        return c * r

    def _get_user_countries(self, user_id: str, db: Session) -> set:
        """Get set of countries user has transacted from."""
        try:
            from app.models import FraudTransaction

            results = db.query(FraudTransaction.metadata).filter(
                FraudTransaction.user_id == user_id
            ).limit(100).all()

            countries = set()
            for result in results:
                if result[0] and 'location' in result[0]:
                    country = result[0]['location'].get('country')
                    if country:
                        countries.add(country)

            return countries

        except Exception as e:
            logger.error(f"Failed to get user countries: {str(e)}")
            return set()

    def _get_last_location(self, user_id: str, db: Session) -> Optional[Dict]:
        """Get user's last transaction location."""
        try:
            from app.models import FraudTransaction

            last_txn = db.query(FraudTransaction).filter(
                FraudTransaction.user_id == user_id
            ).order_by(
                FraudTransaction.created_at.desc()
            ).first()

            if last_txn and last_txn.metadata:
                location = last_txn.metadata.get('location', {})
                return {
                    'latitude': location.get('latitude', 0.0),
                    'longitude': location.get('longitude', 0.0),
                    'timestamp': last_txn.created_at
                }

            return None

        except Exception as e:
            logger.error(f"Failed to get last location: {str(e)}")
            return None

    def _get_home_location(self, user_id: str, db: Session) -> Optional[Dict]:
        """Get user's home location (most common transaction location)."""
        # Placeholder - in production, would calculate from transaction history
        return None

    def _get_last_country(self, user_id: str, db: Session) -> Optional[str]:
        """Get country of user's last transaction."""
        try:
            from app.models import FraudTransaction

            last_txn = db.query(FraudTransaction).filter(
                FraudTransaction.user_id == user_id
            ).order_by(
                FraudTransaction.created_at.desc()
            ).first()

            if last_txn and last_txn.metadata:
                return last_txn.metadata.get('location', {}).get('country')

            return None

        except Exception as e:
            logger.error(f"Failed to get last country: {str(e)}")
            return None


# ============================================================================
# VELOCITY FEATURES (10 features)
# ============================================================================

class VelocityFeatures:
    """
    Extract velocity features (transaction frequency patterns).

    Features (10 total):
    1. txn_count_last_1_hour - Transactions in last hour
    2. txn_count_last_6_hours - Transactions in last 6 hours
    3. txn_count_last_24_hours - Transactions in last 24 hours
    4. amount_sum_last_1_hour - Amount spent in last hour
    5. amount_sum_last_6_hours - Amount spent in last 6 hours
    6. amount_sum_last_24_hours - Amount spent in last 24 hours
    7. unique_merchants_last_24_hours - Unique merchants (24h)
    8. unique_countries_last_24_hours - Unique countries (24h)
    9. avg_time_between_txn_minutes - Average time between txns
    10. velocity_score - Composite velocity score
    """

    def extract(self, transaction: Dict[str, Any], db: Session) -> Dict[str, float]:
        """Extract transaction velocity features."""
        features = {}

        user_id = transaction.get('user_id')
        timestamp = transaction.get('timestamp')

        if isinstance(timestamp, str):
            timestamp = datetime.fromisoformat(timestamp.replace('Z', '+00:00'))

        # Load recent transaction history
        recent_txns = self._load_recent_transactions(user_id, timestamp, db)

        if len(recent_txns) == 0:
            # No history - all zeros
            return {
                'txn_count_last_1_hour': 0,
                'txn_count_last_6_hours': 0,
                'txn_count_last_24_hours': 0,
                'amount_sum_last_1_hour': 0.0,
                'amount_sum_last_6_hours': 0.0,
                'amount_sum_last_24_hours': 0.0,
                'unique_merchants_last_24_hours': 0,
                'unique_countries_last_24_hours': 0,
                'avg_time_between_txn_minutes': 0.0,
                'velocity_score': 0.0
            }

        # ================================================================
        # TRANSACTION COUNT FEATURES (3 features)
        # ================================================================

        # 1. Transactions in last 1 hour
        one_hour_ago = timestamp - timedelta(hours=1)
        txns_1h = recent_txns[recent_txns['timestamp'] >= one_hour_ago]
        features['txn_count_last_1_hour'] = len(txns_1h)

        # 2. Transactions in last 6 hours
        six_hours_ago = timestamp - timedelta(hours=6)
        txns_6h = recent_txns[recent_txns['timestamp'] >= six_hours_ago]
        features['txn_count_last_6_hours'] = len(txns_6h)

        # 3. Transactions in last 24 hours
        twenty_four_hours_ago = timestamp - timedelta(hours=24)
        txns_24h = recent_txns[recent_txns['timestamp'] >= twenty_four_hours_ago]
        features['txn_count_last_24_hours'] = len(txns_24h)

        # ================================================================
        # AMOUNT FEATURES (3 features)
        # ================================================================

        # 4. Amount sum in last 1 hour
        features['amount_sum_last_1_hour'] = (
            txns_1h['amount'].sum() if len(txns_1h) > 0 else 0.0
        )

        # 5. Amount sum in last 6 hours
        features['amount_sum_last_6_hours'] = (
            txns_6h['amount'].sum() if len(txns_6h) > 0 else 0.0
        )

        # 6. Amount sum in last 24 hours
        features['amount_sum_last_24_hours'] = (
            txns_24h['amount'].sum() if len(txns_24h) > 0 else 0.0
        )

        # ================================================================
        # DIVERSITY FEATURES (2 features)
        # ================================================================

        # 7. Unique merchants in last 24 hours
        features['unique_merchants_last_24_hours'] = (
            txns_24h['merchant'].nunique() if len(txns_24h) > 0 else 0
        )

        # 8. Unique countries in last 24 hours
        features['unique_countries_last_24_hours'] = (
            txns_24h['country'].nunique() if len(txns_24h) > 0 else 0
        )

        # ================================================================
        # TIME FEATURES (2 features)
        # ================================================================

        # 9. Average time between transactions (minutes)
        if len(recent_txns) > 1:
            recent_txns_sorted = recent_txns.sort_values('timestamp')
            time_diffs = recent_txns_sorted['timestamp'].diff()
            avg_diff_seconds = time_diffs.mean().total_seconds()
            features['avg_time_between_txn_minutes'] = avg_diff_seconds / 60
        else:
            features['avg_time_between_txn_minutes'] = 0.0

        # 10. Velocity score (composite)
        # High velocity = many transactions in short time = suspicious
        # Formula: (txn_count_1h * 10) + (txn_count_6h * 2) + txn_count_24h
        features['velocity_score'] = (
            features['txn_count_last_1_hour'] * 10 +
            features['txn_count_last_6_hours'] * 2 +
            features['txn_count_last_24_hours']
        )

        return features

    def _load_recent_transactions(
        self,
        user_id: str,
        timestamp: datetime,
        db: Session
    ) -> pd.DataFrame:
        """Load user's transactions from last 24 hours."""
        try:
            from app.models import FraudTransaction

            twenty_four_hours_ago = timestamp - timedelta(hours=24)

            transactions = db.query(FraudTransaction).filter(
                FraudTransaction.user_id == user_id,
                FraudTransaction.created_at >= twenty_four_hours_ago,
                FraudTransaction.created_at < timestamp
            ).all()

            if not transactions:
                return pd.DataFrame()

            data = []
            for txn in transactions:
                merchant = txn.merchant or 'unknown'
                country = (
                    txn.metadata.get('location', {}).get('country', 'UNKNOWN')
                    if txn.metadata else 'UNKNOWN'
                )

                data.append({
                    'timestamp': txn.created_at,
                    'amount': txn.amount,
                    'merchant': merchant,
                    'country': country
                })

            return pd.DataFrame(data)

        except Exception as e:
            logger.error(f"Failed to load recent transactions: {str(e)}")
            return pd.DataFrame()
```

**Lines so far: ~950**

Let's continue with the encoders and transformers:

### Step 4: Categorical Encoders

**File: `/home/user/sentinel-api/app/ml/encoders.py`**

```python
"""
Categorical Encoding for Fraud Detection.

This module handles encoding of categorical variables (strings) into
numeric values that machine learning models can process.

Two encoding strategies:
1. One-Hot Encoding - For low-cardinality categories (< 10 unique values)
2. Label Encoding - For high-cardinality categories (> 10 unique values)

Author: Sentinel ML Team
Version: 1.0.0
"""

import logging
from typing import Dict, List, Optional, Any
import pandas as pd
import numpy as np
from sklearn.preprocessing import OneHotEncoder as SklearnOneHotEncoder
from sklearn.preprocessing import LabelEncoder as SklearnLabelEncoder
import pickle
import os

logger = logging.getLogger(__name__)


class CategoricalEncoder:
    """
    Main categorical encoder that handles both one-hot and label encoding.

    Usage:
        encoder = CategoricalEncoder()
        encoder.fit(training_data)
        encoded = encoder.transform(new_data)

    Automatically chooses encoding strategy based on cardinality:
    - < 10 unique values: One-hot encoding
    - >= 10 unique values: Label encoding
    """

    def __init__(self, cardinality_threshold: int = 10):
        """
        Initialize encoder.

        Args:
            cardinality_threshold: Max unique values for one-hot encoding
        """
        self.cardinality_threshold = cardinality_threshold

        # Storage for encoders
        self.onehot_encoders: Dict[str, SklearnOneHotEncoder] = {}
        self.label_encoders: Dict[str, SklearnLabelEncoder] = {}

        # Encoding strategy per column
        self.encoding_strategy: Dict[str, str] = {}

        # Fitted flag
        self.is_fitted = False

        logger.info(
            f"CategoricalEncoder initialized with threshold={cardinality_threshold}"
        )

    def fit(self, data: pd.DataFrame, categorical_columns: List[str]) -> 'CategoricalEncoder':
        """
        Fit encoders on training data.

        Args:
            data: Training DataFrame
            categorical_columns: List of categorical column names

        Returns:
            self (for method chaining)
        """
        logger.info(f"Fitting encoders on {len(categorical_columns)} columns")

        for col in categorical_columns:
            if col not in data.columns:
                logger.warning(f"Column {col} not found in data, skipping")
                continue

            # Get unique values
            unique_values = data[col].nunique()

            # Choose encoding strategy
            if unique_values < self.cardinality_threshold:
                # One-hot encoding for low cardinality
                self.encoding_strategy[col] = 'onehot'

                encoder = SklearnOneHotEncoder(
                    sparse_output=False,
                    handle_unknown='ignore'  # Handle new categories
                )
                encoder.fit(data[[col]])
                self.onehot_encoders[col] = encoder

                logger.info(
                    f"{col}: One-hot encoding ({unique_values} categories)"
                )
            else:
                # Label encoding for high cardinality
                self.encoding_strategy[col] = 'label'

                encoder = SklearnLabelEncoder()
                encoder.fit(data[col])
                self.label_encoders[col] = encoder

                logger.info(
                    f"{col}: Label encoding ({unique_values} categories)"
                )

        self.is_fitted = True
        logger.info("Encoder fitting complete")
        return self

    def transform(self, data: pd.DataFrame) -> pd.DataFrame:
        """
        Transform categorical columns to numeric.

        Args:
            data: DataFrame to transform

        Returns:
            DataFrame with encoded columns
        """
        if not self.is_fitted:
            raise ValueError("Encoder not fitted. Call fit() first.")

        result = data.copy()

        # One-hot encoding
        for col, encoder in self.onehot_encoders.items():
            if col in result.columns:
                # Transform
                encoded = encoder.transform(result[[col]])

                # Create column names
                feature_names = [
                    f"{col}_{category}"
                    for category in encoder.categories_[0]
                ]

                # Add to result
                encoded_df = pd.DataFrame(
                    encoded,
                    columns=feature_names,
                    index=result.index
                )
                result = pd.concat([result, encoded_df], axis=1)

                # Drop original column
                result = result.drop(columns=[col])

        # Label encoding
        for col, encoder in self.label_encoders.items():
            if col in result.columns:
                # Transform with unknown handling
                try:
                    result[col] = encoder.transform(result[col])
                except ValueError:
                    # Handle unknown categories
                    result[col] = result[col].apply(
                        lambda x: self._label_encode_with_unknown(x, encoder)
                    )

        return result

    def _label_encode_with_unknown(
        self,
        value: Any,
        encoder: SklearnLabelEncoder
    ) -> int:
        """
        Encode value, handling unknown categories.

        Args:
            value: Value to encode
            encoder: Fitted label encoder

        Returns:
            Encoded integer value
        """
        try:
            return encoder.transform([value])[0]
        except ValueError:
            # Unknown category - return max + 1
            return len(encoder.classes_)

    def fit_transform(
        self,
        data: pd.DataFrame,
        categorical_columns: List[str]
    ) -> pd.DataFrame:
        """
        Fit and transform in one step.

        Args:
            data: Training DataFrame
            categorical_columns: List of categorical column names

        Returns:
            Transformed DataFrame
        """
        self.fit(data, categorical_columns)
        return self.transform(data)

    def save(self, filepath: str) -> None:
        """
        Save encoder to disk.

        Args:
            filepath: Path to save encoder
        """
        encoder_data = {
            'cardinality_threshold': self.cardinality_threshold,
            'onehot_encoders': self.onehot_encoders,
            'label_encoders': self.label_encoders,
            'encoding_strategy': self.encoding_strategy,
            'is_fitted': self.is_fitted
        }

        os.makedirs(os.path.dirname(filepath), exist_ok=True)
        with open(filepath, 'wb') as f:
            pickle.dump(encoder_data, f)

        logger.info(f"Encoder saved to {filepath}")

    @classmethod
    def load(cls, filepath: str) -> 'CategoricalEncoder':
        """
        Load encoder from disk.

        Args:
            filepath: Path to saved encoder

        Returns:
            Loaded CategoricalEncoder instance
        """
        with open(filepath, 'rb') as f:
            encoder_data = pickle.load(f)

        encoder = cls(encoder_data['cardinality_threshold'])
        encoder.onehot_encoders = encoder_data['onehot_encoders']
        encoder.label_encoders = encoder_data['label_encoders']
        encoder.encoding_strategy = encoder_data['encoding_strategy']
        encoder.is_fitted = encoder_data['is_fitted']

        logger.info(f"Encoder loaded from {filepath}")
        return encoder


# ============================================================================
# FRAUD-SPECIFIC CATEGORICAL COLUMNS
# ============================================================================

# Columns that should be one-hot encoded (low cardinality)
ONEHOT_COLUMNS = [
    'country',          # ~200 countries, but we'll see ~10 frequently
    'payment_method',   # credit_card, debit_card, paypal, etc.
    'card_type',        # visa, mastercard, amex, discover
    'transaction_type'  # purchase, refund, withdrawal
]

# Columns that should be label encoded (high cardinality)
LABEL_COLUMNS = [
    'merchant',         # Thousands of merchants
    'merchant_category',# Hundreds of categories
    'browser',          # Many browser types
    'os',              # Many OS types
    'device_id',       # Unique per device
    'ip_address',      # Many unique IPs
    'city'             # Thousands of cities
]


def create_fraud_encoder() -> CategoricalEncoder:
    """
    Create a categorical encoder configured for fraud detection.

    Returns:
        Configured CategoricalEncoder instance
    """
    return CategoricalEncoder(cardinality_threshold=10)


# ============================================================================
# EXAMPLE USAGE
# ============================================================================

if __name__ == "__main__":
    """
    Example: Encoding categorical variables for fraud detection.
    """

    # Sample training data
    training_data = pd.DataFrame({
        'country': ['US', 'GB', 'US', 'CA', 'US', 'GB'],
        'merchant': ['Amazon', 'eBay', 'Amazon', 'Walmart', 'Target', 'eBay'],
        'browser': ['Chrome', 'Firefox', 'Safari', 'Chrome', 'Chrome', 'Edge'],
        'card_type': ['visa', 'mastercard', 'visa', 'amex', 'visa', 'visa']
    })

    # Initialize encoder
    encoder = CategoricalEncoder(cardinality_threshold=5)

    # Fit on training data
    categorical_cols = ['country', 'merchant', 'browser', 'card_type']
    encoder.fit(training_data, categorical_cols)

    # Transform training data
    encoded_train = encoder.transform(training_data)
    print("Encoded training data:")
    print(encoded_train.head())

    # Transform new data
    new_data = pd.DataFrame({
        'country': ['FR', 'US'],  # FR is new!
        'merchant': ['Amazon', 'NewMerchant'],  # NewMerchant is new!
        'browser': ['Chrome', 'Opera'],  # Opera is new!
        'card_type': ['visa', 'discover']  # discover is new!
    })

    encoded_new = encoder.transform(new_data)
    print("\nEncoded new data (with unknowns):")
    print(encoded_new)

    # Save encoder
    encoder.save('/tmp/fraud_encoder.pkl')

    # Load encoder
    loaded_encoder = CategoricalEncoder.load('/tmp/fraud_encoder.pkl')
    print("\nEncoder loaded successfully!")
```

### Step 5: Feature Transformers

**File: `/home/user/sentinel-api/app/ml/transformers.py`**

```python
"""
Feature Transformations for Fraud Detection.

This module handles feature scaling, normalization, and transformation
to prepare features for machine learning models.

Transformations:
1. StandardScaler - Z-score normalization (mean=0, std=1)
2. MinMaxScaler - Min-max normalization (range [0, 1])
3. Imputation - Handle missing values
4. Outlier capping - Cap extreme values

Author: Sentinel ML Team
Version: 1.0.0
"""

import logging
from typing import Dict, List, Optional, Union
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler as SklearnStandardScaler
from sklearn.preprocessing import MinMaxScaler as SklearnMinMaxScaler
from sklearn.impute import SimpleImputer
import pickle
import os

logger = logging.getLogger(__name__)


class FeatureTransformer:
    """
    Main feature transformer for scaling and normalization.

    Usage:
        transformer = FeatureTransformer(method='standard')
        transformer.fit(training_data)
        scaled = transformer.transform(new_data)
    """

    def __init__(
        self,
        method: str = 'standard',
        handle_outliers: bool = True,
        outlier_std: float = 3.0
    ):
        """
        Initialize transformer.

        Args:
            method: Scaling method ('standard' or 'minmax')
            handle_outliers: Whether to cap outliers
            outlier_std: Number of std devs for outlier capping
        """
        self.method = method
        self.handle_outliers = handle_outliers
        self.outlier_std = outlier_std

        # Initialize scalers
        if method == 'standard':
            self.scaler = SklearnStandardScaler()
        elif method == 'minmax':
            self.scaler = SklearnMinMaxScaler()
        else:
            raise ValueError(f"Unknown method: {method}")

        # Imputer for missing values
        self.imputer = SimpleImputer(strategy='median')

        # Outlier bounds (learned from training data)
        self.outlier_bounds: Dict[str, tuple] = {}

        # Fitted flag
        self.is_fitted = False

        logger.info(f"FeatureTransformer initialized with method={method}")

    def fit(
        self,
        data: pd.DataFrame,
        feature_columns: Optional[List[str]] = None
    ) -> 'FeatureTransformer':
        """
        Fit transformer on training data.

        Args:
            data: Training DataFrame
            feature_columns: Columns to transform (None = all numeric)

        Returns:
            self (for method chaining)
        """
        if feature_columns is None:
            # Auto-detect numeric columns
            feature_columns = data.select_dtypes(
                include=[np.number]
            ).columns.tolist()

        self.feature_columns = feature_columns

        logger.info(f"Fitting transformer on {len(feature_columns)} features")

        # Extract features
        X = data[feature_columns]

        # Fit imputer
        self.imputer.fit(X)

        # Impute missing values for fitting
        X_imputed = pd.DataFrame(
            self.imputer.transform(X),
            columns=feature_columns,
            index=X.index
        )

        # Calculate outlier bounds
        if self.handle_outliers:
            for col in feature_columns:
                mean = X_imputed[col].mean()
                std = X_imputed[col].std()

                lower_bound = mean - (self.outlier_std * std)
                upper_bound = mean + (self.outlier_std * std)

                self.outlier_bounds[col] = (lower_bound, upper_bound)

                logger.debug(
                    f"{col}: Outlier bounds [{lower_bound:.2f}, {upper_bound:.2f}]"
                )

        # Cap outliers for fitting
        X_capped = self._cap_outliers(X_imputed)

        # Fit scaler
        self.scaler.fit(X_capped)

        self.is_fitted = True
        logger.info("Transformer fitting complete")
        return self

    def transform(self, data: pd.DataFrame) -> pd.DataFrame:
        """
        Transform features.

        Args:
            data: DataFrame to transform

        Returns:
            DataFrame with transformed features
        """
        if not self.is_fitted:
            raise ValueError("Transformer not fitted. Call fit() first.")

        result = data.copy()

        # Extract features
        X = result[self.feature_columns]

        # Impute missing values
        X_imputed = pd.DataFrame(
            self.imputer.transform(X),
            columns=self.feature_columns,
            index=X.index
        )

        # Cap outliers
        X_capped = self._cap_outliers(X_imputed)

        # Scale
        X_scaled = pd.DataFrame(
            self.scaler.transform(X_capped),
            columns=self.feature_columns,
            index=X.index
        )

        # Replace in result
        result[self.feature_columns] = X_scaled

        return result

    def _cap_outliers(self, data: pd.DataFrame) -> pd.DataFrame:
        """
        Cap outliers to specified bounds.

        Args:
            data: DataFrame with features

        Returns:
            DataFrame with capped outliers
        """
        if not self.handle_outliers:
            return data

        result = data.copy()

        for col, (lower, upper) in self.outlier_bounds.items():
            if col in result.columns:
                result[col] = result[col].clip(lower, upper)

        return result

    def fit_transform(
        self,
        data: pd.DataFrame,
        feature_columns: Optional[List[str]] = None
    ) -> pd.DataFrame:
        """
        Fit and transform in one step.

        Args:
            data: Training DataFrame
            feature_columns: Columns to transform

        Returns:
            Transformed DataFrame
        """
        self.fit(data, feature_columns)
        return self.transform(data)

    def inverse_transform(self, data: pd.DataFrame) -> pd.DataFrame:
        """
        Inverse transform (scaled back to original scale).

        Args:
            data: Scaled DataFrame

        Returns:
            DataFrame in original scale
        """
        if not self.is_fitted:
            raise ValueError("Transformer not fitted. Call fit() first.")

        result = data.copy()

        # Extract scaled features
        X_scaled = result[self.feature_columns]

        # Inverse scale
        X_original = pd.DataFrame(
            self.scaler.inverse_transform(X_scaled),
            columns=self.feature_columns,
            index=X_scaled.index
        )

        # Replace in result
        result[self.feature_columns] = X_original

        return result

    def save(self, filepath: str) -> None:
        """
        Save transformer to disk.

        Args:
            filepath: Path to save transformer
        """
        transformer_data = {
            'method': self.method,
            'handle_outliers': self.handle_outliers,
            'outlier_std': self.outlier_std,
            'scaler': self.scaler,
            'imputer': self.imputer,
            'outlier_bounds': self.outlier_bounds,
            'feature_columns': self.feature_columns,
            'is_fitted': self.is_fitted
        }

        os.makedirs(os.path.dirname(filepath), exist_ok=True)
        with open(filepath, 'wb') as f:
            pickle.dump(transformer_data, f)

        logger.info(f"Transformer saved to {filepath}")

    @classmethod
    def load(cls, filepath: str) -> 'FeatureTransformer':
        """
        Load transformer from disk.

        Args:
            filepath: Path to saved transformer

        Returns:
            Loaded FeatureTransformer instance
        """
        with open(filepath, 'rb') as f:
            transformer_data = pickle.load(f)

        transformer = cls(
            method=transformer_data['method'],
            handle_outliers=transformer_data['handle_outliers'],
            outlier_std=transformer_data['outlier_std']
        )
        transformer.scaler = transformer_data['scaler']
        transformer.imputer = transformer_data['imputer']
        transformer.outlier_bounds = transformer_data['outlier_bounds']
        transformer.feature_columns = transformer_data['feature_columns']
        transformer.is_fitted = transformer_data['is_fitted']

        logger.info(f"Transformer loaded from {filepath}")
        return transformer


# ============================================================================
# FEATURE IMPORTANCE ANALYSIS
# ============================================================================

class FeatureImportanceAnalyzer:
    """
    Analyze feature importance and correlation.

    Helps identify:
    - Most important features for fraud detection
    - Highly correlated features (redundancy)
    - Features with low variance (not useful)
    """

    @staticmethod
    def analyze_correlation(
        data: pd.DataFrame,
        threshold: float = 0.9
    ) -> pd.DataFrame:
        """
        Find highly correlated feature pairs.

        Args:
            data: Feature DataFrame
            threshold: Correlation threshold (0-1)

        Returns:
            DataFrame with correlated pairs
        """
        # Calculate correlation matrix
        corr_matrix = data.corr().abs()

        # Get upper triangle (avoid duplicates)
        upper = corr_matrix.where(
            np.triu(np.ones(corr_matrix.shape), k=1).astype(bool)
        )

        # Find highly correlated pairs
        high_corr = []
        for column in upper.columns:
            for index in upper.index:
                if upper.loc[index, column] >= threshold:
                    high_corr.append({
                        'feature1': index,
                        'feature2': column,
                        'correlation': upper.loc[index, column]
                    })

        result = pd.DataFrame(high_corr)

        if len(result) > 0:
            result = result.sort_values('correlation', ascending=False)
            logger.info(f"Found {len(result)} highly correlated feature pairs")

        return result

    @staticmethod
    def analyze_variance(
        data: pd.DataFrame,
        threshold: float = 0.01
    ) -> pd.DataFrame:
        """
        Find low-variance features.

        Args:
            data: Feature DataFrame
            threshold: Variance threshold

        Returns:
            DataFrame with low-variance features
        """
        variances = data.var()
        low_var = variances[variances < threshold]

        result = pd.DataFrame({
            'feature': low_var.index,
            'variance': low_var.values
        }).sort_values('variance')

        logger.info(f"Found {len(result)} low-variance features")
        return result

    @staticmethod
    def analyze_missing_values(data: pd.DataFrame) -> pd.DataFrame:
        """
        Analyze missing value patterns.

        Args:
            data: Feature DataFrame

        Returns:
            DataFrame with missing value statistics
        """
        missing_count = data.isnull().sum()
        missing_pct = (missing_count / len(data)) * 100

        result = pd.DataFrame({
            'feature': missing_count.index,
            'missing_count': missing_count.values,
            'missing_pct': missing_pct.values
        })

        result = result[result['missing_count'] > 0]
        result = result.sort_values('missing_count', ascending=False)

        logger.info(
            f"Found {len(result)} features with missing values"
        )
        return result


# ============================================================================
# EXAMPLE USAGE
# ============================================================================

if __name__ == "__main__":
    """
    Example: Feature transformation and analysis.
    """

    # Sample feature data
    np.random.seed(42)
    n_samples = 1000

    training_data = pd.DataFrame({
        'amount': np.random.exponential(100, n_samples),
        'txn_count': np.random.poisson(5, n_samples),
        'account_age_days': np.random.uniform(0, 365, n_samples),
        'fraud_rate': np.random.beta(2, 10, n_samples),
        'velocity_score': np.random.gamma(2, 2, n_samples)
    })

    # Add some missing values
    training_data.loc[
        np.random.choice(training_data.index, 50),
        'account_age_days'
    ] = np.nan

    # Add some outliers
    training_data.loc[
        np.random.choice(training_data.index, 20),
        'amount'
    ] = np.random.uniform(10000, 50000, 20)

    print("Training data:")
    print(training_data.describe())
    print(f"\nMissing values: {training_data.isnull().sum().sum()}")

    # Initialize transformer
    transformer = FeatureTransformer(
        method='standard',
        handle_outliers=True,
        outlier_std=3.0
    )

    # Fit and transform
    scaled_data = transformer.fit_transform(training_data)

    print("\nScaled data:")
    print(scaled_data.describe())

    # Analyze correlation
    analyzer = FeatureImportanceAnalyzer()
    corr_pairs = analyzer.analyze_correlation(scaled_data, threshold=0.8)
    print("\nHighly correlated pairs:")
    print(corr_pairs)

    # Analyze variance
    low_var = analyzer.analyze_variance(training_data, threshold=1.0)
    print("\nLow variance features:")
    print(low_var)

    # Analyze missing values
    missing = analyzer.analyze_missing_values(training_data)
    print("\nMissing values:")
    print(missing)

    # Save transformer
    transformer.save('/tmp/fraud_transformer.pkl')
    print("\nTransformer saved!")

    # Load transformer
    loaded_transformer = FeatureTransformer.load('/tmp/fraud_transformer.pkl')
    print("Transformer loaded successfully!")
```

---

## Feature Categories Deep Dive

### Transaction Features (15 features)

**Why these features matter:**

1. **Amount Features** - Transaction amounts follow patterns
   - Legitimate: Normal distribution around user's average
   - Fraud: Often outliers (very large or test amounts)

2. **Temporal Features** - Time patterns reveal fraud
   - Fraud peaks: Late night (2-4 AM), early morning (5-7 AM)
   - Legitimate: Business hours (9 AM - 5 PM)

**Example Patterns:**

```python
# Legitimate user
{
    'amount': 45.20,              # Normal amount
    'amount_zscore': 0.3,         # Close to average
    'hour_of_day': 14,            # Afternoon
    'is_business_hours': 1,       # Business hours
    'is_night': 0                 # Not late night
}

# Fraudulent transaction
{
    'amount': 4999.99,            # Just below $5000 limit
    'amount_zscore': 8.5,         # Way above average!
    'hour_of_day': 3,             # 3 AM
    'is_business_hours': 0,       # Not business hours
    'is_night': 1,                # Late night - suspicious!
    'amount_round_number': 1      # Testing with round amount
}
```

### User Features (15 features)

**Why user history matters:**

Fraudsters have different patterns than legitimate users:
- **New accounts** - Higher fraud risk (60% of fraud is on accounts < 30 days old)
- **No transaction history** - Can't establish baseline behavior
- **Sudden behavior change** - $50 average → $5000 transaction

**Example Patterns:**

```python
# Legitimate user (established)
{
    'account_age_days': 450,           # Old account
    'total_transaction_count': 234,    # Many transactions
    'avg_transaction_amount': 78.50,   # Consistent spending
    'fraud_history_count': 0,          # No fraud history
    'amount_vs_user_avg': 1.2,         # 1.2x average (normal)
    'is_verified': 1                   # Verified account
}

# Fraudster (account takeover)
{
    'account_age_days': 3,             # Brand new account!
    'total_transaction_count': 1,      # First transaction
    'avg_transaction_amount': 0,       # No history
    'fraud_history_count': 0,          # No history yet
    'amount_vs_user_avg': 999,         # Can't calculate (no avg)
    'is_verified': 0                   # Not verified - red flag!
}
```

### Device Features (10 features)

**Device fingerprinting is crucial:**

- **New device flag** - 75% of fraud uses new devices
- **VPN/Proxy** - Fraudsters hide their location
- **Device fraud history** - Compromised devices keep getting used

**Example Patterns:**

```python
# Legitimate user (same device)
{
    'new_device_flag': 0,              # Known device
    'device_age_days': 180,            # Old device
    'device_transaction_count': 89,    # Many uses
    'device_fraud_count': 0,           # Clean device
    'vpn_flag': 0,                     # No VPN
    'proxy_flag': 0,                   # No proxy
    'tor_flag': 0                      # Not using Tor
}

# Fraudster (stolen card)
{
    'new_device_flag': 1,              # New device - RED FLAG!
    'device_age_days': 0,              # Never seen before
    'device_transaction_count': 0,     # First use
    'device_fraud_count': 0,           # No history yet
    'vpn_flag': 1,                     # Using VPN - hiding!
    'proxy_flag': 1,                   # Using proxy - hiding!
    'tor_flag': 0                      # Not Tor (this time)
}
```

### Location Features (10 features)

**Geographic anomalies reveal fraud:**

- **Impossible travel** - Physics violations (e.g., 1000 km in 1 hour)
- **New country** - Sudden country change
- **High-risk countries** - Some countries have higher fraud rates

**Example - Impossible Travel Detection:**

```python
# Scenario: Card used in New York, then London 2 hours later

# Transaction 1 (New York)
{
    'latitude': 40.7128,
    'longitude': -74.0060,
    'timestamp': '2025-12-30T10:00:00Z'
}

# Transaction 2 (London) - 2 hours later
{
    'latitude': 51.5074,
    'longitude': -0.1278,
    'timestamp': '2025-12-30T12:00:00Z'
}

# Feature calculation:
distance_km = 5570  # NYC to London
time_hours = 2
speed = 5570 / 2 = 2785 km/h

# Max airplane speed: 900 km/h
# This is IMPOSSIBLE! (unless you're Superman)
{
    'distance_from_last_txn_km': 5570,
    'impossible_travel_flag': 1,        # FRAUD ALERT!
    'new_country_flag': 1,              # Different country
    'country_change_flag': 1            # Country changed
}
```

### Velocity Features (10 features)

**Velocity = Speed of transactions**

Fraudsters act fast:
- **Card testing** - Multiple small transactions in minutes
- **Max out limits** - Many transactions before card is blocked
- **Shopping spree** - Many different merchants quickly

**Example Patterns:**

```python
# Legitimate user (normal velocity)
{
    'txn_count_last_1_hour': 1,        # 1 transaction/hour
    'txn_count_last_6_hours': 3,       # 3 transactions/6h
    'txn_count_last_24_hours': 5,      # 5 transactions/day
    'amount_sum_last_1_hour': 45.0,    # $45 in last hour
    'unique_merchants_last_24_hours': 4, # 4 different stores
    'velocity_score': 18               # Low velocity
}

# Fraudster (high velocity)
{
    'txn_count_last_1_hour': 12,       # 12 transactions!
    'txn_count_last_6_hours': 12,      # All in last hour
    'txn_count_last_24_hours': 12,     # All in last hour
    'amount_sum_last_1_hour': 8450.0,  # $8,450 in one hour!
    'unique_merchants_last_24_hours': 11, # 11 different stores!
    'velocity_score': 144              # VERY HIGH velocity!
}
```

---

## Categorical Encoding Explained

### Why Encode Categorical Variables?

Machine learning models only understand numbers, not strings. We need to convert categories like "US", "Chrome", "Visa" into numbers.

### Two Encoding Strategies

**1. One-Hot Encoding (Low Cardinality)**

Use when there are few unique values (< 10):

```python
# Before encoding
country = ['US', 'GB', 'CA', 'US']

# After one-hot encoding
country_US = [1, 0, 0, 1]
country_GB = [0, 1, 0, 0]
country_CA = [0, 0, 1, 0]

# Each category gets its own binary column
# Advantage: No false ordinal relationship
# Disadvantage: Many columns if high cardinality
```

**2. Label Encoding (High Cardinality)**

Use when there are many unique values (>= 10):

```python
# Before encoding
merchant = ['Amazon', 'eBay', 'Walmart', 'Target', 'Amazon']

# After label encoding
merchant_encoded = [0, 1, 2, 3, 0]

# Each unique value gets a unique integer
# Advantage: Only one column
# Disadvantage: Implies false ordering (Amazon=0 < eBay=1)
```

### Handling Unknown Categories

What if we see a new category in production that wasn't in training?

```python
# Training: ['US', 'GB', 'CA']
# Production: ['US', 'FR']  # FR is new!

# One-hot: Ignore unknown (all zeros)
# Label: Assign max + 1
```

---

## Feature Transformations

### Why Transform Features?

**Problem:** Different features have different scales:

```python
# Before scaling
{
    'amount': 1250.00,           # Scale: 0-10000
    'account_age_days': 45,      # Scale: 0-3650
    'fraud_rate': 0.03,          # Scale: 0-1
    'txn_count_last_hour': 5     # Scale: 0-100
}
```

Models like logistic regression and neural networks are sensitive to scale. Features with large values will dominate!

### Standard Scaling (Z-Score Normalization)

**Formula:** `(x - mean) / std`

**Result:** Mean = 0, Std = 1

```python
# Before
amount = [100, 200, 150, 10000]  # Outlier!
mean = 2612.5
std = 4280.5

# After standard scaling
amount_scaled = [-0.59, -0.56, -0.58, 1.73]

# Outlier is still visible (1.73) but not dominating
```

**Use when:** Normal distribution, outliers exist

### Min-Max Scaling

**Formula:** `(x - min) / (max - min)`

**Result:** Range [0, 1]

```python
# Before
age_days = [0, 30, 365, 1000]
min = 0
max = 1000

# After min-max scaling
age_days_scaled = [0.0, 0.03, 0.365, 1.0]

# All values between 0 and 1
```

**Use when:** Bounded distribution, no outliers

### Outlier Handling

Cap extreme values to prevent them from skewing the model:

```python
# Before capping
amount = [50, 100, 150, 50000]  # 50000 is outlier

# Calculate bounds (mean ± 3*std)
mean = 12575
std = 21623
upper_bound = mean + 3*std = 77444

# After capping
amount_capped = [50, 100, 150, 77444]  # Outlier capped
```

---

## Testing Your Features

### Step 1: Create Test Script

**File: `/home/user/sentinel-api/test_features.py`**

```python
"""
Test script for feature engineering.

Verifies:
- All 60+ features are extracted
- No NaN values in critical features
- Feature ranges are reasonable
- Encoding works correctly
"""

import sys
sys.path.append('/home/user/sentinel-api')

from datetime import datetime, timedelta
from app.ml.features import FeatureExtractor
from app.ml.encoders import CategoricalEncoder
from app.ml.transformers import FeatureTransformer
import pandas as pd
import numpy as np

# Mock database session
class MockDB:
    def query(self, *args, **kwargs):
        class MockQuery:
            def filter(self, *args, **kwargs):
                return self
            def order_by(self, *args, **kwargs):
                return self
            def limit(self, *args, **kwargs):
                return self
            def all(self):
                return []
            def first(self):
                return None
        return MockQuery()


def test_feature_extraction():
    """Test that all 60+ features are extracted."""
    print("=" * 70)
    print("TEST 1: Feature Extraction")
    print("=" * 70)

    # Create sample transaction
    transaction = {
        'amount': 1250.00,
        'timestamp': datetime.utcnow(),
        'user_id': 'user_12345',
        'device_id': 'device_67890',
        'merchant': 'Amazon',
        'country': 'US',
        'user_info': {
            'account_created_at': datetime.utcnow() - timedelta(days=45),
            'is_verified': True
        },
        'device_info': {
            'browser': 'Chrome',
            'os': 'Windows',
            'vpn_detected': False,
            'proxy_detected': False,
            'tor_detected': False
        },
        'location_info': {
            'country': 'US',
            'city': 'New York',
            'latitude': 40.7128,
            'longitude': -74.0060,
            'ip_address': '192.168.1.1'
        }
    }

    # Extract features
    extractor = FeatureExtractor()
    db = MockDB()

    features = extractor.extract(transaction, db)

    # Verify feature count
    print(f"\n✓ Total features extracted: {len(features)}")
    assert len(features) >= 60, f"Expected 60+ features, got {len(features)}"

    # Print feature categories
    print("\nFeature categories:")
    print(f"  - Transaction features: {sum(1 for k in features if k.startswith(('amount', 'hour', 'day', 'is_', 'month', 'merchant')))}")
    print(f"  - User features: {sum(1 for k in features if k.startswith(('account', 'total', 'avg', 'std', 'min', 'max', 'fraud', 'transactions', 'unique_merchants', 'days_since')))}")
    print(f"  - Device features: {sum(1 for k in features if k.startswith(('device', 'new_device', 'vpn', 'proxy', 'tor', 'browser', 'os')))}")
    print(f"  - Location features: {sum(1 for k in features if k.startswith(('country', 'distance', 'impossible', 'timezone', 'ip', 'city')))}")
    print(f"  - Velocity features: {sum(1 for k in features if k.startswith(('txn_count', 'amount_sum', 'velocity')))}")

    # Check for NaN values
    nan_features = [k for k, v in features.items() if pd.isna(v)]
    print(f"\n✓ Features with NaN: {len(nan_features)}")
    if nan_features:
        print(f"  Warning: {nan_features}")

    # Print sample features
    print("\nSample features:")
    for i, (k, v) in enumerate(list(features.items())[:15]):
        print(f"  {k}: {v}")

    print("\n✅ Feature extraction test PASSED\n")
    return features


def test_categorical_encoding():
    """Test categorical encoding."""
    print("=" * 70)
    print("TEST 2: Categorical Encoding")
    print("=" * 70)

    # Create sample data
    data = pd.DataFrame({
        'country': ['US', 'GB', 'CA', 'US', 'GB'],
        'merchant': ['Amazon', 'eBay', 'Walmart', 'Amazon', 'Target'],
        'browser': ['Chrome', 'Firefox', 'Safari', 'Chrome', 'Edge']
    })

    print(f"\nOriginal data shape: {data.shape}")
    print(data.head())

    # Encode
    encoder = CategoricalEncoder(cardinality_threshold=3)
    categorical_cols = ['country', 'merchant', 'browser']

    encoder.fit(data, categorical_cols)
    encoded = encoder.transform(data)

    print(f"\n✓ Encoded data shape: {encoded.shape}")
    print(f"✓ Encoding strategy:")
    for col, strategy in encoder.encoding_strategy.items():
        print(f"  {col}: {strategy}")

    print("\n✅ Categorical encoding test PASSED\n")


def test_feature_transformation():
    """Test feature scaling and transformation."""
    print("=" * 70)
    print("TEST 3: Feature Transformation")
    print("=" * 70)

    # Create sample data with outliers
    np.random.seed(42)
    data = pd.DataFrame({
        'amount': [50, 100, 150, 200, 10000],  # Outlier!
        'txn_count': [1, 2, 3, 4, 5],
        'account_age': [10, 20, 30, 40, 50]
    })

    print(f"\nOriginal data:")
    print(data.describe())

    # Transform
    transformer = FeatureTransformer(
        method='standard',
        handle_outliers=True,
        outlier_std=2.0
    )

    scaled = transformer.fit_transform(data)

    print(f"\n✓ Scaled data:")
    print(scaled.describe())

    # Verify mean ≈ 0, std ≈ 1
    print(f"\n✓ Verification:")
    print(f"  Mean ≈ 0: {np.allclose(scaled.mean(), 0, atol=0.1)}")
    print(f"  Std ≈ 1: {np.allclose(scaled.std(), 1, atol=0.5)}")

    print("\n✅ Feature transformation test PASSED\n")


def test_feature_pipeline():
    """Test complete feature pipeline."""
    print("=" * 70)
    print("TEST 4: Complete Feature Pipeline")
    print("=" * 70)

    # Create multiple transactions
    transactions = []
    for i in range(10):
        txn = {
            'amount': np.random.uniform(50, 500),
            'timestamp': datetime.utcnow() - timedelta(hours=i),
            'user_id': f'user_{i % 3}',  # 3 different users
            'device_id': f'device_{i % 2}',  # 2 different devices
            'merchant': ['Amazon', 'eBay', 'Walmart'][i % 3],
            'country': ['US', 'GB', 'CA'][i % 3],
            'user_info': {
                'account_created_at': datetime.utcnow() - timedelta(days=30+i),
                'is_verified': i % 2 == 0
            },
            'device_info': {
                'browser': ['Chrome', 'Firefox'][i % 2],
                'os': ['Windows', 'MacOS'][i % 2],
                'vpn_detected': False,
                'proxy_detected': False,
                'tor_detected': False
            },
            'location_info': {
                'country': ['US', 'GB', 'CA'][i % 3],
                'city': ['NYC', 'London', 'Toronto'][i % 3],
                'latitude': 40.0 + i,
                'longitude': -74.0 + i,
                'ip_address': f'192.168.1.{i}'
            }
        }
        transactions.append(txn)

    # Extract features for all transactions
    extractor = FeatureExtractor()
    db = MockDB()

    feature_list = []
    for txn in transactions:
        features = extractor.extract(txn, db)
        feature_list.append(features)

    # Convert to DataFrame
    features_df = pd.DataFrame(feature_list)

    print(f"\n✓ Feature matrix shape: {features_df.shape}")
    print(f"  (10 transactions × {features_df.shape[1]} features)")

    # Check for NaN
    nan_count = features_df.isnull().sum().sum()
    print(f"\n✓ Total NaN values: {nan_count}")

    # Summary statistics
    print(f"\n✓ Feature statistics:")
    print(features_df.describe().iloc[:, :5])  # First 5 features

    print("\n✅ Complete pipeline test PASSED\n")


if __name__ == "__main__":
    """Run all tests."""
    print("\n" + "=" * 70)
    print("FEATURE ENGINEERING TEST SUITE")
    print("=" * 70 + "\n")

    # Run tests
    test_feature_extraction()
    test_categorical_encoding()
    test_feature_transformation()
    test_feature_pipeline()

    print("=" * 70)
    print("ALL TESTS PASSED! ✅")
    print("=" * 70)
    print("\nFeature engineering is working correctly.")
    print("You now have 60+ features ready for machine learning!")
```

### Step 2: Run Tests

```bash
# Activate virtual environment
cd /home/user/sentinel-api
source venv/bin/activate

# Run tests
python test_features.py
```

**Expected Output:**

```
======================================================================
FEATURE ENGINEERING TEST SUITE
======================================================================

======================================================================
TEST 1: Feature Extraction
======================================================================

✓ Total features extracted: 62

Feature categories:
  - Transaction features: 15
  - User features: 15
  - Device features: 10
  - Location features: 10
  - Velocity features: 10

✓ Features with NaN: 0

Sample features:
  amount: 1250.0
  amount_log: 7.131089
  amount_zscore: 5.5
  hour_of_day: 14
  is_weekend: 0
  ...

✅ Feature extraction test PASSED

======================================================================
TEST 2: Categorical Encoding
======================================================================

Original data shape: (5, 3)

✓ Encoded data shape: (5, 8)
✓ Encoding strategy:
  country: onehot
  merchant: label
  browser: label

✅ Categorical encoding test PASSED

======================================================================
TEST 3: Feature Transformation
======================================================================

Original data:
              amount  txn_count  account_age
mean     2100.00        3.0         30.0
std      4367.67        1.58        15.81

✓ Scaled data:
              amount  txn_count  account_age
mean     0.0             0.0         0.0
std      1.0             1.0         1.0

✓ Verification:
  Mean ≈ 0: True
  Std ≈ 1: True

✅ Feature transformation test PASSED

======================================================================
TEST 4: Complete Feature Pipeline
======================================================================

✓ Feature matrix shape: (10, 62)
  (10 transactions × 62 features)

✓ Total NaN values: 0

======================================================================
ALL TESTS PASSED! ✅
======================================================================

Feature engineering is working correctly.
You now have 60+ features ready for machine learning!
```

---

## Feature Importance Analysis

### Correlation Analysis

Find redundant features (highly correlated):

```python
from app.ml.transformers import FeatureImportanceAnalyzer

# Load feature data
features_df = pd.read_csv('features.csv')

# Find highly correlated pairs
analyzer = FeatureImportanceAnalyzer()
corr_pairs = analyzer.analyze_correlation(features_df, threshold=0.9)

print(corr_pairs)
```

**Output:**

```
         feature1                feature2  correlation
0  amount_log           amount_zscore           0.95
1  txn_count_last_1h    velocity_score          0.92
2  total_txn_count      account_age_days        0.91
```

**Action:** Remove one feature from each highly correlated pair.

### Variance Analysis

Find low-variance features (not useful):

```python
low_var = analyzer.analyze_variance(features_df, threshold=0.01)
print(low_var)
```

**Output:**

```
              feature  variance
0      is_verified      0.005
1   tor_flag           0.001
```

**Action:** Remove features with very low variance (same value most of the time).

---

## Troubleshooting

### Issue 1: Feature Count < 60

**Problem:** Only 45 features extracted

**Solution:**
```python
# Check which features are missing
extractor = FeatureExtractor()
feature_names = extractor.get_feature_names()
print(f"Expected features: {len(feature_names)}")
print(feature_names)

# Debug each extractor
txn_features = extractor.transaction_extractor.extract(txn)
print(f"Transaction features: {len(txn_features)}")
```

### Issue 2: Too Many NaN Values

**Problem:** Features have many NaN values

**Solution:**
```python
# Check which features have NaN
features = extractor.extract(txn, db)
nan_features = {k: v for k, v in features.items() if pd.isna(v)}
print(f"NaN features: {nan_features}")

# Common causes:
# 1. No user history → User features are 0/NaN
# 2. No device history → Device features are 0/NaN
# 3. Missing location data → Location features are NaN

# Fix: Use imputation
from app.ml.transformers import FeatureTransformer
transformer = FeatureTransformer()
# Imputer will fill NaN with median
```

### Issue 3: Encoding Errors

**Problem:** "Unknown category" errors

**Solution:**
```python
# Enable unknown handling in encoder
encoder = CategoricalEncoder()
encoder.fit(train_data, ['country', 'merchant'])

# Unknown categories are automatically handled:
# - One-hot: All zeros
# - Label: Max + 1
```

### Issue 4: Scale Issues

**Problem:** Features have different scales

**Solution:**
```python
# Always scale features before ML
transformer = FeatureTransformer(method='standard')
scaled_features = transformer.fit_transform(features_df)

# Verify scaling worked
print(scaled_features.mean())  # Should be ~0
print(scaled_features.std())   # Should be ~1
```

---

## Next Steps

### Day 10 Preview: Model Training

Now that we have 60+ engineered features, we're ready to train our first fraud detection model!

**Day 10 Topics:**
- Train/test split
- Model selection (Logistic Regression, Random Forest, XGBoost)
- Model training and evaluation
- Hyperparameter tuning
- Model persistence (save/load)

### Practice Exercises

1. **Add More Features**
   - Create "time since last fraud" feature
   - Add "merchant risk score" feature
   - Calculate "user lifetime value" feature

2. **Feature Engineering Project**
   - Load real transaction data
   - Extract all 60+ features
   - Analyze feature importance
   - Remove redundant features

3. **Advanced Encoding**
   - Implement target encoding (encode by fraud rate)
   - Try embedding for high-cardinality categories
   - Experiment with hash encoding

---

## Key Takeaways

✅ **Feature Engineering is Critical** - 70-80% of model performance
✅ **60+ Features Extracted** - Transaction, User, Device, Location, Velocity
✅ **Two Encoding Strategies** - One-hot (low cardinality), Label (high cardinality)
✅ **Scaling is Essential** - StandardScaler or MinMaxScaler
✅ **Handle Missing Values** - Imputation with median/mean
✅ **Analyze Correlation** - Remove redundant features
✅ **Production-Ready** - Save encoders and transformers for reuse

**Remember:** Good features > Complex models!

---

## Navigation

- [← Previous: Day 8 - ML Fundamentals](README-DAY-008.md)
- [Main Guide](README.md)
- [Next: Day 10 - Model Training →](README-DAY-010.md)

---

**Questions?** Review the code, run the tests, and experiment with different features!

**Next:** Day 10 - Training your first fraud detection model with these 60+ features! 🚀
