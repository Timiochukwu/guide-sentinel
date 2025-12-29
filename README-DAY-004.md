# Day 4: First Fraud Detection Rules & Rules Engine

**Navigation:** [← Previous: Day 3](README-DAY-003.md) | [Main Guide](README.md) | [Next: Day 5 →](README-DAY-005.md)

---

## Table of Contents

1. [Overview](#overview)
2. [Understanding Rule-Based Fraud Detection](#understanding-rule-based-fraud-detection)
3. [Rules Engine Architecture](#rules-engine-architecture)
4. [Project Structure](#project-structure)
5. [Complete Code Implementation](#complete-code-implementation)
6. [Understanding Each Component](#understanding-each-component)
7. [Testing the Rules Engine](#testing-the-rules-engine)
8. [Rule Configuration & Tuning](#rule-configuration--tuning)
9. [Troubleshooting](#troubleshooting)
10. [Next Steps](#next-steps)

---

## Overview

### What You'll Build Today

Welcome to Day 4! Today we're building the **heart of fraud detection** - a flexible rules engine that can evaluate transactions against multiple fraud detection rules.

**Day 4 Objectives:**
- ✅ Build a flexible rules engine framework
- ✅ Implement 3 fundamental fraud detection rules
- ✅ Understand rule-based fraud detection concepts
- ✅ Test rules with real transaction data
- ✅ Learn how to add new rules easily

**The Three Rules We'll Implement:**

1. **Velocity Rule** - Detects too many transactions in a short time period
   - Example: User makes 10 purchases in 5 minutes → HIGH RISK
   - Real-world: Catches stolen credit cards being maxed out quickly

2. **Amount Threshold Rule** - Flags unusually large transaction amounts
   - Example: User's average purchase is $50, suddenly buys $5,000 item → MEDIUM RISK
   - Real-world: Detects account takeover or unusual spending patterns

3. **Location Mismatch Rule** - Identifies impossible travel patterns
   - Example: Transaction in New York at 10:00 AM, then Paris at 10:15 AM → HIGH RISK
   - Real-world: Catches card-not-present fraud and account sharing

**Time Estimate:** 2-3 hours

**Prerequisites:**
- Completed Days 1-3 (FastAPI, Database, Schemas)
- Understanding of Pydantic models
- Basic SQL knowledge
- Curl or Postman for testing

**No New Packages Required:**
We'll use only the packages already installed in Days 1-3:
- FastAPI (web framework)
- SQLAlchemy (database ORM)
- Pydantic (data validation)

---

## Understanding Rule-Based Fraud Detection

### What is Rule-Based Fraud Detection?

**Rule-based fraud detection** is the oldest and most transparent approach to fighting fraud. It works by defining explicit rules (if-then conditions) that flag suspicious behavior.

**How It Works:**

```
IF condition is met THEN flag as suspicious

Examples:
- IF transaction_amount > $10,000 THEN flag HIGH_RISK
- IF transaction_count_last_hour > 10 THEN flag HIGH_RISK
- IF location_distance > 500 miles AND time_diff < 1 hour THEN flag HIGH_RISK
```

### Rules vs Machine Learning

Understanding when to use rules vs ML is crucial for building effective fraud detection:

| Aspect | Rule-Based | Machine Learning |
|--------|-----------|------------------|
| **Transparency** | ✅ Fully explainable | ⚠️ Black box |
| **Setup Time** | ✅ Hours/Days | ⚠️ Weeks/Months |
| **Requires Training Data** | ✅ No | ❌ Yes (thousands of examples) |
| **Adapts to New Patterns** | ❌ Manual updates | ✅ Automatic |
| **False Positives** | ⚠️ Can be high | ✅ Lower with tuning |
| **Regulatory Compliance** | ✅ Easy to explain | ⚠️ Complex |
| **Maintenance** | ⚠️ Requires tuning | ⚠️ Requires retraining |

**Best Practice:** Use **both** approaches together!
- **Rules** for known fraud patterns (velocity checks, amount limits)
- **ML** for discovering new patterns and anomalies

### Real-World Example: Stripe's Approach

Stripe (payment processor handling billions in transactions) uses:
1. **100+ rules** for known fraud patterns
2. **ML models** for anomaly detection
3. **Human review** for high-value edge cases

**Why start with rules?**
- Get protection immediately (no training needed)
- Build training data for future ML models
- Handle compliance requirements (explainability)
- Catch 60-80% of fraud with well-tuned rules

---

## Rules Engine Architecture

### Design Pattern: Strategy Pattern

Our rules engine uses the **Strategy Pattern** - a behavioral design pattern that lets you define a family of algorithms (rules), encapsulate each one, and make them interchangeable.

**Without Strategy Pattern (Bad):**

```python
def check_fraud(transaction):
    # All rules hardcoded in one function
    if transaction.amount > 10000:
        return "HIGH_RISK"
    if count_recent_transactions(transaction.user_id) > 10:
        return "HIGH_RISK"
    if check_location_impossible(transaction):
        return "HIGH_RISK"
    # Hard to add new rules, hard to test, hard to maintain
```

**With Strategy Pattern (Good):**

```python
class RulesEngine:
    def __init__(self):
        self.rules = [
            AmountRule(),
            VelocityRule(),
            LocationRule(),
            # Easy to add new rules!
        ]

    def evaluate(self, transaction):
        results = []
        for rule in self.rules:
            result = rule.evaluate(transaction)
            results.append(result)
        return results
```

### Components of Our Rules Engine

```
┌─────────────────────────────────────────────────────────────┐
│                    Rules Engine                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  evaluate(transaction) → List[RuleResult]           │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │  Rule 1  │  │  Rule 2  │  │  Rule 3  │  ... add more   │
│  └──────────┘  └──────────┘  └──────────┘                 │
└─────────────────────────────────────────────────────────────┘

Each Rule:
  ├── evaluate(transaction) → RuleResult
  ├── name: str
  ├── priority: int
  └── enabled: bool

RuleResult:
  ├── rule_name: str
  ├── triggered: bool
  ├── risk_score: int (0-100)
  ├── reason: str
  └── metadata: dict
```

### Rule Lifecycle

```
1. Transaction arrives via API
   ↓
2. Rules Engine receives transaction
   ↓
3. For each enabled rule:
   a. Rule evaluates transaction
   b. Rule returns RuleResult
   c. Store result in database
   ↓
4. Aggregate results:
   - Calculate total risk score
   - Determine risk level (LOW/MEDIUM/HIGH)
   - Generate explanation
   ↓
5. Return fraud check response
```

### Rule Priority and Scoring

**Priority:** Determines the order rules are evaluated (1 = highest priority)

**Risk Score:** Each rule assigns a score (0-100):
- **0-30:** Low risk
- **31-60:** Medium risk
- **61-100:** High risk

**Score Aggregation Methods:**

1. **Maximum Score** (we'll use this):
   ```python
   final_score = max([result.risk_score for result in results])
   ```
   - Pros: Simple, conservative (flags anything suspicious)
   - Cons: One rule can dominate

2. **Weighted Average**:
   ```python
   final_score = sum([r.risk_score * r.weight for r in results]) / total_weight
   ```
   - Pros: Balanced, considers all rules
   - Cons: Can miss critical single-rule violations

3. **Additive with Cap**:
   ```python
   final_score = min(100, sum([r.risk_score for r in results]))
   ```
   - Pros: Multiple moderate risks accumulate
   - Cons: Can escalate too quickly

---

## Project Structure

### Directory Layout

```
sentinel/
├── app/
│   ├── __init__.py                 # From Day 1
│   ├── main.py                     # From Day 1
│   ├── database.py                 # From Day 2
│   ├── models.py                   # From Day 2
│   ├── schemas.py                  # From Day 3
│   ├── config.py                   # From Day 3
│   │
│   └── rules/                      # NEW - Day 4
│       ├── __init__.py             # Rules package
│       ├── base.py                 # Base rule class (abstract)
│       ├── engine.py               # Rules engine
│       ├── velocity_rule.py        # Rule 1: Transaction velocity
│       ├── amount_rule.py          # Rule 2: Large amounts
│       └── location_rule.py        # Rule 3: Location checks
│
├── .env                            # From Day 1
├── requirements.txt                # From Day 1
└── README.md                       # From Day 1
```

### File Responsibilities

| File | Responsibility | Lines of Code |
|------|----------------|---------------|
| `__init__.py` | Package exports | ~10 |
| `base.py` | Abstract base rule class | ~80 |
| `engine.py` | Rules engine orchestration | ~120 |
| `velocity_rule.py` | Velocity fraud detection | ~90 |
| `amount_rule.py` | Amount threshold detection | ~80 |
| `location_rule.py` | Location/travel detection | ~110 |

---

## Complete Code Implementation

### 1. app/rules/__init__.py

This file makes `rules` a Python package and exports the main components for easy importing.

```python
"""
Sentinel Rules Engine Package

This package contains the fraud detection rules engine and individual
rule implementations. The rules engine evaluates transactions against
multiple rules to detect fraudulent activity.

Architecture:
- BaseRule: Abstract base class for all rules
- RulesEngine: Orchestrates rule evaluation
- Individual Rules: Specific fraud detection logic

Usage:
    from app.rules import RulesEngine

    engine = RulesEngine()
    results = await engine.evaluate(transaction_request, db)
"""

from app.rules.base import BaseRule, RuleResult
from app.rules.engine import RulesEngine
from app.rules.velocity_rule import VelocityRule
from app.rules.amount_rule import AmountThresholdRule
from app.rules.location_rule import LocationMismatchRule

__all__ = [
    "BaseRule",
    "RuleResult",
    "RulesEngine",
    "VelocityRule",
    "AmountThresholdRule",
    "LocationMismatchRule",
]
```

**Explanation:**
- `__all__`: Defines what gets exported when someone does `from app.rules import *`
- We export the base classes and all rule implementations
- Clean imports: `from app.rules import RulesEngine` instead of `from app.rules.engine import RulesEngine`

---

### 2. app/rules/base.py

The foundation of our rules engine - defines the interface that all rules must implement.

```python
"""
Base Rule Class and Result Model

This module defines the abstract base class for all fraud detection rules
and the data model for rule evaluation results.

Every rule must:
1. Inherit from BaseRule
2. Implement the evaluate() method
3. Return a RuleResult object

The BaseRule uses Python's abc (Abstract Base Class) module to enforce
that all concrete rules implement the evaluate() method.
"""

from abc import ABC, abstractmethod
from typing import Any, Dict, Optional
from datetime import datetime
from pydantic import BaseModel, Field
from sqlalchemy.ext.asyncio import AsyncSession

# Import will be used by concrete rules
from app.schemas import TransactionCheckRequest


class RuleResult(BaseModel):
    """
    Result of a single rule evaluation.

    This model encapsulates everything we need to know about
    how a rule evaluated a transaction.

    Attributes:
        rule_name: Name of the rule that generated this result
        triggered: Whether the rule flagged the transaction as suspicious
        risk_score: Numeric risk score (0-100)
                   0-30: Low risk
                   31-60: Medium risk
                   61-100: High risk
        reason: Human-readable explanation of why the rule triggered
        metadata: Additional data about the evaluation (thresholds, counts, etc.)
        evaluated_at: Timestamp when the rule was evaluated
    """
    rule_name: str = Field(..., description="Name of the rule")
    triggered: bool = Field(..., description="Whether rule flagged transaction")
    risk_score: int = Field(..., ge=0, le=100, description="Risk score 0-100")
    reason: str = Field(..., description="Explanation of the result")
    metadata: Dict[str, Any] = Field(default_factory=dict, description="Additional data")
    evaluated_at: datetime = Field(default_factory=datetime.utcnow, description="Evaluation timestamp")

    class Config:
        json_schema_extra = {
            "example": {
                "rule_name": "VelocityRule",
                "triggered": True,
                "risk_score": 75,
                "reason": "10 transactions in last 5 minutes exceeds threshold of 5",
                "metadata": {
                    "transaction_count": 10,
                    "time_window_minutes": 5,
                    "threshold": 5
                },
                "evaluated_at": "2024-01-15T10:30:00Z"
            }
        }


class BaseRule(ABC):
    """
    Abstract base class for all fraud detection rules.

    This class defines the interface that all rules must implement.
    Using Python's ABC (Abstract Base Class) ensures that any class
    inheriting from BaseRule MUST implement the evaluate() method.

    Attributes:
        name: Human-readable rule name
        description: What the rule detects
        priority: Evaluation order (1 = highest priority)
        enabled: Whether the rule is active
    """

    def __init__(
        self,
        name: str,
        description: str,
        priority: int = 10,
        enabled: bool = True
    ):
        """
        Initialize the base rule.

        Args:
            name: Rule name (e.g., "VelocityRule")
            description: What the rule detects
            priority: Lower numbers = higher priority (1 = first)
            enabled: Whether to evaluate this rule
        """
        self.name = name
        self.description = description
        self.priority = priority
        self.enabled = enabled

    @abstractmethod
    async def evaluate(
        self,
        transaction: TransactionCheckRequest,
        db: AsyncSession
    ) -> RuleResult:
        """
        Evaluate a transaction against this rule.

        This is an abstract method - it MUST be implemented by all
        concrete rule classes.

        Args:
            transaction: The transaction to evaluate
            db: Database session for querying historical data

        Returns:
            RuleResult with evaluation outcome

        Raises:
            NotImplementedError: If a concrete rule doesn't implement this
        """
        pass

    def __repr__(self) -> str:
        """String representation of the rule."""
        return f"<{self.name} priority={self.priority} enabled={self.enabled}>"

    def __lt__(self, other: 'BaseRule') -> bool:
        """
        Enable sorting rules by priority.

        Lower priority number = higher priority = evaluated first.
        This allows us to do: sorted(rules) to get priority order.
        """
        return self.priority < other.priority
```

**Key Concepts:**

1. **Abstract Base Class (ABC):**
   ```python
   from abc import ABC, abstractmethod

   class BaseRule(ABC):
       @abstractmethod
       async def evaluate(...):
           pass
   ```
   - Forces all child classes to implement `evaluate()`
   - Attempts to instantiate BaseRule directly will raise `TypeError`

2. **Pydantic BaseModel for Results:**
   - `RuleResult` uses Pydantic for automatic validation
   - Ensures all results have consistent structure
   - Easy serialization to JSON for API responses

3. **Priority System:**
   - Rules execute in priority order (1 = first)
   - `__lt__` method enables: `sorted(rules)` to sort by priority

---

### 3. app/rules/engine.py

The orchestrator that runs all rules and aggregates results.

```python
"""
Rules Engine - Orchestrates Fraud Detection Rules

The RulesEngine is responsible for:
1. Managing a collection of fraud detection rules
2. Evaluating transactions against all enabled rules
3. Aggregating results into a final risk assessment
4. Providing detailed explanations for fraud decisions

The engine runs rules in priority order and collects results,
then uses those results to calculate an overall risk score
and risk level.
"""

from typing import List, Dict, Any
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime

from app.rules.base import BaseRule, RuleResult
from app.rules.velocity_rule import VelocityRule
from app.rules.amount_rule import AmountThresholdRule
from app.rules.location_rule import LocationMismatchRule
from app.schemas import TransactionCheckRequest


class RulesEngine:
    """
    Central engine for evaluating fraud detection rules.

    The engine maintains a list of rules and evaluates each one
    against incoming transactions. Results are aggregated to
    determine overall risk.

    Example:
        engine = RulesEngine()
        results = await engine.evaluate(transaction, db)
        risk_score = results['risk_score']
    """

    def __init__(self):
        """
        Initialize the rules engine with default rules.

        Rules are stored in priority order (lowest number = highest priority).
        You can easily add new rules by appending to self.rules.
        """
        self.rules: List[BaseRule] = [
            VelocityRule(priority=1),           # Check velocity first
            AmountThresholdRule(priority=2),    # Then amounts
            LocationMismatchRule(priority=3),   # Then location
        ]

        # Sort rules by priority (lowest number first)
        self.rules.sort()

    def add_rule(self, rule: BaseRule) -> None:
        """
        Add a new rule to the engine.

        Args:
            rule: Rule instance to add

        Example:
            engine.add_rule(MyCustomRule(priority=4))
        """
        self.rules.append(rule)
        self.rules.sort()  # Re-sort after adding

    def remove_rule(self, rule_name: str) -> bool:
        """
        Remove a rule by name.

        Args:
            rule_name: Name of the rule to remove

        Returns:
            True if rule was removed, False if not found
        """
        initial_count = len(self.rules)
        self.rules = [r for r in self.rules if r.name != rule_name]
        return len(self.rules) < initial_count

    def enable_rule(self, rule_name: str) -> bool:
        """
        Enable a rule by name.

        Args:
            rule_name: Name of the rule to enable

        Returns:
            True if rule was found and enabled
        """
        for rule in self.rules:
            if rule.name == rule_name:
                rule.enabled = True
                return True
        return False

    def disable_rule(self, rule_name: str) -> bool:
        """
        Disable a rule by name.

        Args:
            rule_name: Name of the rule to disable

        Returns:
            True if rule was found and disabled
        """
        for rule in self.rules:
            if rule.name == rule_name:
                rule.enabled = False
                return True
        return False

    async def evaluate(
        self,
        transaction: TransactionCheckRequest,
        db: AsyncSession
    ) -> Dict[str, Any]:
        """
        Evaluate a transaction against all enabled rules.

        This is the main entry point for fraud detection. It:
        1. Runs each enabled rule in priority order
        2. Collects all RuleResults
        3. Calculates aggregate risk score
        4. Determines risk level (LOW/MEDIUM/HIGH)
        5. Generates explanation

        Args:
            transaction: Transaction to evaluate
            db: Database session for queries

        Returns:
            Dictionary containing:
            - rule_results: List of individual rule results
            - risk_score: Overall risk score (0-100)
            - risk_level: LOW/MEDIUM/HIGH
            - triggered_rules: List of rule names that flagged transaction
            - explanation: Human-readable summary
            - evaluation_time: When evaluation occurred
        """
        results: List[RuleResult] = []

        # Evaluate each enabled rule
        for rule in self.rules:
            if not rule.enabled:
                continue

            try:
                result = await rule.evaluate(transaction, db)
                results.append(result)
            except Exception as e:
                # Log error but don't fail entire evaluation
                # In production, you'd log this to a monitoring system
                print(f"Error evaluating rule {rule.name}: {str(e)}")
                # Add a failed result so we know the rule didn't run
                results.append(RuleResult(
                    rule_name=rule.name,
                    triggered=False,
                    risk_score=0,
                    reason=f"Rule evaluation failed: {str(e)}",
                    metadata={"error": str(e)}
                ))

        # Aggregate results
        return self._aggregate_results(results, transaction)

    def _aggregate_results(
        self,
        results: List[RuleResult],
        transaction: TransactionCheckRequest
    ) -> Dict[str, Any]:
        """
        Aggregate individual rule results into overall assessment.

        Aggregation strategy: Use the MAXIMUM risk score from any rule.

        Alternative strategies you could implement:
        - Weighted average of all scores
        - Sum of scores (capped at 100)
        - Custom logic based on rule combinations

        Args:
            results: List of rule results
            transaction: Original transaction (for context)

        Returns:
            Aggregated fraud assessment
        """
        # Calculate overall risk score (use maximum)
        risk_score = max([r.risk_score for r in results]) if results else 0

        # Determine risk level based on score
        if risk_score >= 61:
            risk_level = "HIGH"
        elif risk_score >= 31:
            risk_level = "MEDIUM"
        else:
            risk_level = "LOW"

        # Find which rules triggered
        triggered_rules = [
            r.rule_name for r in results if r.triggered
        ]

        # Generate explanation
        explanation = self._generate_explanation(
            results, risk_score, risk_level, triggered_rules
        )

        return {
            "rule_results": [r.dict() for r in results],
            "risk_score": risk_score,
            "risk_level": risk_level,
            "triggered_rules": triggered_rules,
            "explanation": explanation,
            "evaluation_time": datetime.utcnow().isoformat(),
            "transaction_id": transaction.transaction_id,
            "user_id": transaction.user_info.user_id,
        }

    def _generate_explanation(
        self,
        results: List[RuleResult],
        risk_score: int,
        risk_level: str,
        triggered_rules: List[str]
    ) -> str:
        """
        Generate human-readable explanation of fraud decision.

        This is crucial for:
        - Customer service (explaining why a transaction was declined)
        - Compliance (audit trail)
        - Debugging (understanding rule behavior)

        Args:
            results: All rule results
            risk_score: Overall risk score
            risk_level: Overall risk level
            triggered_rules: Rules that flagged transaction

        Returns:
            Explanation string
        """
        if not triggered_rules:
            return (
                f"Transaction appears legitimate. Risk score: {risk_score}/100 ({risk_level}). "
                "No fraud indicators detected."
            )

        # Build detailed explanation
        parts = [
            f"Transaction flagged as {risk_level} risk (score: {risk_score}/100).",
            f"Triggered {len(triggered_rules)} rule(s): {', '.join(triggered_rules)}.",
            "",
            "Details:"
        ]

        for result in results:
            if result.triggered:
                parts.append(f"- {result.rule_name}: {result.reason}")

        return "\n".join(parts)

    def get_rule_status(self) -> List[Dict[str, Any]]:
        """
        Get status of all rules in the engine.

        Useful for debugging and monitoring.

        Returns:
            List of rule status dictionaries
        """
        return [
            {
                "name": rule.name,
                "description": rule.description,
                "priority": rule.priority,
                "enabled": rule.enabled,
            }
            for rule in self.rules
        ]
```

**Key Features:**

1. **Extensibility:**
   ```python
   engine.add_rule(MyCustomRule())
   engine.remove_rule("VelocityRule")
   ```

2. **Error Handling:**
   - If one rule fails, others still run
   - Failures are logged but don't crash the system

3. **Aggregation Strategy:**
   - Currently uses MAX score (most conservative)
   - Easy to change to weighted average or other methods

---

### 4. app/rules/velocity_rule.py

Detects when a user makes too many transactions in a short time period.

```python
"""
Velocity Rule - Transaction Frequency Detection

This rule detects when a user makes an unusually high number of
transactions in a short time period. This is a common indicator of:

1. Stolen card being maxed out quickly
2. Testing stolen card numbers (card testing)
3. Account takeover with rapid fund extraction
4. Bot/automated fraud attacks

The rule queries the database for recent transactions by the same
user and counts how many occurred within a time window.

Configuration:
- time_window_minutes: How far back to look (default: 10 minutes)
- threshold: Maximum allowed transactions in window (default: 5)
- high_risk_threshold: Triggers high risk score (default: 10)
"""

from datetime import datetime, timedelta
from typing import Any, Dict
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func, and_

from app.rules.base import BaseRule, RuleResult
from app.schemas import TransactionCheckRequest
from app.models import FraudTransaction


class VelocityRule(BaseRule):
    """
    Detects suspicious transaction velocity (frequency).

    Example scenarios:
    - Normal: User makes 2 purchases in 10 minutes → PASS
    - Suspicious: User makes 6 purchases in 10 minutes → MEDIUM RISK
    - Fraudulent: User makes 15 purchases in 10 minutes → HIGH RISK
    """

    def __init__(
        self,
        time_window_minutes: int = 10,
        threshold: int = 5,
        high_risk_threshold: int = 10,
        priority: int = 1,
        enabled: bool = True
    ):
        """
        Initialize velocity rule with thresholds.

        Args:
            time_window_minutes: Time window to check (default: 10 min)
            threshold: Max transactions before flagging (default: 5)
            high_risk_threshold: Threshold for high risk score (default: 10)
            priority: Rule priority (default: 1 = highest)
            enabled: Whether rule is active (default: True)
        """
        super().__init__(
            name="VelocityRule",
            description="Detects too many transactions in short time period",
            priority=priority,
            enabled=enabled
        )
        self.time_window_minutes = time_window_minutes
        self.threshold = threshold
        self.high_risk_threshold = high_risk_threshold

    async def evaluate(
        self,
        transaction: TransactionCheckRequest,
        db: AsyncSession
    ) -> RuleResult:
        """
        Evaluate transaction velocity for this user.

        Logic:
        1. Calculate time window (now - time_window_minutes)
        2. Query database for user's transactions in window
        3. Count transactions
        4. If count > threshold, flag as suspicious
        5. Calculate risk score based on how far over threshold

        Args:
            transaction: Transaction to evaluate
            db: Database session

        Returns:
            RuleResult with velocity assessment
        """
        user_id = transaction.user_info.user_id

        # Calculate time window
        now = datetime.utcnow()
        window_start = now - timedelta(minutes=self.time_window_minutes)

        # Query database for recent transactions by this user
        # Note: We're querying FraudTransaction table created in Day 2
        query = select(func.count(FraudTransaction.id)).where(
            and_(
                FraudTransaction.user_id == user_id,
                FraudTransaction.timestamp >= window_start,
                FraudTransaction.timestamp <= now
            )
        )

        result = await db.execute(query)
        transaction_count = result.scalar() or 0

        # Add 1 for current transaction
        transaction_count += 1

        # Determine if rule triggers
        triggered = transaction_count > self.threshold

        # Calculate risk score
        if not triggered:
            risk_score = 0
            reason = (
                f"Transaction velocity normal: {transaction_count} transactions "
                f"in last {self.time_window_minutes} minutes (threshold: {self.threshold})"
            )
        else:
            # Calculate how far over threshold
            overage = transaction_count - self.threshold

            # High risk if over high_risk_threshold
            if transaction_count >= self.high_risk_threshold:
                risk_score = 80
                reason = (
                    f"CRITICAL: {transaction_count} transactions in "
                    f"{self.time_window_minutes} minutes (threshold: {self.threshold})"
                )
            else:
                # Scale risk score based on overage
                # 1 over = 40, 2 over = 50, 3 over = 60, etc.
                risk_score = min(75, 30 + (overage * 10))
                reason = (
                    f"Elevated transaction velocity: {transaction_count} transactions "
                    f"in {self.time_window_minutes} minutes exceeds threshold of {self.threshold}"
                )

        # Build metadata for transparency
        metadata: Dict[str, Any] = {
            "transaction_count": transaction_count,
            "time_window_minutes": self.time_window_minutes,
            "threshold": self.threshold,
            "high_risk_threshold": self.high_risk_threshold,
            "window_start": window_start.isoformat(),
            "overage": transaction_count - self.threshold if triggered else 0,
        }

        return RuleResult(
            rule_name=self.name,
            triggered=triggered,
            risk_score=risk_score,
            reason=reason,
            metadata=metadata
        )
```

**How It Works:**

1. **Query Recent Transactions:**
   ```sql
   SELECT COUNT(*) FROM fraud_transactions
   WHERE user_id = ?
     AND timestamp >= (NOW() - INTERVAL '10 minutes')
     AND timestamp <= NOW()
   ```

2. **Calculate Risk:**
   - 0-5 transactions: No risk (score = 0)
   - 6-9 transactions: Medium risk (score = 40-70)
   - 10+ transactions: High risk (score = 80)

3. **Metadata:** Includes all parameters for debugging

---

### 5. app/rules/amount_rule.py

Flags transactions with unusually large amounts.

```python
"""
Amount Threshold Rule - Large Transaction Detection

This rule detects transactions with unusually large amounts.
Large amounts are risky because:

1. Fraudsters try to maximize stolen value quickly
2. Account takeover often involves large transfers
3. Money laundering uses large transactions
4. Testing limits of compromised accounts

The rule uses two approaches:
1. Absolute threshold: Flag any transaction over $X
2. Relative threshold: Flag if transaction is Y times larger than user's average

Configuration:
- absolute_threshold: Hard limit (default: $10,000)
- relative_multiplier: Multiplier of user average (default: 10x)
"""

from typing import Any, Dict
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func

from app.rules.base import BaseRule, RuleResult
from app.schemas import TransactionCheckRequest
from app.models import FraudTransaction


class AmountThresholdRule(BaseRule):
    """
    Detects unusually large transaction amounts.

    Example scenarios:
    - Normal: User buys $50 item, average = $45 → PASS
    - Suspicious: User buys $5,000 item, average = $50 → HIGH RISK
    - Suspicious: User buys $15,000 item → HIGH RISK (absolute threshold)
    """

    def __init__(
        self,
        absolute_threshold: float = 10000.0,
        relative_multiplier: float = 10.0,
        min_transactions_for_average: int = 3,
        priority: int = 2,
        enabled: bool = True
    ):
        """
        Initialize amount threshold rule.

        Args:
            absolute_threshold: Flag any transaction over this amount
            relative_multiplier: Flag if amount > (user_avg * multiplier)
            min_transactions_for_average: Need N past transactions to calc average
            priority: Rule priority (default: 2)
            enabled: Whether rule is active (default: True)
        """
        super().__init__(
            name="AmountThresholdRule",
            description="Detects unusually large transaction amounts",
            priority=priority,
            enabled=enabled
        )
        self.absolute_threshold = absolute_threshold
        self.relative_multiplier = relative_multiplier
        self.min_transactions_for_average = min_transactions_for_average

    async def evaluate(
        self,
        transaction: TransactionCheckRequest,
        db: AsyncSession
    ) -> RuleResult:
        """
        Evaluate if transaction amount is suspicious.

        Logic:
        1. Check absolute threshold (e.g., > $10,000)
        2. Calculate user's average transaction amount
        3. Check relative threshold (e.g., > 10x average)
        4. Assign risk score based on severity

        Args:
            transaction: Transaction to evaluate
            db: Database session

        Returns:
            RuleResult with amount assessment
        """
        amount = transaction.amount
        user_id = transaction.user_info.user_id

        # Check 1: Absolute threshold
        absolute_triggered = amount > self.absolute_threshold

        # Check 2: Relative threshold (compare to user's average)
        relative_triggered = False
        user_average = None

        # Query user's historical average transaction amount
        query = select(
            func.avg(FraudTransaction.amount),
            func.count(FraudTransaction.id)
        ).where(
            FraudTransaction.user_id == user_id
        )

        result = await db.execute(query)
        row = result.first()

        if row:
            avg_amount, transaction_count = row

            # Only use relative check if user has enough history
            if transaction_count >= self.min_transactions_for_average and avg_amount:
                user_average = float(avg_amount)
                relative_triggered = amount > (user_average * self.relative_multiplier)

        # Determine if rule triggers (either check)
        triggered = absolute_triggered or relative_triggered

        # Calculate risk score
        if not triggered:
            risk_score = 0
            reason = (
                f"Transaction amount ${amount:,.2f} is within normal limits "
                f"(absolute threshold: ${self.absolute_threshold:,.2f}"
            )
            if user_average:
                reason += f", user average: ${user_average:,.2f}"
            reason += ")"
        else:
            # Determine reason and risk score
            reasons = []

            if absolute_triggered:
                # High risk for exceeding absolute threshold
                risk_score = 85
                overage = amount - self.absolute_threshold
                reasons.append(
                    f"Exceeds absolute threshold by ${overage:,.2f} "
                    f"(${amount:,.2f} > ${self.absolute_threshold:,.2f})"
                )

            if relative_triggered and user_average:
                # Medium-high risk for exceeding relative threshold
                multiplier = amount / user_average
                if not absolute_triggered:
                    risk_score = 70
                reasons.append(
                    f"Amount is {multiplier:.1f}x user's average of ${user_average:,.2f} "
                    f"(threshold: {self.relative_multiplier}x)"
                )

            if not absolute_triggered and not relative_triggered:
                # Shouldn't happen, but defensive
                risk_score = 50
                reasons.append("Amount flagged by heuristic")

            reason = "Large transaction amount detected: " + "; ".join(reasons)

        # Build metadata
        metadata: Dict[str, Any] = {
            "amount": amount,
            "absolute_threshold": self.absolute_threshold,
            "absolute_triggered": absolute_triggered,
            "relative_threshold_multiplier": self.relative_multiplier,
            "relative_triggered": relative_triggered,
        }

        if user_average is not None:
            metadata["user_average_amount"] = user_average
            metadata["amount_to_average_ratio"] = amount / user_average if user_average > 0 else None

        return RuleResult(
            rule_name=self.name,
            triggered=triggered,
            risk_score=risk_score,
            reason=reason,
            metadata=metadata
        )
```

**How It Works:**

1. **Absolute Check:**
   ```python
   if amount > $10,000:
       trigger = True
       risk = 85
   ```

2. **Relative Check:**
   ```sql
   SELECT AVG(amount) FROM fraud_transactions WHERE user_id = ?
   ```
   ```python
   if amount > (user_average * 10):
       trigger = True
       risk = 70
   ```

3. **Risk Scoring:**
   - Absolute violation: Risk = 85
   - Relative violation only: Risk = 70
   - Both: Risk = 85 (use maximum)

---

### 6. app/rules/location_rule.py

Detects impossible travel patterns between transactions.

```python
"""
Location Mismatch Rule - Impossible Travel Detection

This rule detects when transactions occur in locations that are
physically impossible to travel between in the time elapsed.

Example:
- Transaction 1: New York at 10:00 AM
- Transaction 2: London at 10:30 AM
- Distance: ~3,500 miles
- Time: 30 minutes
- Impossible: Would require traveling at 7,000 mph!

This is a strong indicator of:
1. Card-not-present fraud (online fraud with stolen card details)
2. Account sharing/selling
3. Compromised credentials used from multiple locations

The rule uses the Haversine formula to calculate distance between
two geographic coordinates, then checks if the speed required to
travel that distance is physically possible.

Configuration:
- max_reasonable_speed_mph: Maximum plausible speed (default: 600 mph = jet)
- suspicious_speed_mph: Speed that triggers medium risk (default: 300 mph)
"""

import math
from typing import Any, Dict, Optional, Tuple
from datetime import datetime
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, desc

from app.rules.base import BaseRule, RuleResult
from app.schemas import TransactionCheckRequest
from app.models import FraudTransaction


class LocationMismatchRule(BaseRule):
    """
    Detects impossible travel patterns between transactions.

    Example scenarios:
    - Normal: NY at 10:00 AM, NJ at 11:00 AM (50 miles/1 hour) → PASS
    - Suspicious: NY at 10:00 AM, LA at 2:00 PM (2,500 miles/4 hours = 625 mph) → MEDIUM RISK
    - Fraudulent: NY at 10:00 AM, Tokyo at 10:05 AM (6,700 miles/5 min = 80,400 mph) → HIGH RISK
    """

    def __init__(
        self,
        max_reasonable_speed_mph: float = 600.0,
        suspicious_speed_mph: float = 300.0,
        min_distance_miles: float = 50.0,
        priority: int = 3,
        enabled: bool = True
    ):
        """
        Initialize location mismatch rule.

        Args:
            max_reasonable_speed_mph: Maximum plausible speed (600 = jet plane)
            suspicious_speed_mph: Speed that triggers flag (300 = fast car)
            min_distance_miles: Ignore short distances (default: 50 miles)
            priority: Rule priority (default: 3)
            enabled: Whether rule is active (default: True)
        """
        super().__init__(
            name="LocationMismatchRule",
            description="Detects impossible travel patterns",
            priority=priority,
            enabled=enabled
        )
        self.max_reasonable_speed_mph = max_reasonable_speed_mph
        self.suspicious_speed_mph = suspicious_speed_mph
        self.min_distance_miles = min_distance_miles

    async def evaluate(
        self,
        transaction: TransactionCheckRequest,
        db: AsyncSession
    ) -> RuleResult:
        """
        Evaluate if transaction location is suspicious.

        Logic:
        1. Get current transaction location
        2. Find user's most recent transaction with location
        3. Calculate distance between locations
        4. Calculate time elapsed
        5. Calculate required travel speed
        6. Flag if speed is impossible/suspicious

        Args:
            transaction: Transaction to evaluate
            db: Database session

        Returns:
            RuleResult with location assessment
        """
        user_id = transaction.user_info.user_id

        # Get current location
        current_lat = transaction.location.latitude
        current_lon = transaction.location.longitude
        current_time = datetime.utcnow()

        # Find user's most recent transaction with location data
        query = select(FraudTransaction).where(
            FraudTransaction.user_id == user_id
        ).order_by(
            desc(FraudTransaction.timestamp)
        ).limit(1)

        result = await db.execute(query)
        previous_transaction = result.scalar_one_or_none()

        # If no previous transaction, can't evaluate
        if not previous_transaction:
            return RuleResult(
                rule_name=self.name,
                triggered=False,
                risk_score=0,
                reason="No previous transaction found for location comparison",
                metadata={"first_transaction": True}
            )

        # Check if previous transaction has location data
        # Note: In Day 2, location might be stored in metadata JSONB field
        # For this example, we'll assume latitude/longitude columns exist
        # You may need to adapt based on your schema
        prev_lat = previous_transaction.metadata.get("latitude")
        prev_lon = previous_transaction.metadata.get("longitude")
        prev_time = previous_transaction.timestamp

        if prev_lat is None or prev_lon is None:
            return RuleResult(
                rule_name=self.name,
                triggered=False,
                risk_score=0,
                reason="Previous transaction has no location data",
                metadata={"previous_transaction_id": previous_transaction.transaction_id}
            )

        # Calculate distance and time difference
        distance_miles = self._calculate_distance(
            prev_lat, prev_lon, current_lat, current_lon
        )

        time_diff_seconds = (current_time - prev_time).total_seconds()
        time_diff_hours = time_diff_seconds / 3600

        # Avoid division by zero
        if time_diff_hours == 0:
            time_diff_hours = 0.01  # 36 seconds

        # Calculate required travel speed
        required_speed_mph = distance_miles / time_diff_hours

        # Evaluate if suspicious
        # Skip if distance is too small (local movements)
        if distance_miles < self.min_distance_miles:
            return RuleResult(
                rule_name=self.name,
                triggered=False,
                risk_score=0,
                reason=f"Transaction within local area ({distance_miles:.1f} miles)",
                metadata={
                    "distance_miles": distance_miles,
                    "time_diff_hours": time_diff_hours,
                    "required_speed_mph": required_speed_mph,
                }
            )

        # Check if impossible travel
        if required_speed_mph > self.max_reasonable_speed_mph:
            # Impossible travel - HIGH RISK
            risk_score = 90
            reason = (
                f"IMPOSSIBLE TRAVEL: {distance_miles:.0f} miles in "
                f"{time_diff_hours:.1f} hours requires {required_speed_mph:.0f} mph "
                f"(max reasonable: {self.max_reasonable_speed_mph:.0f} mph)"
            )
            triggered = True
        elif required_speed_mph > self.suspicious_speed_mph:
            # Suspicious travel - MEDIUM RISK
            risk_score = 60
            reason = (
                f"Suspicious travel speed: {distance_miles:.0f} miles in "
                f"{time_diff_hours:.1f} hours = {required_speed_mph:.0f} mph "
                f"(suspicious threshold: {self.suspicious_speed_mph:.0f} mph)"
            )
            triggered = True
        else:
            # Normal travel
            risk_score = 0
            reason = (
                f"Travel speed reasonable: {distance_miles:.0f} miles in "
                f"{time_diff_hours:.1f} hours = {required_speed_mph:.0f} mph"
            )
            triggered = False

        # Build metadata
        metadata: Dict[str, Any] = {
            "previous_transaction_id": previous_transaction.transaction_id,
            "previous_location": {
                "latitude": prev_lat,
                "longitude": prev_lon,
            },
            "current_location": {
                "latitude": current_lat,
                "longitude": current_lon,
            },
            "distance_miles": round(distance_miles, 2),
            "time_diff_seconds": round(time_diff_seconds, 2),
            "time_diff_hours": round(time_diff_hours, 2),
            "required_speed_mph": round(required_speed_mph, 2),
            "max_reasonable_speed_mph": self.max_reasonable_speed_mph,
            "suspicious_speed_mph": self.suspicious_speed_mph,
        }

        return RuleResult(
            rule_name=self.name,
            triggered=triggered,
            risk_score=risk_score,
            reason=reason,
            metadata=metadata
        )

    @staticmethod
    def _calculate_distance(
        lat1: float, lon1: float, lat2: float, lon2: float
    ) -> float:
        """
        Calculate distance between two coordinates using Haversine formula.

        The Haversine formula calculates the great-circle distance between
        two points on a sphere (Earth) given their longitudes and latitudes.

        Args:
            lat1: Latitude of point 1 (degrees)
            lon1: Longitude of point 1 (degrees)
            lat2: Latitude of point 2 (degrees)
            lon2: Longitude of point 2 (degrees)

        Returns:
            Distance in miles

        Math:
            a = sin²(Δφ/2) + cos(φ1)⋅cos(φ2)⋅sin²(Δλ/2)
            c = 2⋅atan2(√a, √(1−a))
            d = R⋅c

            where φ = latitude, λ = longitude, R = Earth's radius
        """
        # Earth's radius in miles
        earth_radius_miles = 3958.8

        # Convert latitude and longitude from degrees to radians
        lat1_rad = math.radians(lat1)
        lon1_rad = math.radians(lon1)
        lat2_rad = math.radians(lat2)
        lon2_rad = math.radians(lon2)

        # Haversine formula
        dlat = lat2_rad - lat1_rad
        dlon = lon2_rad - lon1_rad

        a = (
            math.sin(dlat / 2) ** 2 +
            math.cos(lat1_rad) * math.cos(lat2_rad) * math.sin(dlon / 2) ** 2
        )
        c = 2 * math.asin(math.sqrt(a))
        distance = earth_radius_miles * c

        return distance
```

**How It Works:**

1. **Haversine Formula:**
   ```
   Calculates great-circle distance between two points on Earth
   Accounts for Earth's curvature
   More accurate than simple Pythagorean distance
   ```

2. **Speed Calculation:**
   ```python
   distance = haversine(lat1, lon1, lat2, lon2)  # miles
   time = (timestamp2 - timestamp1) / 3600        # hours
   speed = distance / time                        # mph
   ```

3. **Risk Assessment:**
   - Speed > 600 mph: Impossible (risk = 90)
   - Speed > 300 mph: Suspicious (risk = 60)
   - Speed < 300 mph: Normal (risk = 0)
   - Distance < 50 miles: Skip check (local movement)

---

## Understanding Each Component

### How Rules Are Evaluated

**Step-by-step evaluation flow:**

```python
# 1. Transaction arrives
transaction = TransactionCheckRequest(
    transaction_id="tx_123",
    amount=5000.0,
    user_info=UserInfo(user_id="user_456"),
    location=LocationInfo(latitude=40.7128, longitude=-74.0060)
)

# 2. Create rules engine
engine = RulesEngine()

# 3. Engine evaluates all rules
results = await engine.evaluate(transaction, db)

# 4. Each rule runs independently
for rule in engine.rules:
    if rule.enabled:
        result = await rule.evaluate(transaction, db)
        # Returns RuleResult with risk score

# 5. Engine aggregates results
final_score = max(all_risk_scores)
risk_level = determine_level(final_score)

# 6. Return comprehensive response
return {
    "risk_score": final_score,
    "risk_level": risk_level,
    "triggered_rules": ["VelocityRule", "AmountThresholdRule"],
    "explanation": "..."
}
```

### Rule Independence

**Key principle:** Rules should be **independent** and **composable**.

**Bad Design (Dependent Rules):**
```python
class VelocityRule:
    def evaluate(self, transaction):
        # ❌ Bad: Depends on AmountRule result
        if AmountRule.evaluate(transaction).triggered:
            return high_risk()
```

**Good Design (Independent Rules):**
```python
class VelocityRule:
    def evaluate(self, transaction):
        # ✅ Good: Evaluates independently
        velocity = count_recent_transactions()
        if velocity > threshold:
            return high_risk()
```

**Why?**
- Can test rules in isolation
- Can enable/disable rules without breaking others
- Can run rules in parallel (future optimization)
- Easier to debug

### Database Queries in Rules

**Performance consideration:** Each rule may query the database.

**Current approach (Simple):**
```python
# Each rule queries independently
class VelocityRule:
    async def evaluate(self, transaction, db):
        # Query 1: Count recent transactions
        count = await db.execute(select(count()).where(...))

class AmountRule:
    async def evaluate(self, transaction, db):
        # Query 2: Get average amount
        avg = await db.execute(select(avg()).where(...))
```

**Optimized approach (Advanced - future improvement):**
```python
# Single query fetches all needed data
transaction_data = await fetch_user_history(user_id, db)

# Rules use cached data
class VelocityRule:
    def evaluate(self, transaction, transaction_data):
        count = len(transaction_data.recent)
```

**Trade-off:**
- Simple: Easier to understand, easier to add rules
- Optimized: Faster, but more complex

For Day 4, we use the **simple approach**. Optimization comes later.

---

## Testing the Rules Engine

### Integration with FastAPI

First, let's add an endpoint to `app/main.py` to use our rules engine:

```python
# Add to app/main.py

from app.rules import RulesEngine
from app.database import get_db

# Create global rules engine instance
rules_engine = RulesEngine()

@app.post(
    "/fraud/check",
    tags=["Fraud Detection"],
    summary="Check transaction for fraud",
    response_model=Dict[str, Any]
)
async def check_fraud(
    transaction: TransactionCheckRequest,
    db: AsyncSession = Depends(get_db)
):
    """
    Evaluate a transaction against all fraud detection rules.

    Returns:
        Fraud assessment with risk score, risk level, and explanation
    """
    results = await rules_engine.evaluate(transaction, db)
    return results
```

**Add this code to your existing `app/main.py` file after the `/info` endpoint.**

### Test 1: Normal Transaction (Should Pass)

```bash
curl -X POST "http://localhost:8000/fraud/check" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_001",
    "amount": 50.00,
    "currency": "USD",
    "timestamp": "2024-01-15T10:00:00Z",
    "merchant_info": {
      "merchant_id": "merch_123",
      "merchant_name": "Coffee Shop",
      "merchant_category": "food_drink"
    },
    "user_info": {
      "user_id": "user_001",
      "email": "john@example.com",
      "account_age_days": 365
    },
    "device": {
      "device_id": "dev_001",
      "device_type": "mobile",
      "os": "iOS",
      "ip_address": "192.168.1.1"
    },
    "location": {
      "latitude": 40.7128,
      "longitude": -74.0060,
      "city": "New York",
      "country": "US"
    }
  }'
```

**Expected Response:**

```json
{
  "rule_results": [
    {
      "rule_name": "VelocityRule",
      "triggered": false,
      "risk_score": 0,
      "reason": "Transaction velocity normal: 1 transactions in last 10 minutes (threshold: 5)",
      "metadata": {
        "transaction_count": 1,
        "time_window_minutes": 10,
        "threshold": 5,
        "overage": 0
      },
      "evaluated_at": "2024-01-15T10:00:00Z"
    },
    {
      "rule_name": "AmountThresholdRule",
      "triggered": false,
      "risk_score": 0,
      "reason": "Transaction amount $50.00 is within normal limits",
      "metadata": {
        "amount": 50.0,
        "absolute_threshold": 10000.0,
        "absolute_triggered": false
      },
      "evaluated_at": "2024-01-15T10:00:00Z"
    },
    {
      "rule_name": "LocationMismatchRule",
      "triggered": false,
      "risk_score": 0,
      "reason": "No previous transaction found for location comparison",
      "metadata": {
        "first_transaction": true
      },
      "evaluated_at": "2024-01-15T10:00:00Z"
    }
  ],
  "risk_score": 0,
  "risk_level": "LOW",
  "triggered_rules": [],
  "explanation": "Transaction appears legitimate. Risk score: 0/100 (LOW). No fraud indicators detected.",
  "evaluation_time": "2024-01-15T10:00:00.123456",
  "transaction_id": "tx_001",
  "user_id": "user_001"
}
```

### Test 2: High Amount (Should Trigger Amount Rule)

```bash
curl -X POST "http://localhost:8000/fraud/check" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_002",
    "amount": 15000.00,
    "currency": "USD",
    "timestamp": "2024-01-15T10:05:00Z",
    "merchant_info": {
      "merchant_id": "merch_456",
      "merchant_name": "Luxury Goods",
      "merchant_category": "retail"
    },
    "user_info": {
      "user_id": "user_001",
      "email": "john@example.com",
      "account_age_days": 365
    },
    "device": {
      "device_id": "dev_001",
      "device_type": "mobile",
      "os": "iOS",
      "ip_address": "192.168.1.1"
    },
    "location": {
      "latitude": 40.7128,
      "longitude": -74.0060,
      "city": "New York",
      "country": "US"
    }
  }'
```

**Expected Response:**

```json
{
  "rule_results": [
    {
      "rule_name": "VelocityRule",
      "triggered": false,
      "risk_score": 0,
      "reason": "Transaction velocity normal...",
      "metadata": {...}
    },
    {
      "rule_name": "AmountThresholdRule",
      "triggered": true,
      "risk_score": 85,
      "reason": "Large transaction amount detected: Exceeds absolute threshold by $5,000.00 ($15,000.00 > $10,000.00)",
      "metadata": {
        "amount": 15000.0,
        "absolute_threshold": 10000.0,
        "absolute_triggered": true,
        "relative_triggered": false
      },
      "evaluated_at": "2024-01-15T10:05:00Z"
    },
    {
      "rule_name": "LocationMismatchRule",
      "triggered": false,
      "risk_score": 0,
      "reason": "Transaction within local area...",
      "metadata": {...}
    }
  ],
  "risk_score": 85,
  "risk_level": "HIGH",
  "triggered_rules": ["AmountThresholdRule"],
  "explanation": "Transaction flagged as HIGH risk (score: 85/100).\nTriggered 1 rule(s): AmountThresholdRule.\n\nDetails:\n- AmountThresholdRule: Large transaction amount detected: Exceeds absolute threshold by $5,000.00",
  "evaluation_time": "2024-01-15T10:05:00.123456",
  "transaction_id": "tx_002",
  "user_id": "user_001"
}
```

### Test 3: Impossible Travel (Should Trigger Location Rule)

First, create a transaction in New York:

```bash
curl -X POST "http://localhost:8000/fraud/check" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_003",
    "amount": 100.00,
    "currency": "USD",
    "timestamp": "2024-01-15T10:00:00Z",
    "merchant_info": {
      "merchant_id": "merch_123",
      "merchant_name": "NYC Shop",
      "merchant_category": "retail"
    },
    "user_info": {
      "user_id": "user_002",
      "email": "jane@example.com",
      "account_age_days": 100
    },
    "device": {
      "device_id": "dev_002",
      "device_type": "mobile",
      "os": "Android",
      "ip_address": "192.168.1.2"
    },
    "location": {
      "latitude": 40.7128,
      "longitude": -74.0060,
      "city": "New York",
      "country": "US"
    }
  }'
```

Then, create a transaction in London 30 minutes later:

```bash
curl -X POST "http://localhost:8000/fraud/check" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_004",
    "amount": 100.00,
    "currency": "GBP",
    "timestamp": "2024-01-15T10:30:00Z",
    "merchant_info": {
      "merchant_id": "merch_789",
      "merchant_name": "London Shop",
      "merchant_category": "retail"
    },
    "user_info": {
      "user_id": "user_002",
      "email": "jane@example.com",
      "account_age_days": 100
    },
    "device": {
      "device_id": "dev_003",
      "device_type": "mobile",
      "os": "Android",
      "ip_address": "82.45.123.45"
    },
    "location": {
      "latitude": 51.5074,
      "longitude": -0.1278,
      "city": "London",
      "country": "GB"
    }
  }'
```

**Expected Response:**

```json
{
  "rule_results": [
    {...},
    {...},
    {
      "rule_name": "LocationMismatchRule",
      "triggered": true,
      "risk_score": 90,
      "reason": "IMPOSSIBLE TRAVEL: 3459 miles in 0.5 hours requires 6918 mph (max reasonable: 600 mph)",
      "metadata": {
        "previous_transaction_id": "tx_003",
        "distance_miles": 3459.42,
        "time_diff_hours": 0.5,
        "required_speed_mph": 6918.84,
        "max_reasonable_speed_mph": 600.0
      },
      "evaluated_at": "2024-01-15T10:30:00Z"
    }
  ],
  "risk_score": 90,
  "risk_level": "HIGH",
  "triggered_rules": ["LocationMismatchRule"],
  "explanation": "Transaction flagged as HIGH risk (score: 90/100).\nTriggered 1 rule(s): LocationMismatchRule.\n\nDetails:\n- LocationMismatchRule: IMPOSSIBLE TRAVEL: 3459 miles in 0.5 hours requires 6918 mph",
  "evaluation_time": "2024-01-15T10:30:00.123456",
  "transaction_id": "tx_004",
  "user_id": "user_002"
}
```

### Test 4: High Velocity (Multiple Transactions)

Send 6 transactions rapidly (within 1 minute):

```bash
#!/bin/bash
# Save as test_velocity.sh

for i in {1..6}; do
  echo "Sending transaction $i..."
  curl -X POST "http://localhost:8000/fraud/check" \
    -H "Content-Type: application/json" \
    -d "{
      \"transaction_id\": \"tx_vel_$i\",
      \"amount\": 50.00,
      \"currency\": \"USD\",
      \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"merchant_info\": {
        \"merchant_id\": \"merch_$i\",
        \"merchant_name\": \"Shop $i\",
        \"merchant_category\": \"retail\"
      },
      \"user_info\": {
        \"user_id\": \"user_003\",
        \"email\": \"velocity@example.com\",
        \"account_age_days\": 50
      },
      \"device\": {
        \"device_id\": \"dev_003\",
        \"device_type\": \"mobile\",
        \"os\": \"iOS\",
        \"ip_address\": \"192.168.1.3\"
      },
      \"location\": {
        \"latitude\": 40.7128,
        \"longitude\": -74.0060,
        \"city\": \"New York\",
        \"country\": \"US\"
      }
    }" | jq '.risk_score, .risk_level, .triggered_rules'

  sleep 2  # Wait 2 seconds between transactions
done
```

**Expected Output:**

```
Transaction 1: risk_score=0, risk_level=LOW, triggered_rules=[]
Transaction 2: risk_score=0, risk_level=LOW, triggered_rules=[]
Transaction 3: risk_score=0, risk_level=LOW, triggered_rules=[]
Transaction 4: risk_score=0, risk_level=LOW, triggered_rules=[]
Transaction 5: risk_score=0, risk_level=LOW, triggered_rules=[]
Transaction 6: risk_score=40, risk_level=MEDIUM, triggered_rules=["VelocityRule"]
```

---

## Rule Configuration & Tuning

### Understanding Thresholds

**Finding the right thresholds is crucial.** Too strict = many false positives (legitimate transactions blocked). Too lenient = fraud slips through.

### Tuning Process

**1. Start with Industry Defaults:**

| Rule | Parameter | Default | Rationale |
|------|-----------|---------|-----------|
| Velocity | time_window | 10 min | Fraud often happens quickly |
| Velocity | threshold | 5 txns | Normal users rarely make 5+ purchases in 10 min |
| Amount | absolute_threshold | $10,000 | Large purchases are uncommon |
| Amount | relative_multiplier | 10x | 10x normal spending is suspicious |
| Location | max_speed | 600 mph | Jet plane speed |
| Location | suspicious_speed | 300 mph | Fast car/train |

**2. Collect Data:**

Run rules in "shadow mode" (log results but don't block):

```python
# In production
result = await engine.evaluate(transaction, db)
log_to_analytics(result)  # Log for analysis

# Don't block transaction yet, just observe
```

**3. Analyze False Positives/Negatives:**

```sql
-- False Positives: Flagged but legitimate
SELECT COUNT(*)
FROM fraud_checks
WHERE risk_level = 'HIGH'
  AND actual_fraud = FALSE;

-- False Negatives: Missed fraud
SELECT COUNT(*)
FROM fraud_checks
WHERE risk_level = 'LOW'
  AND actual_fraud = TRUE;
```

**4. Adjust Thresholds:**

Example: If 30% of high-risk flags are false positives:
- Increase amount threshold: $10,000 → $15,000
- Increase velocity threshold: 5 → 7 transactions
- Decrease velocity window: 10 min → 5 min

**5. A/B Test:**

Run two configurations and compare:
- Config A: Conservative (current thresholds)
- Config B: Adjusted (new thresholds)

Measure:
- False positive rate
- False negative rate
- Revenue impact (blocked legitimate sales)

### Configuration File Approach

**Better practice:** Store thresholds in configuration:

```python
# app/config.py

class RulesConfig(BaseSettings):
    # Velocity Rule
    VELOCITY_TIME_WINDOW: int = 10  # minutes
    VELOCITY_THRESHOLD: int = 5
    VELOCITY_HIGH_RISK_THRESHOLD: int = 10

    # Amount Rule
    AMOUNT_ABSOLUTE_THRESHOLD: float = 10000.0
    AMOUNT_RELATIVE_MULTIPLIER: float = 10.0

    # Location Rule
    LOCATION_MAX_SPEED_MPH: float = 600.0
    LOCATION_SUSPICIOUS_SPEED_MPH: float = 300.0
    LOCATION_MIN_DISTANCE_MILES: float = 50.0

    class Config:
        env_file = ".env"

# Usage
config = RulesConfig()

engine = RulesEngine()
engine.add_rule(VelocityRule(
    time_window_minutes=config.VELOCITY_TIME_WINDOW,
    threshold=config.VELOCITY_THRESHOLD,
))
```

**Benefits:**
- Change thresholds without code changes
- Different configs for dev/staging/prod
- Easy A/B testing

---

## Troubleshooting

### Issue 1: Rules Not Triggering

**Symptom:** All transactions return risk_score=0

**Possible Causes:**

1. **Database Empty:**
   ```python
   # Rules need historical data to compare
   # Solution: Ensure transactions are being saved to database
   ```

2. **Rules Disabled:**
   ```python
   # Check rule status
   engine = RulesEngine()
   print(engine.get_rule_status())
   ```

3. **Thresholds Too High:**
   ```python
   # Lower thresholds for testing
   VelocityRule(threshold=2)  # Instead of 5
   ```

### Issue 2: Import Errors

**Symptom:** `ModuleNotFoundError: No module named 'app.rules'`

**Solution:**

1. Check directory structure:
   ```bash
   ls -la app/rules/
   # Should see: __init__.py, base.py, engine.py, etc.
   ```

2. Ensure `__init__.py` exists:
   ```bash
   touch app/rules/__init__.py
   ```

3. Check PYTHONPATH:
   ```bash
   export PYTHONPATH="${PYTHONPATH}:/path/to/sentinel"
   ```

### Issue 3: Database Query Errors

**Symptom:** `AttributeError: 'FraudTransaction' has no attribute 'latitude'`

**Solution:**

Our location rule expects location data in the `metadata` JSONB field:

```python
# In location_rule.py, we access:
prev_lat = previous_transaction.metadata.get("latitude")

# Make sure to save location in metadata when creating transactions
transaction = FraudTransaction(
    transaction_id="tx_001",
    user_id="user_001",
    amount=100.0,
    metadata={
        "latitude": 40.7128,
        "longitude": -74.0060,
        "city": "New York"
    }
)
```

### Issue 4: Haversine Formula Returns Incorrect Distance

**Symptom:** Distance calculation seems wrong

**Debug:**

```python
# Test with known coordinates
from app.rules.location_rule import LocationMismatchRule

rule = LocationMismatchRule()

# New York to Los Angeles
ny_lat, ny_lon = 40.7128, -74.0060
la_lat, la_lon = 34.0522, -118.2437

distance = rule._calculate_distance(ny_lat, ny_lon, la_lat, la_lon)
print(f"NY to LA: {distance:.0f} miles")
# Should be approximately 2,451 miles

# New York to London
london_lat, london_lon = 51.5074, -0.1278
distance = rule._calculate_distance(ny_lat, ny_lon, london_lat, london_lon)
print(f"NY to London: {distance:.0f} miles")
# Should be approximately 3,459 miles
```

### Issue 5: All Transactions Marked HIGH RISK

**Symptom:** Every transaction gets risk_score=80-90

**Possible Causes:**

1. **Test Data Unrealistic:**
   - Using same timestamp for all transactions
   - Using impossible locations
   - Solution: Use realistic test data

2. **Rules Too Strict:**
   - Thresholds too low
   - Solution: Adjust thresholds

3. **Bug in Aggregation:**
   ```python
   # Check in engine.py _aggregate_results()
   print("Individual scores:", [r.risk_score for r in results])
   print("Max score:", max([r.risk_score for r in results]))
   ```

---

## Next Steps

Congratulations! You've built a functional fraud detection rules engine with three production-ready rules.

### What You've Accomplished

✅ **Built a Rules Engine Framework**
- Flexible, extensible architecture using Strategy Pattern
- Easy to add new rules (just inherit from BaseRule)
- Rules are independent and testable

✅ **Implemented 3 Fraud Detection Rules**
- **VelocityRule:** Catches rapid-fire transactions
- **AmountThresholdRule:** Flags unusually large amounts
- **LocationMismatchRule:** Detects impossible travel

✅ **Learned Rule-Based Fraud Detection**
- How rules complement machine learning
- When to use rules vs ML
- How to tune thresholds

✅ **Tested with Real Scenarios**
- Normal transactions (pass)
- High amounts (trigger)
- Impossible travel (trigger)
- High velocity (trigger)

### Day 5 Preview: Advanced Rules & Historical Analysis

Tomorrow, we'll expand our fraud detection capabilities:

**New Rules:**
1. **Device Fingerprint Rule** - Detect new/suspicious devices
2. **Behavioral Pattern Rule** - Compare to user's normal behavior
3. **Time-of-Day Rule** - Flag unusual transaction times
4. **Country Mismatch Rule** - Detect transactions from risky countries

**Enhancements:**
1. **Historical Analysis** - Analyze user's transaction patterns
2. **Risk Profiles** - Build user risk profiles over time
3. **Rule Weights** - Weighted scoring instead of max score
4. **Rule Combinations** - Detect specific rule combinations

**New Concepts:**
- Feature engineering for rules
- Temporal pattern detection
- Behavioral baselining
- Rule optimization techniques

### Recommended Practice

Before moving to Day 5:

1. **Experiment with Thresholds:**
   - Lower velocity threshold to 3 transactions
   - Change amount threshold to $5,000
   - Observe how results change

2. **Add Custom Rule:**
   ```python
   class MyCustomRule(BaseRule):
       async def evaluate(self, transaction, db):
           # Your logic here
           pass
   ```

3. **Test Edge Cases:**
   - Zero amount transactions
   - Missing location data
   - First-time users
   - International transactions

4. **Read the Code:**
   - Study each rule implementation
   - Understand the Haversine formula
   - Review the engine's aggregation logic

### Additional Resources

**Books:**
- "Web Application Security" by Andrew Hoffman (Chapter on fraud detection)
- "Machine Learning for Fraud Detection" by Eric Siegel

**Articles:**
- Stripe's fraud detection blog posts
- PayPal's risk engineering articles
- AWS fraud detection best practices

**Tools:**
- **jq** - JSON processor for testing curl responses
- **Postman** - API testing tool
- **pgAdmin** - PostgreSQL GUI for inspecting data

### Getting Help

If you're stuck:

1. **Check the troubleshooting section** above
2. **Review error messages carefully** - they usually point to the issue
3. **Add debug print statements** to understand flow
4. **Test rules individually** before using in engine
5. **Verify database has data** for rules to query

---

**Navigation:** [← Previous: Day 3](README-DAY-003.md) | [Main Guide](README.md) | [Next: Day 5 →](README-DAY-005.md)

---

**Built with ❤️ for the Sentinel Fraud Detection Platform**

*Remember: Fraud detection is an ongoing battle. Rules are your first line of defense, but they must evolve as fraudsters adapt. Always monitor, analyze, and improve your rules based on real-world data.*
