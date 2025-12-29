# Day 5: Fraud Detection API Endpoint - Integration

**Building Your First End-to-End Fraud Detection System**

**Navigation:** [← Previous: Day 4](README-DAY-004.md) | [Main Guide](README.md) | [Next: Day 6 →](README-DAY-006.md)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Review](#architecture-review)
3. [Service Layer Pattern](#service-layer-pattern)
4. [Rules Engine Quick Review](#rules-engine-quick-review)
5. [Complete Code Implementation](#complete-code-implementation)
6. [API Endpoints Explained](#api-endpoints-explained)
7. [Testing Your Fraud Detection API](#testing-your-fraud-detection-api)
8. [Understanding the Request Flow](#understanding-the-request-flow)
9. [Error Handling Deep Dive](#error-handling-deep-dive)
10. [Troubleshooting](#troubleshooting)
11. [Next Steps](#next-steps)

---

## Overview

### Day 5 Objectives

Today is the big day! We're bringing together everything we've built in Days 1-4 to create a **fully functional fraud detection API**. By the end of this day, you'll have a working system that can:

✅ Accept transaction data via REST API
✅ Validate requests using Pydantic schemas
✅ Run fraud detection rules from your rules engine
✅ Store results in PostgreSQL database
✅ Return fraud scores and risk assessments
✅ Retrieve fraud check history

**What You'll Build:**

```
POST /api/v1/check-fraud
├─→ Validate request (Pydantic from Day 3)
├─→ Run fraud rules (Rules Engine from Day 4)
├─→ Calculate fraud score
├─→ Save to database (SQLAlchemy from Day 2)
└─→ Return fraud assessment

GET /api/v1/transactions/{id}
└─→ Retrieve fraud check result

GET /api/v1/transactions
└─→ List recent fraud checks (paginated)
```

**Estimated Time:** 3-4 hours

**Prerequisites:**
- ✅ Completed Days 1-4
- ✅ PostgreSQL running and configured
- ✅ Virtual environment activated
- ✅ All dependencies installed

---

## Architecture Review

### What We've Built So Far

Let's review what we have from Days 1-4:

#### Day 1: FastAPI Foundation
```python
# Basic FastAPI app with health check
GET /health → { "status": "healthy" }
GET /info → { "name": "Sentinel", "version": "0.1.0" }
```

#### Day 2: Database Layer
```python
# PostgreSQL with SQLAlchemy models
FraudTransaction      # Stores fraud check results
UserRiskProfile       # Tracks user risk history
DeviceFingerprint     # Tracks device patterns
```

#### Day 3: Pydantic Schemas
```python
# Request/Response validation
TransactionCheckRequest   # Input validation
FraudCheckResponse       # Output schema
DeviceInfo, UserInfo     # Nested schemas
```

#### Day 4: Rules Engine
```python
# Fraud detection rules
RuleEngine               # Evaluates all rules
Rule 1: Large Amount     # Flag transactions > threshold
Rule 2: New Device       # Flag new device patterns
Rule 3: High Frequency   # Flag rapid transactions
```

### Today's Integration

```
┌─────────────────────────────────────────────────────┐
│                   CLIENT REQUEST                     │
│         POST /api/v1/check-fraud                     │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│              API LAYER (fraud.py)                    │
│  - Route definition                                  │
│  - Request validation (Pydantic)                     │
│  - Response formatting                               │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│         SERVICE LAYER (fraud_service.py)             │
│  - Business logic                                    │
│  - Rules engine integration                          │
│  - Score calculation                                 │
│  - Database operations                               │
└────────────────┬────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
┌──────────────┐  ┌──────────────┐
│ RULES ENGINE │  │  DATABASE    │
│  (Day 4)     │  │  (Day 2)     │
└──────────────┘  └──────────────┘
```

---

## Service Layer Pattern

### What is a Service Layer?

The **Service Layer Pattern** separates business logic from API routing logic. This makes code:
- **Testable** - Test business logic without HTTP
- **Reusable** - Use same logic in CLI, background jobs, etc.
- **Maintainable** - Clear separation of concerns

### Without Service Layer (❌ Bad)

```python
# fraud.py - Everything in route handler
@app.post("/api/v1/check-fraud")
async def check_fraud(request: TransactionCheckRequest, db: Session):
    # Business logic mixed with routing
    rules_engine = RulesEngine()
    results = rules_engine.evaluate(request)
    score = sum(r.score for r in results)
    risk = "HIGH" if score > 70 else "MEDIUM" if score > 40 else "LOW"

    # Database logic in route
    fraud_tx = FraudTransaction(
        transaction_id=request.transaction_id,
        amount=request.amount,
        fraud_score=score,
        risk_level=risk
    )
    db.add(fraud_tx)
    db.commit()

    return {"score": score, "risk": risk}

# ❌ Problems:
# - Can't test without HTTP server
# - Can't reuse logic elsewhere
# - Hard to mock database
# - Violates Single Responsibility Principle
```

### With Service Layer (✅ Good)

```python
# fraud_service.py - Business logic
class FraudService:
    def check_fraud(self, request: TransactionCheckRequest, db: Session):
        # Reusable business logic
        results = self.rules_engine.evaluate(request)
        score = self.calculate_score(results)
        risk = self.determine_risk(score)
        return self.save_fraud_check(request, score, risk, db)

# fraud.py - Route handler
@app.post("/api/v1/check-fraud")
async def check_fraud(request: TransactionCheckRequest, db: Session = Depends(get_db)):
    service = FraudService()
    return service.check_fraud(request, db)

# ✅ Benefits:
# - Testable: service.check_fraud() without HTTP
# - Reusable: Use in CLI, Celery tasks, etc.
# - Mockable: Easy to mock database
# - Clean: Single Responsibility Principle
```

### Dependency Injection in FastAPI

**Dependency Injection** is a pattern where dependencies (like database connections) are "injected" into functions rather than created inside them.

**Without DI:**
```python
@app.get("/users")
async def get_users():
    # ❌ Creates connection inside function
    db = create_db_connection()
    users = db.query(User).all()
    db.close()
    return users
```

**With DI:**
```python
# dependencies.py
def get_db():
    db = SessionLocal()
    try:
        yield db  # Inject into function
    finally:
        db.close()  # Always cleanup

# fraud.py
@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    # ✅ Database injected, auto-cleanup
    users = db.query(User).all()
    return users
```

**Benefits:**
- **Automatic cleanup** - `finally` block always runs
- **Testability** - Easy to inject mock database
- **Reusability** - Same `get_db()` for all routes
- **Connection pooling** - Efficient resource management

---

## Rules Engine Quick Review

Before we integrate, let's understand what the rules engine from Day 4 does:

### Rules Engine Structure

```python
# From Day 4: app/rules/engine.py
class RuleResult:
    """Result from a single rule evaluation."""
    rule_name: str
    triggered: bool
    score: int
    reason: str

class RulesEngine:
    """Evaluates all fraud detection rules."""

    def evaluate(self, request: TransactionCheckRequest) -> List[RuleResult]:
        """Run all rules and return results."""
        results = []

        # Rule 1: Large Amount Detection
        if request.amount > 100000:
            results.append(RuleResult(
                rule_name="large_amount",
                triggered=True,
                score=30,
                reason=f"Transaction amount ₦{request.amount:,.2f} exceeds threshold"
            ))

        # Rule 2: New Device Detection
        if request.device_info.is_new_device:
            results.append(RuleResult(
                rule_name="new_device",
                triggered=True,
                score=20,
                reason="Transaction from new/unknown device"
            ))

        # Rule 3: High Frequency Detection
        # (Would check Redis for recent transactions)

        return results
```

### How Rules Work

1. **Input:** Transaction data (amount, device, location, etc.)
2. **Processing:** Each rule checks specific fraud indicators
3. **Output:** List of triggered rules with scores

```
Request: {
  "amount": 150000,
  "device_info": { "is_new_device": true }
}
    ↓
RulesEngine.evaluate()
    ↓
[
  RuleResult(rule_name="large_amount", score=30, triggered=True),
  RuleResult(rule_name="new_device", score=20, triggered=True)
]
    ↓
Total Score: 50 → Risk: MEDIUM
```

---

## Complete Code Implementation

### Step 1: Project Structure

First, let's set up the new directories and files:

```bash
# Navigate to project root
cd /home/user/sentinel-api

# Create API module
mkdir -p app/api
touch app/api/__init__.py
touch app/api/fraud.py
touch app/api/dependencies.py

# Create services module
mkdir -p app/services
touch app/services/__init__.py
touch app/services/fraud_service.py

# Verify structure
tree app/
```

**Expected output:**
```
app/
├── __init__.py
├── api/
│   ├── __init__.py
│   ├── dependencies.py
│   └── fraud.py
├── core/
│   ├── __init__.py
│   └── config.py
├── db/
│   ├── __init__.py
│   ├── base.py
│   └── session.py
├── main.py
├── models/
│   ├── __init__.py
│   └── fraud.py
├── rules/
│   ├── __init__.py
│   └── engine.py
├── schemas/
│   ├── __init__.py
│   └── fraud.py
└── services/
    ├── __init__.py
    └── fraud_service.py
```

---

### Step 2: Dependencies Module

**File: `/home/user/sentinel-api/app/api/dependencies.py`**

This file contains shared dependencies that will be injected into route handlers.

```python
"""
API Dependencies for Dependency Injection.

This module provides common dependencies used across API routes:
- Database session management
- Authentication (future)
- Rate limiting (future)
- Logging (future)

FastAPI's Depends() system automatically calls these functions
and injects the results into route handlers.
"""

from typing import Generator
from sqlalchemy.orm import Session
from app.db.session import SessionLocal


def get_db() -> Generator[Session, None, None]:
    """
    Database session dependency.

    Yields a SQLAlchemy session and ensures proper cleanup.

    Usage in route:
        @app.get("/transactions")
        async def get_transactions(db: Session = Depends(get_db)):
            return db.query(FraudTransaction).all()

    Benefits:
    - Automatic session cleanup (even if route raises exception)
    - Connection pooling (reuses connections efficiently)
    - Testability (easy to override with mock database)

    Yields:
        Session: SQLAlchemy database session

    Example:
        The 'try/finally' ensures cleanup:
        1. Request arrives → SessionLocal() creates connection
        2. Route executes → Uses db for queries
        3. Response sent → finally: db.close() always runs
        4. Connection returns to pool → Ready for next request
    """
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


# Future dependencies can be added here:

# async def get_current_user(token: str = Depends(oauth2_scheme)):
#     """Verify JWT token and return current user."""
#     pass

# async def check_rate_limit(request: Request):
#     """Check if client has exceeded rate limit."""
#     pass

# async def get_redis() -> Generator:
#     """Redis connection for caching."""
#     pass
```

**Key Concepts:**

1. **Generator Function**: Uses `yield` instead of `return`
   - Code before `yield` runs before route
   - Code after `yield` runs after route (cleanup)
   - `finally` ensures cleanup even if exception occurs

2. **Type Hints**: `Generator[Session, None, None]`
   - First type: What we yield (Session)
   - Second type: What we send (None)
   - Third type: What we return (None)

---

### Step 3: Fraud Service Layer

**File: `/home/user/sentinel-api/app/services/fraud_service.py`**

This is the heart of our fraud detection logic.

```python
"""
Fraud Detection Service Layer.

This service encapsulates all business logic for fraud detection:
- Transaction fraud checking
- Rules engine integration
- Fraud score calculation
- Database persistence
- Fraud history retrieval

Separating business logic from API routes provides:
- Testability: Can test without HTTP server
- Reusability: Use in CLI tools, background jobs, etc.
- Maintainability: Single Responsibility Principle
"""

from typing import List, Optional, Dict, Any
from datetime import datetime, timedelta
from sqlalchemy.orm import Session
from sqlalchemy import desc

from app.schemas.fraud import (
    TransactionCheckRequest,
    FraudCheckResponse,
    FraudFlag,
    RiskLevel
)
from app.models.fraud import FraudTransaction, UserRiskProfile, DeviceFingerprint
from app.rules.engine import RulesEngine, RuleResult


class FraudService:
    """
    Service for fraud detection operations.

    This class handles:
    - Running fraud detection rules
    - Calculating fraud scores
    - Determining risk levels
    - Persisting fraud checks to database
    - Retrieving fraud check history
    """

    def __init__(self):
        """Initialize the fraud service with rules engine."""
        self.rules_engine = RulesEngine()

    def check_fraud(
        self,
        request: TransactionCheckRequest,
        db: Session
    ) -> FraudCheckResponse:
        """
        Perform complete fraud check on a transaction.

        This is the main entry point for fraud detection. It:
        1. Runs all fraud detection rules
        2. Calculates total fraud score
        3. Determines risk level
        4. Updates user risk profile
        5. Saves fraud check to database
        6. Returns detailed fraud assessment

        Args:
            request: Transaction data to check
            db: Database session

        Returns:
            FraudCheckResponse with fraud score, risk level, and triggered rules

        Example:
            >>> service = FraudService()
            >>> request = TransactionCheckRequest(
            ...     transaction_id="tx_123",
            ...     amount=150000,
            ...     ...
            ... )
            >>> response = service.check_fraud(request, db)
            >>> print(response.fraud_score)  # 50
            >>> print(response.risk_level)   # RiskLevel.MEDIUM
        """
        # Step 1: Run fraud detection rules
        rule_results = self.rules_engine.evaluate(request)

        # Step 2: Calculate total fraud score
        fraud_score = self._calculate_fraud_score(rule_results)

        # Step 3: Determine risk level based on score
        risk_level = self._determine_risk_level(fraud_score)

        # Step 4: Build list of triggered fraud flags
        fraud_flags = self._build_fraud_flags(rule_results)

        # Step 5: Update user risk profile
        self._update_user_risk_profile(
            user_id=request.user_id,
            fraud_score=fraud_score,
            db=db
        )

        # Step 6: Update device fingerprint
        if request.device_info:
            self._update_device_fingerprint(
                device_id=request.device_info.device_id,
                user_id=request.user_id,
                db=db
            )

        # Step 7: Save fraud check to database
        fraud_transaction = self._save_fraud_check(
            request=request,
            fraud_score=fraud_score,
            risk_level=risk_level,
            rule_results=rule_results,
            db=db
        )

        # Step 8: Build and return response
        return FraudCheckResponse(
            transaction_id=request.transaction_id,
            fraud_score=fraud_score,
            risk_level=risk_level,
            is_fraudulent=risk_level == RiskLevel.HIGH,
            fraud_flags=fraud_flags,
            timestamp=fraud_transaction.created_at,
            recommendation=self._get_recommendation(risk_level)
        )

    def get_transaction(
        self,
        transaction_id: str,
        db: Session
    ) -> Optional[FraudCheckResponse]:
        """
        Retrieve a fraud check result by transaction ID.

        Args:
            transaction_id: Transaction identifier
            db: Database session

        Returns:
            FraudCheckResponse if found, None otherwise
        """
        fraud_tx = db.query(FraudTransaction).filter(
            FraudTransaction.transaction_id == transaction_id
        ).first()

        if not fraud_tx:
            return None

        return self._fraud_transaction_to_response(fraud_tx)

    def get_recent_transactions(
        self,
        db: Session,
        limit: int = 50,
        offset: int = 0
    ) -> List[FraudCheckResponse]:
        """
        Retrieve recent fraud checks with pagination.

        Args:
            db: Database session
            limit: Maximum number of results (default: 50)
            offset: Number of results to skip (default: 0)

        Returns:
            List of FraudCheckResponse objects

        Example:
            # Get first page (transactions 0-49)
            >>> page1 = service.get_recent_transactions(db, limit=50, offset=0)

            # Get second page (transactions 50-99)
            >>> page2 = service.get_recent_transactions(db, limit=50, offset=50)
        """
        fraud_txs = db.query(FraudTransaction).order_by(
            desc(FraudTransaction.created_at)
        ).limit(limit).offset(offset).all()

        return [
            self._fraud_transaction_to_response(tx)
            for tx in fraud_txs
        ]

    def get_user_fraud_history(
        self,
        user_id: str,
        db: Session,
        days: int = 30
    ) -> Dict[str, Any]:
        """
        Get fraud history for a specific user.

        Args:
            user_id: User identifier
            db: Database session
            days: Number of days to look back (default: 30)

        Returns:
            Dictionary with user fraud statistics
        """
        cutoff_date = datetime.utcnow() - timedelta(days=days)

        # Query user's transactions
        transactions = db.query(FraudTransaction).filter(
            FraudTransaction.user_id == user_id,
            FraudTransaction.created_at >= cutoff_date
        ).all()

        # Calculate statistics
        total_transactions = len(transactions)
        flagged_transactions = sum(1 for tx in transactions if tx.risk_level in ["MEDIUM", "HIGH"])
        total_amount = sum(tx.amount for tx in transactions)
        avg_fraud_score = sum(tx.fraud_score for tx in transactions) / total_transactions if total_transactions > 0 else 0

        # Get user risk profile
        risk_profile = db.query(UserRiskProfile).filter(
            UserRiskProfile.user_id == user_id
        ).first()

        return {
            "user_id": user_id,
            "period_days": days,
            "total_transactions": total_transactions,
            "flagged_transactions": flagged_transactions,
            "total_amount": total_amount,
            "average_fraud_score": round(avg_fraud_score, 2),
            "current_risk_score": risk_profile.risk_score if risk_profile else 0,
            "account_status": risk_profile.status if risk_profile else "UNKNOWN"
        }

    # ============================================================
    # PRIVATE HELPER METHODS
    # ============================================================

    def _calculate_fraud_score(self, rule_results: List[RuleResult]) -> int:
        """
        Calculate total fraud score from rule results.

        The fraud score is the sum of all triggered rule scores.
        Score range: 0-100

        Args:
            rule_results: List of rule evaluation results

        Returns:
            Total fraud score (0-100)
        """
        total_score = sum(
            result.score for result in rule_results
            if result.triggered
        )
        # Cap at 100
        return min(total_score, 100)

    def _determine_risk_level(self, fraud_score: int) -> RiskLevel:
        """
        Determine risk level based on fraud score.

        Risk level thresholds:
        - LOW: 0-40
        - MEDIUM: 41-70
        - HIGH: 71-100

        Args:
            fraud_score: Total fraud score

        Returns:
            RiskLevel enum value
        """
        if fraud_score >= 71:
            return RiskLevel.HIGH
        elif fraud_score >= 41:
            return RiskLevel.MEDIUM
        else:
            return RiskLevel.LOW

    def _build_fraud_flags(self, rule_results: List[RuleResult]) -> List[FraudFlag]:
        """
        Convert rule results to fraud flags for response.

        Args:
            rule_results: List of rule evaluation results

        Returns:
            List of FraudFlag objects for triggered rules
        """
        return [
            FraudFlag(
                flag_type=result.rule_name,
                severity=self._score_to_severity(result.score),
                description=result.reason
            )
            for result in rule_results
            if result.triggered
        ]

    def _score_to_severity(self, score: int) -> str:
        """Convert rule score to severity level."""
        if score >= 30:
            return "HIGH"
        elif score >= 15:
            return "MEDIUM"
        else:
            return "LOW"

    def _get_recommendation(self, risk_level: RiskLevel) -> str:
        """
        Get recommendation based on risk level.

        Args:
            risk_level: Determined risk level

        Returns:
            Action recommendation string
        """
        recommendations = {
            RiskLevel.LOW: "APPROVE - Transaction appears legitimate",
            RiskLevel.MEDIUM: "REVIEW - Manual review recommended",
            RiskLevel.HIGH: "BLOCK - High fraud risk detected"
        }
        return recommendations.get(risk_level, "REVIEW")

    def _save_fraud_check(
        self,
        request: TransactionCheckRequest,
        fraud_score: int,
        risk_level: RiskLevel,
        rule_results: List[RuleResult],
        db: Session
    ) -> FraudTransaction:
        """
        Save fraud check to database.

        Args:
            request: Original transaction request
            fraud_score: Calculated fraud score
            risk_level: Determined risk level
            rule_results: All rule evaluation results
            db: Database session

        Returns:
            Created FraudTransaction object
        """
        # Build metadata from triggered rules
        triggered_rules = [
            {
                "rule": result.rule_name,
                "score": result.score,
                "reason": result.reason
            }
            for result in rule_results
            if result.triggered
        ]

        fraud_tx = FraudTransaction(
            transaction_id=request.transaction_id,
            user_id=request.user_id,
            amount=request.amount,
            currency=request.currency,
            transaction_type=request.transaction_type,
            fraud_score=fraud_score,
            risk_level=risk_level.value,
            is_flagged=risk_level in [RiskLevel.MEDIUM, RiskLevel.HIGH],
            metadata={
                "triggered_rules": triggered_rules,
                "device_id": request.device_info.device_id if request.device_info else None,
                "ip_address": request.device_info.ip_address if request.device_info else None,
                "location": {
                    "country": request.location_info.country if request.location_info else None,
                    "city": request.location_info.city if request.location_info else None
                }
            }
        )

        db.add(fraud_tx)
        db.commit()
        db.refresh(fraud_tx)

        return fraud_tx

    def _update_user_risk_profile(
        self,
        user_id: str,
        fraud_score: int,
        db: Session
    ) -> None:
        """
        Update or create user risk profile.

        Args:
            user_id: User identifier
            fraud_score: Latest fraud score
            db: Database session
        """
        risk_profile = db.query(UserRiskProfile).filter(
            UserRiskProfile.user_id == user_id
        ).first()

        if risk_profile:
            # Update existing profile
            risk_profile.transaction_count += 1
            risk_profile.total_fraud_score += fraud_score
            risk_profile.risk_score = risk_profile.total_fraud_score / risk_profile.transaction_count
            risk_profile.last_transaction_at = datetime.utcnow()

            # Update status based on average risk score
            if risk_profile.risk_score >= 70:
                risk_profile.status = "BLOCKED"
            elif risk_profile.risk_score >= 40:
                risk_profile.status = "FLAGGED"
            else:
                risk_profile.status = "ACTIVE"
        else:
            # Create new profile
            risk_profile = UserRiskProfile(
                user_id=user_id,
                risk_score=fraud_score,
                transaction_count=1,
                total_fraud_score=fraud_score,
                status="ACTIVE" if fraud_score < 40 else "FLAGGED",
                last_transaction_at=datetime.utcnow()
            )
            db.add(risk_profile)

        db.commit()

    def _update_device_fingerprint(
        self,
        device_id: str,
        user_id: str,
        db: Session
    ) -> None:
        """
        Update or create device fingerprint.

        Args:
            device_id: Device identifier
            user_id: User identifier
            db: Database session
        """
        device = db.query(DeviceFingerprint).filter(
            DeviceFingerprint.device_id == device_id
        ).first()

        if device:
            # Update existing device
            device.seen_count += 1
            device.last_seen_at = datetime.utcnow()
        else:
            # Create new device
            device = DeviceFingerprint(
                device_id=device_id,
                user_id=user_id,
                seen_count=1,
                first_seen_at=datetime.utcnow(),
                last_seen_at=datetime.utcnow()
            )
            db.add(device)

        db.commit()

    def _fraud_transaction_to_response(
        self,
        fraud_tx: FraudTransaction
    ) -> FraudCheckResponse:
        """
        Convert database model to response schema.

        Args:
            fraud_tx: FraudTransaction database model

        Returns:
            FraudCheckResponse schema
        """
        # Extract fraud flags from metadata
        fraud_flags = []
        if fraud_tx.metadata and "triggered_rules" in fraud_tx.metadata:
            for rule in fraud_tx.metadata["triggered_rules"]:
                fraud_flags.append(FraudFlag(
                    flag_type=rule["rule"],
                    severity=self._score_to_severity(rule["score"]),
                    description=rule["reason"]
                ))

        return FraudCheckResponse(
            transaction_id=fraud_tx.transaction_id,
            fraud_score=fraud_tx.fraud_score,
            risk_level=RiskLevel(fraud_tx.risk_level),
            is_fraudulent=fraud_tx.is_flagged,
            fraud_flags=fraud_flags,
            timestamp=fraud_tx.created_at,
            recommendation=self._get_recommendation(RiskLevel(fraud_tx.risk_level))
        )
```

---

### Step 4: API Routes

**File: `/home/user/sentinel-api/app/api/fraud.py`**

This file defines our API endpoints.

```python
"""
Fraud Detection API Routes.

This module defines all fraud detection API endpoints:
- POST /api/v1/check-fraud - Check transaction for fraud
- GET /api/v1/transactions/{id} - Get fraud check result
- GET /api/v1/transactions - List recent fraud checks
- GET /api/v1/users/{user_id}/fraud-history - Get user fraud history

All routes use:
- Dependency injection for database sessions
- Pydantic schemas for request/response validation
- Service layer for business logic
- Proper HTTP status codes and error handling
"""

from typing import List, Optional
from fastapi import APIRouter, Depends, HTTPException, Query, Path
from sqlalchemy.orm import Session

from app.api.dependencies import get_db
from app.schemas.fraud import TransactionCheckRequest, FraudCheckResponse
from app.services.fraud_service import FraudService


# Create API router with prefix and tags
router = APIRouter(
    prefix="/api/v1",
    tags=["Fraud Detection"],
    responses={
        500: {"description": "Internal server error"},
        422: {"description": "Validation error"}
    }
)


@router.post(
    "/check-fraud",
    response_model=FraudCheckResponse,
    status_code=200,
    summary="Check Transaction for Fraud",
    description="""
    Perform comprehensive fraud detection on a transaction.

    This endpoint:
    - Validates transaction data
    - Runs fraud detection rules
    - Calculates fraud score (0-100)
    - Determines risk level (LOW/MEDIUM/HIGH)
    - Saves check to database
    - Returns detailed fraud assessment

    **Risk Levels:**
    - LOW (0-40): Transaction appears legitimate
    - MEDIUM (41-70): Requires manual review
    - HIGH (71-100): High fraud risk, block recommended
    """,
    response_description="Fraud check results with score and risk level"
)
async def check_fraud(
    request: TransactionCheckRequest,
    db: Session = Depends(get_db)
) -> FraudCheckResponse:
    """
    Check a transaction for fraud indicators.

    Args:
        request: Transaction data to check
        db: Database session (injected)

    Returns:
        FraudCheckResponse with fraud score and risk assessment

    Raises:
        HTTPException: If fraud check fails

    Example:
        POST /api/v1/check-fraud
        {
          "transaction_id": "tx_12345",
          "user_id": "user_67890",
          "amount": 50000,
          "currency": "NGN",
          "transaction_type": "WITHDRAWAL",
          ...
        }

        Response:
        {
          "transaction_id": "tx_12345",
          "fraud_score": 25,
          "risk_level": "LOW",
          "is_fraudulent": false,
          "recommendation": "APPROVE - Transaction appears legitimate"
        }
    """
    try:
        service = FraudService()
        result = service.check_fraud(request, db)
        return result
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Fraud check failed: {str(e)}"
        )


@router.get(
    "/transactions/{transaction_id}",
    response_model=FraudCheckResponse,
    status_code=200,
    summary="Get Fraud Check Result",
    description="Retrieve fraud check result for a specific transaction.",
    response_description="Fraud check result"
)
async def get_transaction(
    transaction_id: str = Path(..., description="Transaction ID to retrieve"),
    db: Session = Depends(get_db)
) -> FraudCheckResponse:
    """
    Retrieve fraud check result by transaction ID.

    Args:
        transaction_id: Transaction identifier
        db: Database session (injected)

    Returns:
        FraudCheckResponse for the transaction

    Raises:
        HTTPException: 404 if transaction not found

    Example:
        GET /api/v1/transactions/tx_12345

        Response:
        {
          "transaction_id": "tx_12345",
          "fraud_score": 25,
          "risk_level": "LOW",
          ...
        }
    """
    service = FraudService()
    result = service.get_transaction(transaction_id, db)

    if not result:
        raise HTTPException(
            status_code=404,
            detail=f"Transaction {transaction_id} not found"
        )

    return result


@router.get(
    "/transactions",
    response_model=List[FraudCheckResponse],
    status_code=200,
    summary="List Recent Fraud Checks",
    description="Retrieve recent fraud checks with pagination.",
    response_description="List of fraud check results"
)
async def list_transactions(
    limit: int = Query(50, ge=1, le=100, description="Maximum number of results"),
    offset: int = Query(0, ge=0, description="Number of results to skip"),
    db: Session = Depends(get_db)
) -> List[FraudCheckResponse]:
    """
    List recent fraud checks with pagination.

    Args:
        limit: Maximum results to return (1-100, default: 50)
        offset: Number of results to skip (default: 0)
        db: Database session (injected)

    Returns:
        List of FraudCheckResponse objects

    Example:
        # Get first page
        GET /api/v1/transactions?limit=50&offset=0

        # Get second page
        GET /api/v1/transactions?limit=50&offset=50

        Response:
        [
          {
            "transaction_id": "tx_12345",
            "fraud_score": 25,
            ...
          },
          ...
        ]
    """
    service = FraudService()
    return service.get_recent_transactions(db, limit, offset)


@router.get(
    "/users/{user_id}/fraud-history",
    status_code=200,
    summary="Get User Fraud History",
    description="Retrieve fraud statistics for a specific user.",
    response_description="User fraud history and statistics"
)
async def get_user_fraud_history(
    user_id: str = Path(..., description="User ID"),
    days: int = Query(30, ge=1, le=365, description="Number of days to analyze"),
    db: Session = Depends(get_db)
):
    """
    Get fraud history and statistics for a user.

    Args:
        user_id: User identifier
        days: Number of days to analyze (1-365, default: 30)
        db: Database session (injected)

    Returns:
        Dictionary with user fraud statistics

    Example:
        GET /api/v1/users/user_67890/fraud-history?days=30

        Response:
        {
          "user_id": "user_67890",
          "period_days": 30,
          "total_transactions": 15,
          "flagged_transactions": 2,
          "total_amount": 750000,
          "average_fraud_score": 23.5,
          "current_risk_score": 25.0,
          "account_status": "ACTIVE"
        }
    """
    service = FraudService()
    return service.get_user_fraud_history(user_id, db, days)
```

---

### Step 5: Update Main Application

**File: `/home/user/sentinel-api/app/main.py`**

Update your main FastAPI application to include the new routes:

```python
"""
Sentinel Fraud Detection Platform - Main Application.

This is the entry point for the FastAPI application.
It configures:
- FastAPI app instance
- API routes
- CORS middleware
- Database initialization
- Error handlers
"""

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

from app.core.config import settings
from app.db.session import engine
from app.db.base import Base
from app.api import fraud


# Lifespan context manager for startup/shutdown events
@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Lifespan context manager for startup and shutdown events.

    Startup:
    - Create database tables
    - Initialize connections

    Shutdown:
    - Close database connections
    - Cleanup resources
    """
    # Startup: Create database tables
    print("🚀 Starting Sentinel Fraud Detection Platform...")
    print("📊 Creating database tables...")
    Base.metadata.create_all(bind=engine)
    print("✅ Database tables created successfully")

    yield  # Application runs here

    # Shutdown
    print("🛑 Shutting down Sentinel...")
    engine.dispose()
    print("✅ Cleanup complete")


# Create FastAPI application
app = FastAPI(
    title=settings.PROJECT_NAME,
    description="Production-grade fraud detection platform for African fintech",
    version=settings.VERSION,
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
    lifespan=lifespan
)


# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # In production, specify exact origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# Include API routers
app.include_router(fraud.router)


# Root endpoint
@app.get("/", tags=["Root"])
async def root():
    """
    Root endpoint - API information.

    Returns:
        Basic API information and links
    """
    return {
        "name": settings.PROJECT_NAME,
        "version": settings.VERSION,
        "status": "operational",
        "docs": "/docs",
        "health": "/health"
    }


# Health check endpoint
@app.get("/health", tags=["Health"])
async def health_check():
    """
    Health check endpoint.

    Returns:
        System health status
    """
    return {
        "status": "healthy",
        "service": "sentinel-api",
        "version": settings.VERSION
    }


# Info endpoint
@app.get("/info", tags=["Info"])
async def info():
    """
    System information endpoint.

    Returns:
        Detailed system information
    """
    return {
        "name": settings.PROJECT_NAME,
        "version": settings.VERSION,
        "environment": settings.ENVIRONMENT,
        "database": "PostgreSQL",
        "api_version": "v1"
    }


# Global exception handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """
    Global exception handler for unhandled errors.

    Args:
        request: The request that caused the error
        exc: The exception that was raised

    Returns:
        JSON error response
    """
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "message": str(exc),
            "path": str(request.url)
        }
    )


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "app.main:app",
        host="0.0.0.0",
        port=8000,
        reload=True
    )
```

---

### Step 6: Update API __init__.py

**File: `/home/user/sentinel-api/app/api/__init__.py`**

```python
"""
API module for Sentinel Fraud Detection Platform.

This module contains all API route definitions:
- fraud.py: Fraud detection endpoints
- dependencies.py: Shared dependencies for dependency injection

Future modules:
- auth.py: Authentication endpoints
- admin.py: Admin endpoints
- webhooks.py: Webhook endpoints
"""

from app.api import fraud

__all__ = ["fraud"]
```

---

### Step 7: Update Services __init__.py

**File: `/home/user/sentinel-api/app/services/__init__.py`**

```python
"""
Services module for business logic.

This module contains service layer classes that encapsulate
business logic separated from API routes:
- fraud_service.py: Fraud detection business logic

Benefits of service layer:
- Testability: Test business logic without HTTP
- Reusability: Use in CLI, background jobs, etc.
- Maintainability: Clear separation of concerns
"""

from app.services.fraud_service import FraudService

__all__ = ["FraudService"]
```

---

## API Endpoints Explained

### 1. POST /api/v1/check-fraud

**Purpose:** Check a transaction for fraud

**Request:**
```json
{
  "transaction_id": "tx_abc123",
  "user_id": "user_xyz789",
  "amount": 75000,
  "currency": "NGN",
  "transaction_type": "WITHDRAWAL",
  "user_info": {
    "email": "user@example.com",
    "phone_number": "+2348012345678",
    "account_age_days": 45
  },
  "device_info": {
    "device_id": "dev_mobile_001",
    "device_type": "MOBILE",
    "os": "Android",
    "browser": "Chrome Mobile",
    "ip_address": "197.210.55.123",
    "is_vpn": false,
    "is_proxy": false,
    "is_new_device": false
  },
  "location_info": {
    "country": "Nigeria",
    "city": "Lagos",
    "latitude": 6.5244,
    "longitude": 3.3792
  }
}
```

**Response:**
```json
{
  "transaction_id": "tx_abc123",
  "fraud_score": 35,
  "risk_level": "LOW",
  "is_fraudulent": false,
  "fraud_flags": [
    {
      "flag_type": "new_user",
      "severity": "MEDIUM",
      "description": "Account created less than 60 days ago"
    }
  ],
  "timestamp": "2024-01-15T10:30:00Z",
  "recommendation": "APPROVE - Transaction appears legitimate"
}
```

**HTTP Status Codes:**
- `200`: Successful fraud check
- `422`: Validation error (invalid request data)
- `500`: Internal server error

---

### 2. GET /api/v1/transactions/{transaction_id}

**Purpose:** Retrieve fraud check result

**Request:**
```bash
GET /api/v1/transactions/tx_abc123
```

**Response:**
```json
{
  "transaction_id": "tx_abc123",
  "fraud_score": 35,
  "risk_level": "LOW",
  "is_fraudulent": false,
  "fraud_flags": [...],
  "timestamp": "2024-01-15T10:30:00Z",
  "recommendation": "APPROVE - Transaction appears legitimate"
}
```

**HTTP Status Codes:**
- `200`: Transaction found
- `404`: Transaction not found

---

### 3. GET /api/v1/transactions

**Purpose:** List recent fraud checks with pagination

**Request:**
```bash
GET /api/v1/transactions?limit=10&offset=0
```

**Query Parameters:**
- `limit` (optional): Number of results (1-100, default: 50)
- `offset` (optional): Number to skip (default: 0)

**Response:**
```json
[
  {
    "transaction_id": "tx_abc123",
    "fraud_score": 35,
    "risk_level": "LOW",
    ...
  },
  {
    "transaction_id": "tx_def456",
    "fraud_score": 75,
    "risk_level": "HIGH",
    ...
  }
]
```

---

### 4. GET /api/v1/users/{user_id}/fraud-history

**Purpose:** Get fraud statistics for a user

**Request:**
```bash
GET /api/v1/users/user_xyz789/fraud-history?days=30
```

**Query Parameters:**
- `days` (optional): Days to analyze (1-365, default: 30)

**Response:**
```json
{
  "user_id": "user_xyz789",
  "period_days": 30,
  "total_transactions": 25,
  "flagged_transactions": 3,
  "total_amount": 1250000,
  "average_fraud_score": 28.4,
  "current_risk_score": 32.0,
  "account_status": "ACTIVE"
}
```

---

## Testing Your Fraud Detection API

### Step 1: Start the Server

```bash
# Activate virtual environment
cd /home/user/sentinel-api
source venv/bin/activate

# Start server with auto-reload
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**Expected output:**
```
🚀 Starting Sentinel Fraud Detection Platform...
📊 Creating database tables...
✅ Database tables created successfully
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process
INFO:     Started server process
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

---

### Step 2: Test with Swagger UI

Open your browser and navigate to:
```
http://localhost:8000/docs
```

You'll see the interactive API documentation with all endpoints.

**Try this:**
1. Click on `POST /api/v1/check-fraud`
2. Click "Try it out"
3. Use this test data:

```json
{
  "transaction_id": "tx_test_001",
  "user_id": "user_test_001",
  "amount": 50000,
  "currency": "NGN",
  "transaction_type": "WITHDRAWAL",
  "user_info": {
    "email": "test@example.com",
    "phone_number": "+2348012345678",
    "account_age_days": 100
  },
  "device_info": {
    "device_id": "dev_test_001",
    "device_type": "MOBILE",
    "os": "Android",
    "browser": "Chrome",
    "ip_address": "197.210.55.100",
    "is_vpn": false,
    "is_proxy": false,
    "is_new_device": false
  },
  "location_info": {
    "country": "Nigeria",
    "city": "Lagos",
    "latitude": 6.5244,
    "longitude": 3.3792
  }
}
```

4. Click "Execute"
5. Check the response - should show fraud score and risk level

---

### Step 3: Test with curl

#### Test 1: Low-Risk Transaction

```bash
curl -X POST "http://localhost:8000/api/v1/check-fraud" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_low_risk_001",
    "user_id": "user_001",
    "amount": 25000,
    "currency": "NGN",
    "transaction_type": "DEPOSIT",
    "user_info": {
      "email": "john.doe@example.com",
      "phone_number": "+2348012345678",
      "account_age_days": 180
    },
    "device_info": {
      "device_id": "dev_android_001",
      "device_type": "MOBILE",
      "os": "Android",
      "browser": "Chrome",
      "ip_address": "197.210.55.123",
      "is_vpn": false,
      "is_proxy": false,
      "is_new_device": false
    },
    "location_info": {
      "country": "Nigeria",
      "city": "Lagos",
      "latitude": 6.5244,
      "longitude": 3.3792
    }
  }' | jq
```

**Expected response:**
```json
{
  "transaction_id": "tx_low_risk_001",
  "fraud_score": 0,
  "risk_level": "LOW",
  "is_fraudulent": false,
  "fraud_flags": [],
  "timestamp": "2024-01-15T10:30:00Z",
  "recommendation": "APPROVE - Transaction appears legitimate"
}
```

---

#### Test 2: Medium-Risk Transaction (Large Amount)

```bash
curl -X POST "http://localhost:8000/api/v1/check-fraud" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_medium_risk_001",
    "user_id": "user_002",
    "amount": 150000,
    "currency": "NGN",
    "transaction_type": "WITHDRAWAL",
    "user_info": {
      "email": "jane.smith@example.com",
      "phone_number": "+2348098765432",
      "account_age_days": 30
    },
    "device_info": {
      "device_id": "dev_ios_001",
      "device_type": "MOBILE",
      "os": "iOS",
      "browser": "Safari",
      "ip_address": "197.210.55.200",
      "is_vpn": false,
      "is_proxy": false,
      "is_new_device": true
    },
    "location_info": {
      "country": "Nigeria",
      "city": "Abuja",
      "latitude": 9.0765,
      "longitude": 7.3986
    }
  }' | jq
```

**Expected response:**
```json
{
  "transaction_id": "tx_medium_risk_001",
  "fraud_score": 50,
  "risk_level": "MEDIUM",
  "is_fraudulent": false,
  "fraud_flags": [
    {
      "flag_type": "large_amount",
      "severity": "HIGH",
      "description": "Transaction amount ₦150,000.00 exceeds threshold"
    },
    {
      "flag_type": "new_device",
      "severity": "MEDIUM",
      "description": "Transaction from new/unknown device"
    }
  ],
  "timestamp": "2024-01-15T10:35:00Z",
  "recommendation": "REVIEW - Manual review recommended"
}
```

---

#### Test 3: High-Risk Transaction (VPN + Large Amount)

```bash
curl -X POST "http://localhost:8000/api/v1/check-fraud" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_high_risk_001",
    "user_id": "user_003",
    "amount": 500000,
    "currency": "NGN",
    "transaction_type": "WITHDRAWAL",
    "user_info": {
      "email": "suspicious@tempmail.com",
      "phone_number": "+2347012345678",
      "account_age_days": 5
    },
    "device_info": {
      "device_id": "dev_unknown_001",
      "device_type": "DESKTOP",
      "os": "Windows",
      "browser": "Firefox",
      "ip_address": "104.20.45.67",
      "is_vpn": true,
      "is_proxy": false,
      "is_new_device": true
    },
    "location_info": {
      "country": "United States",
      "city": "New York",
      "latitude": 40.7128,
      "longitude": -74.0060
    }
  }' | jq
```

**Expected response:**
```json
{
  "transaction_id": "tx_high_risk_001",
  "fraud_score": 85,
  "risk_level": "HIGH",
  "is_fraudulent": true,
  "fraud_flags": [
    {
      "flag_type": "large_amount",
      "severity": "HIGH",
      "description": "Transaction amount ₦500,000.00 exceeds threshold"
    },
    {
      "flag_type": "new_device",
      "severity": "MEDIUM",
      "description": "Transaction from new/unknown device"
    },
    {
      "flag_type": "vpn_detected",
      "severity": "HIGH",
      "description": "Transaction from VPN or anonymizer"
    },
    {
      "flag_type": "new_account",
      "severity": "MEDIUM",
      "description": "Account created less than 30 days ago"
    }
  ],
  "timestamp": "2024-01-15T10:40:00Z",
  "recommendation": "BLOCK - High fraud risk detected"
}
```

---

#### Test 4: Retrieve Transaction

```bash
curl -X GET "http://localhost:8000/api/v1/transactions/tx_low_risk_001" | jq
```

**Expected response:**
```json
{
  "transaction_id": "tx_low_risk_001",
  "fraud_score": 0,
  "risk_level": "LOW",
  ...
}
```

---

#### Test 5: List Recent Transactions

```bash
curl -X GET "http://localhost:8000/api/v1/transactions?limit=5&offset=0" | jq
```

**Expected response:**
```json
[
  {
    "transaction_id": "tx_high_risk_001",
    "fraud_score": 85,
    "risk_level": "HIGH",
    ...
  },
  {
    "transaction_id": "tx_medium_risk_001",
    "fraud_score": 50,
    "risk_level": "MEDIUM",
    ...
  },
  ...
]
```

---

#### Test 6: Get User Fraud History

```bash
curl -X GET "http://localhost:8000/api/v1/users/user_001/fraud-history?days=30" | jq
```

**Expected response:**
```json
{
  "user_id": "user_001",
  "period_days": 30,
  "total_transactions": 1,
  "flagged_transactions": 0,
  "total_amount": 25000,
  "average_fraud_score": 0,
  "current_risk_score": 0,
  "account_status": "ACTIVE"
}
```

---

### Step 4: Verify Database

Check that transactions were saved to PostgreSQL:

```bash
# Connect to PostgreSQL
psql -U sentinel_user -d sentinel_db

# Query fraud transactions
SELECT
    transaction_id,
    user_id,
    amount,
    fraud_score,
    risk_level,
    is_flagged,
    created_at
FROM fraud_transactions
ORDER BY created_at DESC
LIMIT 5;

# Expected output:
#   transaction_id    |   user_id   | amount  | fraud_score | risk_level | is_flagged |         created_at
# --------------------+-------------+---------+-------------+------------+------------+----------------------------
#  tx_high_risk_001   | user_003    | 500000  |          85 | HIGH       | t          | 2024-01-15 10:40:00.123456
#  tx_medium_risk_001 | user_002    | 150000  |          50 | MEDIUM     | t          | 2024-01-15 10:35:00.123456
#  tx_low_risk_001    | user_001    |  25000  |           0 | LOW        | f          | 2024-01-15 10:30:00.123456

# Query user risk profiles
SELECT
    user_id,
    risk_score,
    transaction_count,
    status,
    last_transaction_at
FROM user_risk_profiles
ORDER BY risk_score DESC;

# Exit PostgreSQL
\q
```

---

## Understanding the Request Flow

Let's trace a complete fraud check request through the system:

### Request Flow Diagram

```
1. CLIENT SENDS REQUEST
   ↓
   POST /api/v1/check-fraud
   {
     "transaction_id": "tx_123",
     "amount": 150000,
     ...
   }

2. FASTAPI VALIDATES REQUEST
   ↓
   Pydantic: TransactionCheckRequest
   - Validates amount > 0
   - Validates email format
   - Validates required fields
   ✅ Valid → Continue
   ❌ Invalid → Return 422 error

3. DEPENDENCY INJECTION
   ↓
   get_db() creates database session
   Session injected into route handler

4. ROUTE HANDLER
   ↓
   fraud.py: check_fraud()
   - Receives validated request
   - Receives database session
   - Creates FraudService
   - Calls service.check_fraud()

5. SERVICE LAYER
   ↓
   fraud_service.py: check_fraud()
   - Runs rules engine
   - Calculates fraud score
   - Determines risk level
   - Updates user profile
   - Saves to database
   - Returns response

6. RULES ENGINE
   ↓
   rules_engine.py: evaluate()
   - Rule 1: Check large amount → Score +30
   - Rule 2: Check new device → Score +20
   - Rule 3: Check VPN → Score +25
   - Rule 4: Check account age → Score +10
   Total Score: 85

7. DATABASE PERSISTENCE
   ↓
   - Insert into fraud_transactions
   - Update user_risk_profiles
   - Update device_fingerprints
   - Commit transaction

8. RESPONSE BUILDING
   ↓
   FraudCheckResponse:
   {
     "fraud_score": 85,
     "risk_level": "HIGH",
     "is_fraudulent": true,
     "recommendation": "BLOCK",
     ...
   }

9. FASTAPI SERIALIZES RESPONSE
   ↓
   Pydantic converts to JSON
   Validates response schema

10. CLIENT RECEIVES RESPONSE
    ↓
    HTTP 200 OK
    {
      "fraud_score": 85,
      "risk_level": "HIGH",
      ...
    }
```

### Code Flow Example

```python
# 1. Client request arrives at route handler
@router.post("/api/v1/check-fraud")
async def check_fraud(
    request: TransactionCheckRequest,  # 2. Pydantic validates
    db: Session = Depends(get_db)      # 3. DB session injected
) -> FraudCheckResponse:

    # 4. Create service and call business logic
    service = FraudService()
    result = service.check_fraud(request, db)  # 5. Service layer

    # 10. Return response (FastAPI serializes automatically)
    return result


# 5. Service layer business logic
class FraudService:
    def check_fraud(self, request, db):
        # 6. Run rules engine
        rule_results = self.rules_engine.evaluate(request)

        # Calculate score (sum of triggered rules)
        fraud_score = sum(r.score for r in rule_results if r.triggered)

        # Determine risk level
        risk_level = self._determine_risk_level(fraud_score)

        # 7. Save to database
        fraud_tx = FraudTransaction(
            transaction_id=request.transaction_id,
            fraud_score=fraud_score,
            risk_level=risk_level.value,
            ...
        )
        db.add(fraud_tx)
        db.commit()

        # 8. Build response
        return FraudCheckResponse(
            transaction_id=request.transaction_id,
            fraud_score=fraud_score,
            risk_level=risk_level,
            ...
        )
```

---

## Error Handling Deep Dive

### 1. Validation Errors (422)

**Trigger:** Invalid request data

```bash
curl -X POST "http://localhost:8000/api/v1/check-fraud" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_001",
    "amount": -1000,
    "currency": "NGN"
  }'
```

**Response:**
```json
{
  "detail": [
    {
      "loc": ["body", "amount"],
      "msg": "ensure this value is greater than 0",
      "type": "value_error.number.not_gt"
    },
    {
      "loc": ["body", "user_id"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```

**Why it happens:** Pydantic validation fails before route handler runs.

---

### 2. Not Found Errors (404)

**Trigger:** Transaction doesn't exist

```bash
curl -X GET "http://localhost:8000/api/v1/transactions/nonexistent_tx"
```

**Response:**
```json
{
  "detail": "Transaction nonexistent_tx not found"
}
```

**Code:**
```python
if not result:
    raise HTTPException(
        status_code=404,
        detail=f"Transaction {transaction_id} not found"
    )
```

---

### 3. Server Errors (500)

**Trigger:** Database connection failure, unhandled exception

**Response:**
```json
{
  "error": "Internal server error",
  "message": "connection to database failed",
  "path": "/api/v1/check-fraud"
}
```

**Code:**
```python
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "message": str(exc),
            "path": str(request.url)
        }
    )
```

---

## Troubleshooting

### Issue 1: Module Import Errors

**Error:**
```
ModuleNotFoundError: No module named 'app.api'
```

**Solution:**
```bash
# Ensure you're in project root
cd /home/user/sentinel-api

# Verify directory structure
ls -la app/api/

# Expected files:
# __init__.py
# fraud.py
# dependencies.py

# If files missing, recreate them
touch app/api/__init__.py
touch app/api/fraud.py
touch app/api/dependencies.py
```

---

### Issue 2: Database Connection Error

**Error:**
```
sqlalchemy.exc.OperationalError: could not connect to server
```

**Solution:**
```bash
# Check PostgreSQL is running
sudo systemctl status postgresql

# If not running, start it
sudo systemctl start postgresql

# Verify connection
psql -U sentinel_user -d sentinel_db -c "SELECT 1;"

# Check .env file has correct credentials
cat .env
```

---

### Issue 3: Rules Engine Not Found

**Error:**
```
ModuleNotFoundError: No module named 'app.rules'
```

**Solution:**
You need to create the rules engine from Day 4. Create this file:

**File: `/home/user/sentinel-api/app/rules/engine.py`**

```python
"""
Simple Rules Engine for Day 5.
(Full implementation would be from Day 4)
"""

from typing import List
from dataclasses import dataclass
from app.schemas.fraud import TransactionCheckRequest


@dataclass
class RuleResult:
    """Result from a single rule evaluation."""
    rule_name: str
    triggered: bool
    score: int
    reason: str


class RulesEngine:
    """Evaluates fraud detection rules."""

    def evaluate(self, request: TransactionCheckRequest) -> List[RuleResult]:
        """Run all rules and return results."""
        results = []

        # Rule 1: Large Amount
        if request.amount > 100000:
            results.append(RuleResult(
                rule_name="large_amount",
                triggered=True,
                score=30,
                reason=f"Transaction amount ₦{request.amount:,.2f} exceeds ₦100,000 threshold"
            ))

        # Rule 2: New Device
        if request.device_info and request.device_info.is_new_device:
            results.append(RuleResult(
                rule_name="new_device",
                triggered=True,
                score=20,
                reason="Transaction from new/unknown device"
            ))

        # Rule 3: VPN Detection
        if request.device_info and request.device_info.is_vpn:
            results.append(RuleResult(
                rule_name="vpn_detected",
                triggered=True,
                score=25,
                reason="Transaction from VPN or anonymizer"
            ))

        # Rule 4: New Account
        if request.user_info and request.user_info.account_age_days < 30:
            results.append(RuleResult(
                rule_name="new_account",
                triggered=True,
                score=15,
                reason=f"Account created {request.user_info.account_age_days} days ago (< 30 days)"
            ))

        return results
```

**Create __init__.py:**
```bash
mkdir -p /home/user/sentinel-api/app/rules
touch /home/user/sentinel-api/app/rules/__init__.py
```

---

### Issue 4: Pydantic Validation Fails

**Error:**
```json
{
  "detail": [
    {
      "loc": ["body", "risk_level"],
      "msg": "value is not a valid enumeration member",
      "type": "type_error.enum"
    }
  ]
}
```

**Solution:**
Make sure RiskLevel enum is defined in schemas:

```python
# app/schemas/fraud.py
from enum import Enum

class RiskLevel(str, Enum):
    """Risk level enumeration."""
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"
```

---

### Issue 5: Swagger UI Not Loading

**Error:**
Browser shows blank page at `/docs`

**Solution:**
```bash
# Check server is running
curl http://localhost:8000/health

# Check for errors in terminal
# Look for:
# - Import errors
# - Syntax errors
# - Missing dependencies

# Restart server with verbose logging
uvicorn app.main:app --reload --log-level debug
```

---

## Key Takeaways

### What You Built Today

1. **Service Layer Pattern**
   - Separated business logic from API routes
   - Made code testable and reusable
   - Followed Single Responsibility Principle

2. **Dependency Injection**
   - Automatic database session management
   - Clean resource cleanup
   - Easy testing with mocks

3. **Complete Fraud Detection Flow**
   - Request validation → Rules engine → Score calculation → Database persistence → Response

4. **CRUD Operations**
   - Create: POST /api/v1/check-fraud
   - Read: GET /api/v1/transactions/{id}
   - List: GET /api/v1/transactions
   - History: GET /api/v1/users/{user_id}/fraud-history

5. **Error Handling**
   - Validation errors (422)
   - Not found errors (404)
   - Server errors (500)

### Project Status

```
✅ Day 1: FastAPI foundation
✅ Day 2: PostgreSQL database
✅ Day 3: Pydantic schemas
✅ Day 4: Rules engine
✅ Day 5: Complete fraud detection API ← YOU ARE HERE
⬜ Day 6: Redis caching
⬜ Day 7: Advanced caching
⬜ Day 8: ML fundamentals
```

### Skills Gained

- ✅ API design and versioning
- ✅ Service layer architecture
- ✅ Dependency injection pattern
- ✅ Database transaction management
- ✅ Error handling strategies
- ✅ API documentation
- ✅ Testing with curl and Swagger

---

## Next Steps

### Day 6: Redis Setup & Caching

Tomorrow you'll add Redis caching to achieve **50x performance improvement**:

**What you'll build:**
- Redis connection setup
- Cache fraud check results
- Implement cache-aside pattern
- Add TTL (time-to-live) strategies
- Measure performance improvements

**Why caching matters:**
```
Without cache:
  Every request → Query database → 50ms average

With cache:
  First request → Query database → Cache result → 50ms
  Next requests → Read from cache → 1ms average

Result: 50x faster! ⚡
```

### Recommended Reading

1. **FastAPI Documentation**
   - [Dependency Injection](https://fastapi.tiangolo.com/tutorial/dependencies/)
   - [Error Handling](https://fastapi.tiangolo.com/tutorial/handling-errors/)

2. **SQLAlchemy Documentation**
   - [Session Basics](https://docs.sqlalchemy.org/en/14/orm/session_basics.html)
   - [Relationship Patterns](https://docs.sqlalchemy.org/en/14/orm/relationships.html)

3. **Design Patterns**
   - Service Layer Pattern
   - Repository Pattern
   - Dependency Injection

### Practice Exercises

1. **Add New Endpoint**
   - Create DELETE /api/v1/transactions/{id}
   - Soft delete (set deleted_at timestamp)
   - Update service layer

2. **Add Filtering**
   - Filter transactions by risk_level
   - Filter by date range
   - Filter by user_id

3. **Add Statistics Endpoint**
   - GET /api/v1/statistics
   - Return: total transactions, flagged %, average score

### Challenge: Add Email Notifications

```python
# Challenge: Send email when high-risk transaction detected

class FraudService:
    async def check_fraud(self, request, db):
        result = # ... fraud check logic

        # If high risk, send notification
        if result.risk_level == RiskLevel.HIGH:
            await self._send_alert_email(
                to="security@company.com",
                subject=f"High-Risk Transaction: {request.transaction_id}",
                body=f"Fraud score: {result.fraud_score}"
            )

        return result
```

---

**Navigation:** [← Previous: Day 4](README-DAY-004.md) | [Main Guide](README.md) | [Next: Day 6 →](README-DAY-006.md)

---

**Congratulations!** 🎉

You've successfully built a complete fraud detection API that integrates all components from Days 1-4. Your system can now:

✅ Accept transactions via REST API
✅ Validate data with Pydantic
✅ Run fraud detection rules
✅ Calculate fraud scores
✅ Store results in PostgreSQL
✅ Retrieve fraud history
✅ Handle errors gracefully

This is a major milestone! Tomorrow we'll make it **50x faster** with Redis caching.

Keep building! 🚀

---

**Questions or Issues?**

If you encounter problems:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review error messages carefully
3. Verify all files from Days 1-4 are in place
4. Check database connection
5. Ensure virtual environment is activated

**Happy coding!** 💻
