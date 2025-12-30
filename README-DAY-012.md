# Day 12: Advanced Lending Fraud Rules (Part 1)

## Overview

Welcome to Day 12 of building Sentinel! Today we're implementing **5 advanced lending fraud detection rules** specifically designed for the Nigerian fintech lending landscape. These rules build directly on the rules engine we created on Day 4 and leverage the database models from Day 2.

**What we're building:**
- Loan stacking detection (multiple simultaneous loans)
- SIM swap pattern recognition
- Income manipulation detection
- Rapid repeat application analysis
- Synthetic identity fraud detection

**Why this matters:**
Nigeria's digital lending market has exploded in recent years, with dozens of fintech companies offering instant loans. This rapid growth has attracted sophisticated fraudsters who exploit:
- Lack of centralized credit bureaus
- Easy SIM card replacement
- Weak identity verification
- Multiple lending platforms
- Cash-based economy making income verification difficult

**Prerequisites:**
- Day 4: Rules engine (`app/rules/base.py`, `app/rules/engine.py`)
- Day 2: Database models (`app/models.py`)
- Day 1: FastAPI setup

**No new packages today!** We'll use the existing rules framework.

---

## Understanding Lending Fraud in Nigeria

### The Nigerian Lending Landscape

**Key Characteristics:**
1. **Multiple platforms**: Users can apply to 10+ lenders simultaneously
2. **Instant disbursement**: Loans approved in minutes create urgency attacks
3. **BVN as primary ID**: Bank Verification Number is the main identity anchor
4. **SIM swap vulnerability**: Easy to replace SIM cards at telecom shops
5. **Limited credit history**: Most users have no formal credit records

**Common Fraud Patterns:**

**1. Loan Stacking**
Fraudsters apply to multiple lenders simultaneously, receive 5-10 loans within hours, then disappear. They exploit the lack of real-time data sharing between platforms.

**2. SIM Swap + Account Takeover**
Attacker swaps victim's SIM, resets passwords, applies for loans in victim's name, and diverts funds.

**3. Income Inflation**
Users declare salaries of ₦500,000/month when they earn ₦80,000, knowing income verification is weak.

**4. Application Spam**
Desperate users apply to every platform repeatedly, getting rejected but creating fraud-like patterns.

**5. Synthetic Identities**
Fraudsters create "fresh" identities using real BVNs combined with new phone numbers and email addresses.

---

## Architecture Overview

### Directory Structure

```
app/
└── rules/
    ├── __init__.py
    ├── base.py              # From Day 4
    ├── engine.py            # From Day 4
    └── lending/
        ├── __init__.py
        ├── loan_stacking.py
        ├── sim_swap.py
        ├── income_mismatch.py
        ├── rapid_repeat.py
        └── identity_fraud.py
```

### How Lending Rules Work

Each lending rule:
1. **Inherits from `Rule` base class** (Day 4)
2. **Analyzes lending-specific data**: loan history, income, identity
3. **Returns risk scores**: 0-100 scale
4. **Provides evidence**: Detailed explanation of findings
5. **Integrates with rules engine**: Automatic evaluation

### Data Sources

Our rules will check:
- **Transaction history**: Past transactions from the database
- **Loan applications**: Previous loan requests
- **Identity data**: BVN, phone, email metadata
- **Consortium data**: Shared intelligence across platforms (simulated)
- **Behavioral patterns**: Application timing, amounts, patterns

---

## Rule 1: Loan Stacking Detection

### What is Loan Stacking?

**Definition**: When a borrower takes out multiple loans from different lenders in a short period, often with intent to default.

**Nigerian Context**: With 50+ digital lenders and no real-time credit bureau, a fraudster can:
1. Apply to 10 lenders on Monday morning
2. Get approved by 7 lenders by Monday afternoon
3. Receive ₦500,000 total disbursement
4. Disappear by Tuesday

### Detection Strategy

We'll detect loan stacking by:
1. **Checking active loan count**: How many loans are currently active?
2. **Recent applications**: How many loan requests in the last 7 days?
3. **Cross-platform intelligence**: Consortium data sharing
4. **Disbursement velocity**: Multiple loans funded within hours

### Implementation

Create the lending directory and loan stacking rule:

```bash
mkdir -p /home/user/guide-sentinel/app/rules/lending
```

**File: `/home/user/guide-sentinel/app/rules/lending/__init__.py`**

```python
"""
Lending-specific fraud detection rules.

This module contains rules designed to detect fraud patterns
specific to digital lending in Nigeria.
"""

from .loan_stacking import LoanStackingRule
from .sim_swap import SimSwapRule
from .income_mismatch import IncomeMismatchRule
from .rapid_repeat import RapidRepeatApplicationRule
from .identity_fraud import SyntheticIdentityRule

__all__ = [
    'LoanStackingRule',
    'SimSwapRule',
    'IncomeMismatchRule',
    'RapidRepeatApplicationRule',
    'SyntheticIdentityRule',
]
```

**File: `/home/user/guide-sentinel/app/rules/lending/loan_stacking.py`**

```python
"""
Loan Stacking Detection Rule

Detects when a user has multiple active loans across platforms,
indicating potential loan stacking fraud.

Nigerian Context:
- Users can apply to 50+ digital lenders
- No real-time credit bureau sharing
- Fraudsters exploit this to get multiple loans simultaneously
"""

from datetime import datetime, timedelta
from typing import Dict, Any, List
from sqlalchemy.orm import Session
from sqlalchemy import and_, func

from app.rules.base import Rule, RuleResult
from app.models import Transaction, User


class LoanStackingRule(Rule):
    """
    Detects loan stacking patterns.

    Loan stacking occurs when a borrower takes multiple loans
    from different lenders in a short period. This is a major
    fraud vector in Nigerian fintech.

    Risk Indicators:
    - Multiple active loans (>3 is high risk)
    - Multiple loan applications in past 7 days
    - Recent loan disbursements from other platforms
    - High total debt-to-income ratio
    """

    def __init__(self):
        super().__init__(
            rule_id="LENDING_001",
            name="Loan Stacking Detection",
            description="Detects multiple simultaneous loans across platforms",
            category="lending",
            severity="high"
        )

        # Configuration thresholds
        self.HIGH_RISK_LOAN_COUNT = 3
        self.MEDIUM_RISK_LOAN_COUNT = 2
        self.APPLICATION_WINDOW_DAYS = 7
        self.HIGH_RISK_APPLICATIONS = 5
        self.MEDIUM_RISK_APPLICATIONS = 3

    def evaluate(self, db: Session, transaction_data: Dict[str, Any]) -> RuleResult:
        """
        Evaluate loan stacking risk.

        Args:
            db: Database session
            transaction_data: Transaction context including user_id

        Returns:
            RuleResult with risk score and evidence
        """
        user_id = transaction_data.get('user_id')

        if not user_id:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="No user_id provided for loan stacking check",
                evidence={}
            )

        # Fetch user
        user = db.query(User).filter(User.id == user_id).first()
        if not user:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="User not found",
                evidence={}
            )

        # Check active loans
        active_loans = self._get_active_loans(db, user_id)

        # Check recent loan applications
        recent_applications = self._get_recent_applications(db, user_id)

        # Check recent disbursements
        recent_disbursements = self._get_recent_disbursements(db, user_id)

        # Calculate risk score
        risk_score = self._calculate_risk_score(
            active_loans,
            recent_applications,
            recent_disbursements
        )

        # Build evidence
        evidence = {
            "active_loan_count": active_loans["count"],
            "active_loan_platforms": active_loans["platforms"],
            "total_active_amount": active_loans["total_amount"],
            "recent_applications_7d": len(recent_applications),
            "recent_disbursements_7d": len(recent_disbursements),
            "application_platforms": list(set([app["platform"] for app in recent_applications])),
            "user_email": user.email,
            "check_timestamp": datetime.utcnow().isoformat()
        }

        # Determine if rule passed (lower risk score = pass)
        passed = risk_score < 60

        # Build message
        if risk_score >= 80:
            message = f"HIGH RISK: User has {active_loans['count']} active loans and {len(recent_applications)} applications in 7 days"
        elif risk_score >= 60:
            message = f"MEDIUM RISK: User has {active_loans['count']} active loans"
        else:
            message = f"Low risk: {active_loans['count']} active loans detected"

        return RuleResult(
            rule_id=self.rule_id,
            passed=passed,
            risk_score=risk_score,
            message=message,
            evidence=evidence
        )

    def _get_active_loans(self, db: Session, user_id: str) -> Dict[str, Any]:
        """
        Get count and details of active loans.

        In production, this would query a loans table or consortium API.
        For now, we simulate by checking loan disbursement transactions
        that haven't been repaid.
        """
        # Simulate consortium data - in production this would be an API call
        # to a credit bureau or lending consortium

        # Look for loan disbursement transactions in the last 90 days
        # that are marked as "loan_disbursement" type
        ninety_days_ago = datetime.utcnow() - timedelta(days=90)

        # In our simulation, we'll use metadata to mark loan transactions
        loan_disbursements = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user_id,
                Transaction.created_at >= ninety_days_ago,
                Transaction.metadata.contains({"transaction_type": "loan_disbursement"})
            )
        ).all()

        # Simulate platform data
        platforms = []
        total_amount = 0

        for loan in loan_disbursements:
            # Check if this loan has been repaid
            metadata = loan.metadata or {}
            if not metadata.get("repaid", False):
                platform = metadata.get("platform", "unknown")
                platforms.append(platform)
                total_amount += abs(loan.amount)

        return {
            "count": len(platforms),
            "platforms": platforms,
            "total_amount": total_amount
        }

    def _get_recent_applications(self, db: Session, user_id: str) -> List[Dict[str, Any]]:
        """
        Get recent loan applications.

        In production, this would check a loan_applications table.
        We simulate by looking for specific transaction metadata.
        """
        cutoff_date = datetime.utcnow() - timedelta(days=self.APPLICATION_WINDOW_DAYS)

        # Look for transactions marked as loan applications
        applications = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user_id,
                Transaction.created_at >= cutoff_date,
                Transaction.metadata.contains({"transaction_type": "loan_application"})
            )
        ).all()

        return [
            {
                "platform": app.metadata.get("platform", "unknown"),
                "amount": abs(app.amount),
                "timestamp": app.created_at.isoformat()
            }
            for app in applications
        ]

    def _get_recent_disbursements(self, db: Session, user_id: str) -> List[Dict[str, Any]]:
        """Get loan disbursements in the last 7 days."""
        cutoff_date = datetime.utcnow() - timedelta(days=7)

        disbursements = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user_id,
                Transaction.created_at >= cutoff_date,
                Transaction.metadata.contains({"transaction_type": "loan_disbursement"})
            )
        ).all()

        return [
            {
                "platform": disb.metadata.get("platform", "unknown"),
                "amount": abs(disb.amount),
                "timestamp": disb.created_at.isoformat()
            }
            for disb in disbursements
        ]

    def _calculate_risk_score(
        self,
        active_loans: Dict[str, Any],
        recent_applications: List[Dict[str, Any]],
        recent_disbursements: List[Dict[str, Any]]
    ) -> int:
        """
        Calculate overall loan stacking risk score (0-100).

        Scoring logic:
        - Active loans: 25 points per loan above threshold
        - Recent applications: 10 points per application above threshold
        - Recent disbursements: 15 points per disbursement
        - Multiple platforms: Bonus 20 points if >3 platforms
        """
        score = 0

        # Score active loans
        loan_count = active_loans["count"]
        if loan_count >= self.HIGH_RISK_LOAN_COUNT:
            score += 50
            score += (loan_count - self.HIGH_RISK_LOAN_COUNT) * 10
        elif loan_count >= self.MEDIUM_RISK_LOAN_COUNT:
            score += 30

        # Score recent applications
        app_count = len(recent_applications)
        if app_count >= self.HIGH_RISK_APPLICATIONS:
            score += 30
        elif app_count >= self.MEDIUM_RISK_APPLICATIONS:
            score += 15

        # Score recent disbursements (very suspicious if multiple in 7 days)
        disb_count = len(recent_disbursements)
        if disb_count >= 3:
            score += 40
        elif disb_count >= 2:
            score += 20

        # Platform diversity penalty
        platform_count = len(active_loans["platforms"])
        if platform_count >= 4:
            score += 20
        elif platform_count >= 3:
            score += 10

        # Cap at 100
        return min(score, 100)
```

### How It Works

**Step 1: Active Loan Detection**
```python
active_loans = self._get_active_loans(db, user_id)
# Returns: {"count": 4, "platforms": ["PayLater", "FairMoney", "Carbon", "Branch"]}
```

**Step 2: Recent Application Check**
```python
recent_applications = self._get_recent_applications(db, user_id)
# Returns: [{"platform": "PayLater", "amount": 50000, "timestamp": "..."}]
```

**Step 3: Risk Scoring**
- 4 active loans = 50 base points + 10 bonus = 60 points
- 5+ applications = 30 points
- Multiple disbursements in 7 days = 40 points
- 4+ platforms = 20 points

**Result**: High-risk users score 80+, triggering additional verification.

---

## Rule 2: SIM Swap Detection

### Why SIM Swap Fraud is Critical in Nigeria

**The Attack Pattern:**
1. Fraudster obtains victim's BVN and phone number (data breach, social engineering)
2. Visits telecom shop with fake ID, requests SIM replacement
3. Receives new SIM with victim's phone number
4. Resets passwords for victim's accounts (OTP goes to fraudster)
5. Applies for loans in victim's name
6. Diverts funds to mule account

**Why it's so effective in Nigeria:**
- SIM replacement is easy (just visit any telecom shop)
- Many platforms rely solely on SMS OTP for authentication
- Victims may not notice for 24-48 hours
- By then, multiple loans are disbursed

### Detection Strategy

We detect SIM swap risk by:
1. **Phone number age**: How long has user had this number?
2. **Recent changes**: Was phone number changed in last 7-30 days?
3. **Application timing**: Loan request shortly after SIM/phone change?
4. **Behavioral changes**: Sudden change in device, location after SIM swap?

### Implementation

**File: `/home/user/guide-sentinel/app/rules/lending/sim_swap.py`**

```python
"""
SIM Swap Fraud Detection Rule

Detects recent SIM card changes that may indicate account takeover
or SIM swap fraud.

Nigerian Context:
- SIM replacement is very easy at telecom shops
- Fraudsters use SIM swap to bypass SMS OTP authentication
- Major vector for lending fraud and account takeover
"""

from datetime import datetime, timedelta
from typing import Dict, Any
from sqlalchemy.orm import Session
from sqlalchemy import and_

from app.rules.base import Rule, RuleResult
from app.models import User, Transaction


class SimSwapRule(Rule):
    """
    Detects SIM swap fraud patterns.

    SIM swap fraud occurs when an attacker convinces a telecom
    provider to transfer a victim's phone number to a new SIM card.
    They then use SMS-based authentication to access accounts.

    Risk Indicators:
    - Phone number changed within last 7 days (HIGH RISK)
    - Phone number changed within last 30 days (MEDIUM RISK)
    - Loan application within 24 hours of phone change
    - Device/location change concurrent with phone change
    - New phone number with old BVN
    """

    def __init__(self):
        super().__init__(
            rule_id="LENDING_002",
            name="SIM Swap Detection",
            description="Detects recent SIM card changes indicating potential fraud",
            category="lending",
            severity="critical"
        )

        # Configuration thresholds
        self.CRITICAL_WINDOW_DAYS = 7
        self.HIGH_RISK_WINDOW_DAYS = 14
        self.MEDIUM_RISK_WINDOW_DAYS = 30
        self.MIN_PHONE_AGE_DAYS = 30  # Prefer phone numbers >30 days old

    def evaluate(self, db: Session, transaction_data: Dict[str, Any]) -> RuleResult:
        """
        Evaluate SIM swap risk.

        Args:
            db: Database session
            transaction_data: Transaction context including user_id

        Returns:
            RuleResult with risk score and evidence
        """
        user_id = transaction_data.get('user_id')

        if not user_id:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="No user_id provided for SIM swap check",
                evidence={}
            )

        # Fetch user
        user = db.query(User).filter(User.id == user_id).first()
        if not user:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="User not found",
                evidence={}
            )

        # Check phone number age
        phone_age_result = self._check_phone_age(user)

        # Check for recent phone number changes
        phone_change_result = self._check_recent_phone_changes(user)

        # Check for suspicious timing (loan app right after phone change)
        timing_result = self._check_application_timing(db, user, transaction_data)

        # Check device/location changes
        behavior_result = self._check_behavioral_changes(db, user)

        # Calculate risk score
        risk_score = self._calculate_risk_score(
            phone_age_result,
            phone_change_result,
            timing_result,
            behavior_result
        )

        # Build evidence
        evidence = {
            "phone_age_days": phone_age_result["age_days"],
            "phone_number": user.metadata.get("phone_number", "unknown"),
            "last_phone_change": phone_change_result["last_change"],
            "days_since_change": phone_change_result["days_since_change"],
            "application_after_change_hours": timing_result["hours_after_change"],
            "device_changed": behavior_result["device_changed"],
            "location_changed": behavior_result["location_changed"],
            "user_email": user.email,
            "check_timestamp": datetime.utcnow().isoformat()
        }

        # Determine if rule passed
        passed = risk_score < 70

        # Build message
        if risk_score >= 90:
            message = f"CRITICAL: SIM swap detected within {phone_change_result['days_since_change']} days of loan application"
        elif risk_score >= 70:
            message = f"HIGH RISK: Recent phone number change ({phone_change_result['days_since_change']} days ago)"
        elif risk_score >= 40:
            message = f"MEDIUM RISK: Phone number relatively new ({phone_age_result['age_days']} days old)"
        else:
            message = "Low risk: Phone number appears stable"

        return RuleResult(
            rule_id=self.rule_id,
            passed=passed,
            risk_score=risk_score,
            message=message,
            evidence=evidence
        )

    def _check_phone_age(self, user: User) -> Dict[str, Any]:
        """
        Check how long the user has had their current phone number.

        In production, this would query telecom APIs or consortium data.
        We simulate using user metadata.
        """
        metadata = user.metadata or {}

        # Check when phone number was first registered
        phone_registered = metadata.get("phone_registered_at")

        if phone_registered:
            # Parse date string
            registered_date = datetime.fromisoformat(phone_registered)
            age_days = (datetime.utcnow() - registered_date).days
        else:
            # Assume phone registered when user created account
            age_days = (datetime.utcnow() - user.created_at).days

        return {
            "age_days": age_days,
            "registered_at": phone_registered or user.created_at.isoformat()
        }

    def _check_recent_phone_changes(self, user: User) -> Dict[str, Any]:
        """
        Check if phone number was recently changed.

        In production, this would check a phone_number_history table
        or telecom provider APIs.
        """
        metadata = user.metadata or {}

        # Check for phone change history
        last_change = metadata.get("phone_last_changed")

        if last_change:
            change_date = datetime.fromisoformat(last_change)
            days_since_change = (datetime.utcnow() - change_date).days
        else:
            days_since_change = None
            last_change = None

        return {
            "last_change": last_change,
            "days_since_change": days_since_change
        }

    def _check_application_timing(
        self,
        db: Session,
        user: User,
        transaction_data: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Check if loan application came shortly after phone number change.

        Very suspicious if user changes phone then immediately applies for loan.
        """
        metadata = user.metadata or {}
        last_change = metadata.get("phone_last_changed")

        if not last_change:
            return {"hours_after_change": None}

        change_date = datetime.fromisoformat(last_change)
        application_time = datetime.utcnow()

        hours_difference = (application_time - change_date).total_seconds() / 3600

        return {
            "hours_after_change": round(hours_difference, 2)
        }

    def _check_behavioral_changes(self, db: Session, user: User) -> Dict[str, Any]:
        """
        Check for concurrent behavioral changes.

        If phone number changed AND device changed AND location changed,
        that's very suspicious.
        """
        metadata = user.metadata or {}

        # Check recent transactions for device/location changes
        recent_transactions = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user.id,
                Transaction.created_at >= datetime.utcnow() - timedelta(days=7)
            )
        ).order_by(Transaction.created_at.desc()).limit(10).all()

        devices = set()
        locations = set()

        for txn in recent_transactions:
            txn_meta = txn.metadata or {}
            if txn_meta.get("device_id"):
                devices.add(txn_meta["device_id"])
            if txn_meta.get("ip_address"):
                locations.add(txn_meta["ip_address"])

        # If we see multiple devices or IPs in last 7 days, flag it
        device_changed = len(devices) > 1
        location_changed = len(locations) > 2  # Some variation is normal

        return {
            "device_changed": device_changed,
            "location_changed": location_changed,
            "unique_devices": len(devices),
            "unique_ips": len(locations)
        }

    def _calculate_risk_score(
        self,
        phone_age_result: Dict[str, Any],
        phone_change_result: Dict[str, Any],
        timing_result: Dict[str, Any],
        behavior_result: Dict[str, Any]
    ) -> int:
        """
        Calculate SIM swap risk score (0-100).

        Scoring logic:
        - Phone change within 7 days: 60 points
        - Phone change within 14 days: 40 points
        - Phone change within 30 days: 20 points
        - Application within 24h of change: +30 points
        - Device change concurrent: +20 points
        - Location change concurrent: +10 points
        - Phone number very new (<30 days): +15 points
        """
        score = 0

        # Score based on phone change recency
        days_since_change = phone_change_result["days_since_change"]
        if days_since_change is not None:
            if days_since_change <= self.CRITICAL_WINDOW_DAYS:
                score += 60
            elif days_since_change <= self.HIGH_RISK_WINDOW_DAYS:
                score += 40
            elif days_since_change <= self.MEDIUM_RISK_WINDOW_DAYS:
                score += 20

        # Score based on application timing
        hours_after_change = timing_result["hours_after_change"]
        if hours_after_change is not None:
            if hours_after_change <= 24:
                score += 30
            elif hours_after_change <= 72:
                score += 15

        # Score based on behavioral changes
        if behavior_result["device_changed"]:
            score += 20
        if behavior_result["location_changed"]:
            score += 10

        # Score based on phone age
        phone_age_days = phone_age_result["age_days"]
        if phone_age_days < self.MIN_PHONE_AGE_DAYS:
            score += 15

        # Cap at 100
        return min(score, 100)
```

### How It Works

**Scenario 1: Legitimate User**
- Phone number: 2 years old
- No recent changes
- Risk score: 0-10 (PASS)

**Scenario 2: SIM Swap Attack**
- Phone changed 3 days ago
- Loan application 12 hours after change
- Device changed concurrently
- Risk score: 60 + 30 + 20 = 110 → capped at 100 (CRITICAL)

---

## Rule 3: Income Manipulation Detection

### The Income Verification Challenge

**The Problem:**
In Nigeria's largely cash-based economy, verifying income is extremely difficult:
- Many people work in informal sector (no pay stubs)
- Salary accounts may not show full income (cash payments)
- Self-employed users have irregular income
- Users routinely inflate income on applications

**Common Fraud:**
- User declares ₦500,000/month salary
- Bank statement shows ₦80,000 monthly inflow
- Spending patterns match ₦80,000 income level
- Clear income manipulation

### Detection Strategy

We detect income fraud by:
1. **Comparing declared vs observed income**: Check transaction patterns
2. **Spending analysis**: Does spending match declared income?
3. **Income consistency**: Regular salary credits vs irregular deposits
4. **Lifestyle matching**: High income should show in transaction patterns

### Implementation

**File: `/home/user/guide-sentinel/app/rules/lending/income_mismatch.py`**

```python
"""
Income Manipulation Detection Rule

Compares declared income with actual transaction patterns to detect
income inflation fraud.

Nigerian Context:
- Cash-based economy makes verification hard
- Users often inflate income on applications
- Bank statements may not show full picture
- Self-employed users have irregular income
"""

from datetime import datetime, timedelta
from typing import Dict, Any, List
from sqlalchemy.orm import Session
from sqlalchemy import and_, func

from app.rules.base import Rule, RuleResult
from app.models import Transaction, User


class IncomeMismatchRule(Rule):
    """
    Detects income manipulation through transaction analysis.

    This rule compares the user's declared income with their actual
    transaction patterns to identify significant discrepancies that
    may indicate fraud.

    Risk Indicators:
    - Declared income >> observed income (>2x difference)
    - Spending patterns don't match declared income
    - Irregular income (not consistent salary credits)
    - Recent sudden "income" deposits before application
    """

    def __init__(self):
        super().__init__(
            rule_id="LENDING_003",
            name="Income Manipulation Detection",
            description="Detects mismatch between declared and observed income",
            category="lending",
            severity="high"
        )

        # Configuration thresholds
        self.INCOME_ANALYSIS_DAYS = 90
        self.HIGH_RISK_MULTIPLIER = 2.5  # Declared is 2.5x observed
        self.MEDIUM_RISK_MULTIPLIER = 1.8
        self.MIN_TRANSACTIONS_REQUIRED = 10  # Need minimum data

    def evaluate(self, db: Session, transaction_data: Dict[str, Any]) -> RuleResult:
        """
        Evaluate income manipulation risk.

        Args:
            db: Database session
            transaction_data: Must include 'user_id' and 'declared_income'

        Returns:
            RuleResult with risk score and evidence
        """
        user_id = transaction_data.get('user_id')
        declared_income = transaction_data.get('declared_monthly_income', 0)

        if not user_id:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="No user_id provided for income check",
                evidence={}
            )

        if declared_income <= 0:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="No declared income provided",
                evidence={}
            )

        # Fetch user
        user = db.query(User).filter(User.id == user_id).first()
        if not user:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="User not found",
                evidence={}
            )

        # Analyze actual income from transactions
        income_analysis = self._analyze_observed_income(db, user_id)

        # Analyze spending patterns
        spending_analysis = self._analyze_spending(db, user_id)

        # Check for income manipulation patterns
        manipulation_signals = self._detect_manipulation_signals(
            db, user_id, declared_income
        )

        # Calculate risk score
        risk_score = self._calculate_risk_score(
            declared_income,
            income_analysis,
            spending_analysis,
            manipulation_signals
        )

        # Build evidence
        evidence = {
            "declared_monthly_income": declared_income,
            "observed_monthly_income": income_analysis["monthly_average"],
            "income_ratio": income_analysis["income_ratio"],
            "transaction_count": income_analysis["transaction_count"],
            "income_regularity": income_analysis["regularity"],
            "average_monthly_spending": spending_analysis["monthly_average"],
            "spending_income_ratio": spending_analysis["spending_income_ratio"],
            "manipulation_flags": manipulation_signals["flags"],
            "user_email": user.email,
            "check_timestamp": datetime.utcnow().isoformat()
        }

        # Determine if rule passed
        passed = risk_score < 60

        # Build message
        if risk_score >= 80:
            message = f"HIGH RISK: Declared income (₦{declared_income:,}) is {income_analysis['income_ratio']:.1f}x observed income"
        elif risk_score >= 60:
            message = f"MEDIUM RISK: Income mismatch detected (declared: ₦{declared_income:,}, observed: ₦{income_analysis['monthly_average']:,.0f})"
        else:
            message = "Low risk: Income appears consistent with transaction patterns"

        return RuleResult(
            rule_id=self.rule_id,
            passed=passed,
            risk_score=risk_score,
            message=message,
            evidence=evidence
        )

    def _analyze_observed_income(self, db: Session, user_id: str) -> Dict[str, Any]:
        """
        Analyze actual income from transaction history.

        We look for:
        - Regular salary credits (employer payments)
        - Total inflow amounts
        - Consistency of income
        """
        cutoff_date = datetime.utcnow() - timedelta(days=self.INCOME_ANALYSIS_DAYS)

        # Get all credit transactions (positive amounts)
        credits = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user_id,
                Transaction.created_at >= cutoff_date,
                Transaction.amount > 0,
                Transaction.status == 'completed'
            )
        ).all()

        if not credits:
            return {
                "monthly_average": 0,
                "transaction_count": 0,
                "regularity": "insufficient_data",
                "income_ratio": 0
            }

        # Calculate total inflow
        total_inflow = sum(txn.amount for txn in credits)

        # Calculate monthly average
        days_analyzed = min(self.INCOME_ANALYSIS_DAYS,
                           (datetime.utcnow() - credits[-1].created_at).days or 1)
        monthly_average = (total_inflow / days_analyzed) * 30

        # Detect salary-like patterns (regular monthly credits)
        salary_credits = self._detect_salary_pattern(credits)

        # Determine regularity
        if salary_credits["count"] >= 2:
            regularity = "regular_salary"
        elif len(credits) >= self.MIN_TRANSACTIONS_REQUIRED:
            regularity = "irregular_income"
        else:
            regularity = "insufficient_data"

        return {
            "monthly_average": monthly_average,
            "transaction_count": len(credits),
            "regularity": regularity,
            "salary_pattern": salary_credits,
            "total_inflow_90d": total_inflow,
            "income_ratio": 0  # Will be set later
        }

    def _detect_salary_pattern(self, transactions: List[Transaction]) -> Dict[str, Any]:
        """
        Detect regular salary-like deposits.

        Salary characteristics:
        - Similar amounts (within 20% variance)
        - Regular timing (monthly, bi-weekly)
        - Same source/description pattern
        """
        if len(transactions) < 2:
            return {"count": 0, "average_amount": 0}

        # Group transactions by month
        monthly_credits = {}
        for txn in transactions:
            month_key = txn.created_at.strftime("%Y-%m")
            if month_key not in monthly_credits:
                monthly_credits[month_key] = []
            monthly_credits[month_key].append(txn.amount)

        # Find largest credit per month (likely salary)
        potential_salaries = []
        for month, amounts in monthly_credits.items():
            if amounts:
                potential_salaries.append(max(amounts))

        if not potential_salaries:
            return {"count": 0, "average_amount": 0}

        # Check consistency
        avg_salary = sum(potential_salaries) / len(potential_salaries)
        consistent_count = sum(
            1 for amt in potential_salaries
            if abs(amt - avg_salary) / avg_salary <= 0.2  # Within 20%
        )

        return {
            "count": consistent_count,
            "average_amount": avg_salary,
            "months_analyzed": len(monthly_credits)
        }

    def _analyze_spending(self, db: Session, user_id: str) -> Dict[str, Any]:
        """
        Analyze spending patterns to verify income level.

        High earners should have high spending. If someone claims
        ₦500k salary but spends ₦80k/month, something is wrong.
        """
        cutoff_date = datetime.utcnow() - timedelta(days=self.INCOME_ANALYSIS_DAYS)

        # Get all debit transactions (negative amounts)
        debits = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user_id,
                Transaction.created_at >= cutoff_date,
                Transaction.amount < 0,
                Transaction.status == 'completed'
            )
        ).all()

        if not debits:
            return {
                "monthly_average": 0,
                "transaction_count": 0,
                "spending_income_ratio": 0
            }

        # Calculate total spending
        total_spending = sum(abs(txn.amount) for txn in debits)

        # Calculate monthly average
        days_analyzed = min(self.INCOME_ANALYSIS_DAYS,
                           (datetime.utcnow() - debits[-1].created_at).days or 1)
        monthly_average = (total_spending / days_analyzed) * 30

        return {
            "monthly_average": monthly_average,
            "transaction_count": len(debits),
            "total_spending_90d": total_spending,
            "spending_income_ratio": 0  # Will be calculated later
        }

    def _detect_manipulation_signals(
        self,
        db: Session,
        user_id: str,
        declared_income: float
    ) -> Dict[str, Any]:
        """
        Detect specific income manipulation patterns.

        Red flags:
        - Large deposits right before loan application
        - Round-number "salary" deposits
        - Income from gambling/suspicious sources
        """
        flags = []

        # Check for recent large deposits
        seven_days_ago = datetime.utcnow() - timedelta(days=7)
        recent_large_credits = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user_id,
                Transaction.created_at >= seven_days_ago,
                Transaction.amount > declared_income * 0.5,  # >50% of monthly income
                Transaction.status == 'completed'
            )
        ).all()

        if recent_large_credits:
            flags.append({
                "type": "recent_large_deposit",
                "description": f"Large deposit of ₦{recent_large_credits[0].amount:,.0f} within 7 days of application",
                "severity": "high"
            })

        # Check for suspicious round numbers
        # (Real salaries are rarely exactly ₦500,000.00)
        if declared_income % 10000 == 0 and declared_income >= 100000:
            flags.append({
                "type": "round_number_income",
                "description": f"Declared income is suspiciously round (₦{declared_income:,.0f})",
                "severity": "low"
            })

        return {
            "flags": flags,
            "flag_count": len(flags)
        }

    def _calculate_risk_score(
        self,
        declared_income: float,
        income_analysis: Dict[str, Any],
        spending_analysis: Dict[str, Any],
        manipulation_signals: Dict[str, Any]
    ) -> int:
        """
        Calculate income manipulation risk score (0-100).

        Scoring logic:
        - Large income ratio (declared/observed): Up to 50 points
        - Spending doesn't match declared income: Up to 30 points
        - Irregular income pattern: 10 points
        - Manipulation flags: 10 points each
        """
        score = 0

        # Calculate income ratio
        observed_income = income_analysis["monthly_average"]
        if observed_income > 0:
            income_ratio = declared_income / observed_income
            income_analysis["income_ratio"] = income_ratio

            # Score based on income mismatch
            if income_ratio >= self.HIGH_RISK_MULTIPLIER:
                score += 50
            elif income_ratio >= self.MEDIUM_RISK_MULTIPLIER:
                score += 30
            elif income_ratio >= 1.5:
                score += 15
        else:
            # No income data is itself suspicious
            score += 25

        # Score based on spending patterns
        if observed_income > 0:
            spending_income_ratio = spending_analysis["monthly_average"] / observed_income
            spending_analysis["spending_income_ratio"] = spending_income_ratio

            # If declared high income but spending is low, suspicious
            if declared_income > observed_income * 1.5:
                if spending_income_ratio < 0.5:  # Spending <50% of observed income
                    score += 20

        # Score based on income regularity
        if income_analysis["regularity"] == "irregular_income":
            score += 10
        elif income_analysis["regularity"] == "insufficient_data":
            score += 15

        # Score manipulation flags
        for flag in manipulation_signals["flags"]:
            if flag["severity"] == "high":
                score += 15
            elif flag["severity"] == "medium":
                score += 10
            else:
                score += 5

        # Cap at 100
        return min(score, 100)
```

### How It Works

**Example Analysis:**

**User declares**: ₦500,000/month salary

**Transaction analysis finds**:
- Average monthly inflow: ₦180,000
- Regular salary deposits: ₦150,000/month
- Monthly spending: ₦120,000

**Risk calculation**:
- Income ratio: 500,000 / 180,000 = 2.78x (HIGH RISK = 50 points)
- Spending matches observed income (not declared) = 20 points
- Regular salary pattern = 0 bonus points
- **Total**: 70 points (MEDIUM-HIGH RISK)

**Result**: Flag for manual income verification

---

## Rule 4: Rapid Repeat Applications

### Application Spam and Desperation

**Two Scenarios:**

**Scenario A: Fraudster**
- Applies to 15 lenders in one day
- Uses different emails/phones
- Looking to maximize loan stacking
- HIGH RISK

**Scenario B: Desperate Borrower**
- Rejected by 3 lenders
- Applies to 10 more hoping for approval
- Not fraud, but high credit risk
- MEDIUM RISK

Both patterns need detection, but require different responses.

### Detection Strategy

We detect rapid applications by:
1. **Application velocity**: How many applications in X days?
2. **Rejection history**: Were previous applications rejected?
3. **Cross-platform patterns**: Same user, different platforms
4. **Time spacing**: Applications within minutes vs. days

### Implementation

**File: `/home/user/guide-sentinel/app/rules/lending/rapid_repeat.py`**

```python
"""
Rapid Repeat Application Detection Rule

Detects when users submit multiple loan applications in a short
period, indicating either fraud (loan stacking) or credit distress.

Nigerian Context:
- Users can easily apply to 50+ lenders
- Application spam is common
- Fraudsters apply to many platforms simultaneously
- Desperate users also show this pattern
"""

from datetime import datetime, timedelta
from typing import Dict, Any, List
from sqlalchemy.orm import Session
from sqlalchemy import and_

from app.rules.base import Rule, RuleResult
from app.models import Transaction, User


class RapidRepeatApplicationRule(Rule):
    """
    Detects rapid repeat loan applications.

    Multiple loan applications in a short period can indicate:
    1. Loan stacking fraud (applying to many lenders at once)
    2. Credit distress (desperate borrower)
    3. Application spam

    All of these are high-risk patterns.

    Risk Indicators:
    - >5 applications in 30 days (HIGH RISK)
    - >3 applications in 7 days (HIGH RISK)
    - Multiple rejections followed by more applications
    - Applications to >5 different platforms
    - Applications within minutes/hours of each other
    """

    def __init__(self):
        super().__init__(
            rule_id="LENDING_004",
            name="Rapid Repeat Application Detection",
            description="Detects multiple loan applications in short timeframe",
            category="lending",
            severity="high"
        )

        # Configuration thresholds
        self.HIGH_RISK_APPS_30D = 5
        self.CRITICAL_APPS_7D = 3
        self.HIGH_RISK_PLATFORMS = 5
        self.RAPID_APPLICATION_HOURS = 24  # Multiple apps within 24h

    def evaluate(self, db: Session, transaction_data: Dict[str, Any]) -> RuleResult:
        """
        Evaluate rapid application risk.

        Args:
            db: Database session
            transaction_data: Transaction context including user_id

        Returns:
            RuleResult with risk score and evidence
        """
        user_id = transaction_data.get('user_id')

        if not user_id:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="No user_id provided for application check",
                evidence={}
            )

        # Fetch user
        user = db.query(User).filter(User.id == user_id).first()
        if not user:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="User not found",
                evidence={}
            )

        # Get application history
        apps_30d = self._get_applications(db, user_id, days=30)
        apps_7d = self._get_applications(db, user_id, days=7)
        apps_24h = self._get_applications(db, user_id, hours=24)

        # Analyze application patterns
        pattern_analysis = self._analyze_application_patterns(apps_30d)

        # Check rejection history
        rejection_analysis = self._analyze_rejections(apps_30d)

        # Calculate risk score
        risk_score = self._calculate_risk_score(
            len(apps_30d),
            len(apps_7d),
            len(apps_24h),
            pattern_analysis,
            rejection_analysis
        )

        # Build evidence
        evidence = {
            "applications_30d": len(apps_30d),
            "applications_7d": len(apps_7d),
            "applications_24h": len(apps_24h),
            "unique_platforms": pattern_analysis["unique_platforms"],
            "platform_list": pattern_analysis["platforms"],
            "rejection_count": rejection_analysis["rejection_count"],
            "rejection_rate": rejection_analysis["rejection_rate"],
            "rapid_fire_detected": pattern_analysis["rapid_fire"],
            "user_email": user.email,
            "check_timestamp": datetime.utcnow().isoformat()
        }

        # Determine if rule passed
        passed = risk_score < 60

        # Build message
        if risk_score >= 80:
            message = f"HIGH RISK: {len(apps_30d)} applications in 30 days across {pattern_analysis['unique_platforms']} platforms"
        elif risk_score >= 60:
            message = f"MEDIUM RISK: {len(apps_7d)} applications in 7 days"
        else:
            message = f"Low risk: {len(apps_30d)} applications in 30 days"

        return RuleResult(
            rule_id=self.rule_id,
            passed=passed,
            risk_score=risk_score,
            message=message,
            evidence=evidence
        )

    def _get_applications(
        self,
        db: Session,
        user_id: str,
        days: int = None,
        hours: int = None
    ) -> List[Transaction]:
        """
        Get loan applications within specified timeframe.

        Applications are tracked as transactions with type 'loan_application'.
        """
        if days:
            cutoff = datetime.utcnow() - timedelta(days=days)
        elif hours:
            cutoff = datetime.utcnow() - timedelta(hours=hours)
        else:
            cutoff = datetime.utcnow() - timedelta(days=30)

        applications = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user_id,
                Transaction.created_at >= cutoff,
                Transaction.metadata.contains({"transaction_type": "loan_application"})
            )
        ).order_by(Transaction.created_at.desc()).all()

        return applications

    def _analyze_application_patterns(self, applications: List[Transaction]) -> Dict[str, Any]:
        """
        Analyze application patterns for fraud indicators.

        Look for:
        - Multiple platforms
        - Rapid-fire applications (within minutes)
        - Consistent amounts (copy-paste behavior)
        """
        if not applications:
            return {
                "unique_platforms": 0,
                "platforms": [],
                "rapid_fire": False,
                "consistent_amounts": False
            }

        # Extract platforms
        platforms = []
        amounts = []
        timestamps = []

        for app in applications:
            metadata = app.metadata or {}
            platform = metadata.get("platform", "unknown")
            platforms.append(platform)
            amounts.append(abs(app.amount))
            timestamps.append(app.created_at)

        unique_platforms = len(set(platforms))

        # Check for rapid-fire (multiple apps within 1 hour)
        rapid_fire = False
        if len(timestamps) >= 2:
            timestamps.sort()
            for i in range(len(timestamps) - 1):
                time_diff = (timestamps[i+1] - timestamps[i]).total_seconds() / 3600
                if time_diff <= 1:  # Within 1 hour
                    rapid_fire = True
                    break

        # Check for consistent amounts (copy-paste behavior)
        consistent_amounts = False
        if len(amounts) >= 3:
            # If >80% of amounts are identical, flag it
            from collections import Counter
            amount_counts = Counter(amounts)
            most_common_count = amount_counts.most_common(1)[0][1]
            if most_common_count / len(amounts) >= 0.8:
                consistent_amounts = True

        return {
            "unique_platforms": unique_platforms,
            "platforms": list(set(platforms)),
            "rapid_fire": rapid_fire,
            "consistent_amounts": consistent_amounts
        }

    def _analyze_rejections(self, applications: List[Transaction]) -> Dict[str, Any]:
        """
        Analyze rejection history.

        Users who were rejected but keep applying are high risk.
        """
        if not applications:
            return {
                "rejection_count": 0,
                "rejection_rate": 0
            }

        rejections = 0
        for app in applications:
            metadata = app.metadata or {}
            if metadata.get("status") == "rejected":
                rejections += 1

        rejection_rate = rejections / len(applications) if applications else 0

        return {
            "rejection_count": rejections,
            "rejection_rate": round(rejection_rate, 2)
        }

    def _calculate_risk_score(
        self,
        apps_30d: int,
        apps_7d: int,
        apps_24h: int,
        pattern_analysis: Dict[str, Any],
        rejection_analysis: Dict[str, Any]
    ) -> int:
        """
        Calculate rapid application risk score (0-100).

        Scoring logic:
        - >5 apps in 30 days: 40 points
        - >3 apps in 7 days: 30 points
        - >2 apps in 24 hours: 20 points
        - >5 platforms: 20 points
        - Rapid-fire pattern: 15 points
        - High rejection rate: 15 points
        """
        score = 0

        # Score based on 30-day applications
        if apps_30d >= 10:
            score += 60
        elif apps_30d >= self.HIGH_RISK_APPS_30D:
            score += 40
        elif apps_30d >= 3:
            score += 20

        # Score based on 7-day applications
        if apps_7d >= self.CRITICAL_APPS_7D:
            score += 30
        elif apps_7d >= 2:
            score += 15

        # Score based on 24-hour applications
        if apps_24h >= 3:
            score += 25
        elif apps_24h >= 2:
            score += 20

        # Score based on platform diversity
        if pattern_analysis["unique_platforms"] >= self.HIGH_RISK_PLATFORMS:
            score += 20
        elif pattern_analysis["unique_platforms"] >= 3:
            score += 10

        # Score based on rapid-fire pattern
        if pattern_analysis["rapid_fire"]:
            score += 15

        # Score based on rejections
        if rejection_analysis["rejection_count"] >= 3:
            score += 20
        elif rejection_analysis["rejection_rate"] >= 0.5:
            score += 15

        # Cap at 100
        return min(score, 100)
```

### How It Works

**Scenario: Fraudster**
- 8 applications in 30 days
- 4 applications in 7 days
- 6 different platforms
- Applications within 2 hours of each other

**Risk Score**:
- 8 apps in 30d: 40 points
- 4 apps in 7d: 30 points
- 6 platforms: 20 points
- Rapid-fire: 15 points
- **Total**: 105 → capped at 100 (CRITICAL)

---

## Rule 5: Synthetic Identity Fraud

### What is Synthetic Identity Fraud?

**Definition**: Creating a "new" identity by combining real and fake information, then building credit history before busting out.

**Nigerian Pattern:**
1. Obtain valid BVN (buy on dark web, insider access, data breach)
2. Create new phone number
3. Create new email address
4. Apply for "first-time" loan
5. Build small credit history
6. Apply for maximum loans and disappear

**Why it works:**
- BVN is valid (passes verification)
- Phone/email are "real" (but newly created)
- No negative credit history
- Looks like legitimate first-time borrower

### Detection Strategy

We detect synthetic identities by:
1. **BVN age vs account age**: BVN created years ago, but user account brand new
2. **Email/phone age**: Newly created contact information
3. **Inconsistent history**: No prior financial footprint
4. **Suspicious patterns**: Perfect initial behavior, then bust out

### Implementation

**File: `/home/user/guide-sentinel/app/rules/lending/identity_fraud.py`**

```python
"""
Synthetic Identity Fraud Detection Rule

Detects synthetic identities - combinations of real and fake
information used to create fraudulent accounts.

Nigerian Context:
- BVN databases have been breached
- Fraudsters buy valid BVNs on dark web
- Combine with new phone/email to appear legitimate
- Build brief credit history then bust out
"""

from datetime import datetime, timedelta
from typing import Dict, Any
from sqlalchemy.orm import Session
from sqlalchemy import and_

from app.rules.base import Rule, RuleResult
from app.models import Transaction, User


class SyntheticIdentityRule(Rule):
    """
    Detects synthetic identity fraud patterns.

    Synthetic identity fraud involves creating a fake identity using
    a combination of real information (e.g., valid BVN) and fake
    information (new phone, email). The identity appears legitimate
    but has no real history.

    Risk Indicators:
    - BVN age >> account age (e.g., 5-year-old BVN, 1-week-old account)
    - Brand new email address
    - Brand new phone number
    - No prior transaction history
    - Perfect initial behavior followed by large loan request
    """

    def __init__(self):
        super().__init__(
            rule_id="LENDING_005",
            name="Synthetic Identity Detection",
            description="Detects synthetic identity fraud patterns",
            category="lending",
            severity="critical"
        )

        # Configuration thresholds
        self.SUSPICIOUS_BVN_ACCOUNT_GAP_DAYS = 365  # BVN >1 year older than account
        self.NEW_EMAIL_DAYS = 90  # Email created <90 days ago
        self.NEW_PHONE_DAYS = 30  # Phone <30 days old
        self.MIN_TRANSACTION_HISTORY = 5  # Expect some history

    def evaluate(self, db: Session, transaction_data: Dict[str, Any]) -> RuleResult:
        """
        Evaluate synthetic identity risk.

        Args:
            db: Database session
            transaction_data: Transaction context including user_id

        Returns:
            RuleResult with risk score and evidence
        """
        user_id = transaction_data.get('user_id')

        if not user_id:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="No user_id provided for identity check",
                evidence={}
            )

        # Fetch user
        user = db.query(User).filter(User.id == user_id).first()
        if not user:
            return RuleResult(
                rule_id=self.rule_id,
                passed=True,
                risk_score=0,
                message="User not found",
                evidence={}
            )

        # Check BVN age vs account age
        bvn_analysis = self._analyze_bvn_age(user)

        # Check email age
        email_analysis = self._analyze_email_age(user)

        # Check phone age
        phone_analysis = self._analyze_phone_age(user)

        # Check transaction history
        history_analysis = self._analyze_account_history(db, user)

        # Check for bust-out patterns
        bustout_analysis = self._detect_bustout_pattern(db, user, transaction_data)

        # Calculate risk score
        risk_score = self._calculate_risk_score(
            bvn_analysis,
            email_analysis,
            phone_analysis,
            history_analysis,
            bustout_analysis
        )

        # Build evidence
        evidence = {
            "bvn_age_days": bvn_analysis["bvn_age_days"],
            "account_age_days": bvn_analysis["account_age_days"],
            "bvn_account_gap_days": bvn_analysis["gap_days"],
            "email_age_days": email_analysis["age_days"],
            "phone_age_days": phone_analysis["age_days"],
            "transaction_count": history_analysis["transaction_count"],
            "account_activity_days": history_analysis["activity_days"],
            "bustout_indicators": bustout_analysis["indicators"],
            "user_email": user.email,
            "check_timestamp": datetime.utcnow().isoformat()
        }

        # Determine if rule passed
        passed = risk_score < 70

        # Build message
        if risk_score >= 90:
            message = f"CRITICAL: Synthetic identity suspected - BVN {bvn_analysis['gap_days']} days older than account"
        elif risk_score >= 70:
            message = f"HIGH RISK: New account with minimal history and fresh contacts"
        elif risk_score >= 40:
            message = "MEDIUM RISK: Some identity inconsistencies detected"
        else:
            message = "Low risk: Identity appears legitimate"

        return RuleResult(
            rule_id=self.rule_id,
            passed=passed,
            risk_score=risk_score,
            message=message,
            evidence=evidence
        )

    def _analyze_bvn_age(self, user: User) -> Dict[str, Any]:
        """
        Analyze BVN age vs account age.

        In production, this would query the NIBSS BVN database.
        We simulate using metadata.

        Red flag: BVN is 5 years old but user account is 2 weeks old.
        """
        metadata = user.metadata or {}

        # Get BVN creation date
        bvn_created = metadata.get("bvn_created_at")
        if bvn_created:
            bvn_date = datetime.fromisoformat(bvn_created)
            bvn_age_days = (datetime.utcnow() - bvn_date).days
        else:
            # If no BVN date, assume it's as old as the account (no gap)
            bvn_age_days = (datetime.utcnow() - user.created_at).days

        # Get account age
        account_age_days = (datetime.utcnow() - user.created_at).days

        # Calculate gap
        gap_days = bvn_age_days - account_age_days

        return {
            "bvn_age_days": bvn_age_days,
            "account_age_days": account_age_days,
            "gap_days": max(gap_days, 0)  # Can't be negative
        }

    def _analyze_email_age(self, user: User) -> Dict[str, Any]:
        """
        Analyze email age.

        In production, this would use email verification APIs or
        creation date checks. We simulate using metadata.
        """
        metadata = user.metadata or {}

        email_created = metadata.get("email_created_at")
        if email_created:
            created_date = datetime.fromisoformat(email_created)
            age_days = (datetime.utcnow() - created_date).days
        else:
            # Assume email as old as account
            age_days = (datetime.utcnow() - user.created_at).days

        return {
            "age_days": age_days,
            "is_new": age_days < self.NEW_EMAIL_DAYS
        }

    def _analyze_phone_age(self, user: User) -> Dict[str, Any]:
        """
        Analyze phone number age.

        In production, this would query telecom provider APIs.
        """
        metadata = user.metadata or {}

        phone_created = metadata.get("phone_registered_at")
        if phone_created:
            created_date = datetime.fromisoformat(phone_created)
            age_days = (datetime.utcnow() - created_date).days
        else:
            # Assume phone as old as account
            age_days = (datetime.utcnow() - user.created_at).days

        return {
            "age_days": age_days,
            "is_new": age_days < self.NEW_PHONE_DAYS
        }

    def _analyze_account_history(self, db: Session, user: User) -> Dict[str, Any]:
        """
        Analyze account transaction history.

        Synthetic identities often have minimal history.
        """
        transactions = db.query(Transaction).filter(
            and_(
                Transaction.user_id == user.id,
                Transaction.status == 'completed'
            )
        ).all()

        if not transactions:
            return {
                "transaction_count": 0,
                "activity_days": 0,
                "has_minimal_history": True
            }

        # Calculate days of activity
        earliest = min(txn.created_at for txn in transactions)
        activity_days = (datetime.utcnow() - earliest).days

        return {
            "transaction_count": len(transactions),
            "activity_days": activity_days,
            "has_minimal_history": len(transactions) < self.MIN_TRANSACTION_HISTORY
        }

    def _detect_bustout_pattern(
        self,
        db: Session,
        user: User,
        transaction_data: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Detect bust-out pattern.

        Synthetic identities often:
        1. Start with small, perfect transactions
        2. Build brief credit history
        3. Then apply for maximum loan (bust-out)
        """
        indicators = []

        # Check if this is a large first loan
        metadata = user.metadata or {}
        previous_loans = metadata.get("previous_loan_count", 0)
        requested_amount = transaction_data.get("amount", 0)

        # If first/second loan and requesting large amount, flag it
        if previous_loans <= 1 and requested_amount > 100000:  # >₦100k
            indicators.append({
                "type": "large_first_loan",
                "description": f"Large loan request (₦{requested_amount:,.0f}) with minimal history",
                "severity": "high"
            })

        # Check for perfect initial behavior (suspicious)
        account_age_days = (datetime.utcnow() - user.created_at).days
        if account_age_days < 90:  # Account <3 months old
            # Check transaction history
            transactions = db.query(Transaction).filter(
                Transaction.user_id == user.id
            ).all()

            # If very few transactions but requesting loan, flag it
            if len(transactions) < 5:
                indicators.append({
                    "type": "insufficient_history",
                    "description": f"Only {len(transactions)} transactions but requesting loan",
                    "severity": "medium"
                })

        return {
            "indicators": indicators,
            "indicator_count": len(indicators)
        }

    def _calculate_risk_score(
        self,
        bvn_analysis: Dict[str, Any],
        email_analysis: Dict[str, Any],
        phone_analysis: Dict[str, Any],
        history_analysis: Dict[str, Any],
        bustout_analysis: Dict[str, Any]
    ) -> int:
        """
        Calculate synthetic identity risk score (0-100).

        Scoring logic:
        - Large BVN-account gap: Up to 40 points
        - New email: 20 points
        - New phone: 20 points
        - Minimal transaction history: 20 points
        - Bust-out indicators: 15 points each
        """
        score = 0

        # Score BVN-account gap
        gap_days = bvn_analysis["gap_days"]
        if gap_days >= self.SUSPICIOUS_BVN_ACCOUNT_GAP_DAYS * 2:  # >2 years
            score += 40
        elif gap_days >= self.SUSPICIOUS_BVN_ACCOUNT_GAP_DAYS:  # >1 year
            score += 25
        elif gap_days >= 180:  # >6 months
            score += 15

        # Score new email
        if email_analysis["is_new"]:
            score += 20

        # Score new phone
        if phone_analysis["is_new"]:
            score += 20

        # Score minimal history
        if history_analysis["has_minimal_history"]:
            score += 20

        # Score account age (very new is suspicious)
        if history_analysis["activity_days"] < 7:
            score += 15
        elif history_analysis["activity_days"] < 30:
            score += 10

        # Score bust-out indicators
        for indicator in bustout_analysis["indicators"]:
            if indicator["severity"] == "high":
                score += 15
            elif indicator["severity"] == "medium":
                score += 10

        # Cap at 100
        return min(score, 100)
```

### How It Works

**Legitimate User:**
- BVN: 3 years old
- Account: 2 years old
- Email: 3 years old
- Phone: 2 years old
- 50+ transactions
- Risk score: 0-10 (PASS)

**Synthetic Identity:**
- BVN: 5 years old (stolen/bought)
- Account: 2 weeks old
- Email: 1 week old
- Phone: 1 week old
- 2 transactions
- Risk score: 40 + 20 + 20 + 20 + 15 = 115 → capped at 100 (CRITICAL)

---

## Testing the Rules

Now let's test all 5 rules with realistic scenarios.

### Prerequisites

Make sure your FastAPI server is running and you have some test data.

**Start the server** (if not already running):

```bash
cd /home/user/guide-sentinel
uvicorn app.main:app --reload
```

### Test Data Setup

First, create test users and transactions with the specific metadata our rules need.

**Create test user with metadata:**

```bash
curl -X POST http://localhost:8000/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "testuser@example.com",
    "metadata": {
      "phone_number": "08012345678",
      "phone_registered_at": "2023-01-15T00:00:00",
      "phone_last_changed": "2024-12-20T00:00:00",
      "bvn_created_at": "2018-05-10T00:00:00",
      "email_created_at": "2023-01-10T00:00:00",
      "previous_loan_count": 0
    }
  }'
```

This creates a user with:
- Phone from 2023 (old enough)
- Recent phone change (December 20, 2024)
- BVN from 2018 (much older than account)
- Email from 2023

**Add loan application transactions:**

```bash
# Loan application 1 - Platform A
curl -X POST http://localhost:8000/transactions \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "USER_ID_HERE",
    "amount": -50000,
    "status": "completed",
    "metadata": {
      "transaction_type": "loan_application",
      "platform": "PayLater",
      "status": "pending"
    }
  }'

# Loan application 2 - Platform B (1 hour later)
curl -X POST http://localhost:8000/transactions \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "USER_ID_HERE",
    "amount": -75000,
    "status": "completed",
    "metadata": {
      "transaction_type": "loan_application",
      "platform": "FairMoney",
      "status": "pending"
    }
  }'

# Loan disbursement 1 - Active loan
curl -X POST http://localhost:8000/transactions \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "USER_ID_HERE",
    "amount": 50000,
    "status": "completed",
    "metadata": {
      "transaction_type": "loan_disbursement",
      "platform": "PayLater",
      "repaid": false
    }
  }'
```

### Test Rule 1: Loan Stacking

**Test with API endpoint** (you'll need to create one):

For now, test directly in Python console:

```python
from sqlalchemy.orm import Session
from app.database import SessionLocal
from app.rules.lending.loan_stacking import LoanStackingRule

db = SessionLocal()
rule = LoanStackingRule()

# Test with user who has multiple loans
result = rule.evaluate(db, {"user_id": "YOUR_USER_ID"})

print(f"Rule: {result.rule_id}")
print(f"Passed: {result.passed}")
print(f"Risk Score: {result.risk_score}")
print(f"Message: {result.message}")
print(f"Evidence: {result.evidence}")
```

**Expected output for user with 2+ active loans:**

```
Rule: LENDING_001
Passed: False
Risk Score: 65
Message: MEDIUM RISK: User has 2 active loans
Evidence: {
  'active_loan_count': 2,
  'active_loan_platforms': ['PayLater', 'FairMoney'],
  'total_active_amount': 125000,
  'recent_applications_7d': 2,
  ...
}
```

### Test Rule 2: SIM Swap

```python
from app.rules.lending.sim_swap import SimSwapRule

rule = SimSwapRule()
result = rule.evaluate(db, {"user_id": "YOUR_USER_ID"})

print(f"Risk Score: {result.risk_score}")
print(f"Message: {result.message}")
print(f"Days since phone change: {result.evidence['days_since_change']}")
```

**Expected output for recent SIM swap:**

```
Risk Score: 75
Message: HIGH RISK: Recent phone number change (10 days ago)
Evidence: {
  'days_since_change': 10,
  'application_after_change_hours': 240,
  ...
}
```

### Test Rule 3: Income Mismatch

```python
from app.rules.lending.income_mismatch import IncomeMismatchRule

rule = IncomeMismatchRule()
result = rule.evaluate(db, {
    "user_id": "YOUR_USER_ID",
    "declared_monthly_income": 500000  # User claims ₦500k/month
})

print(f"Risk Score: {result.risk_score}")
print(f"Declared: ₦{result.evidence['declared_monthly_income']:,}")
print(f"Observed: ₦{result.evidence['observed_monthly_income']:,.0f}")
print(f"Ratio: {result.evidence['income_ratio']:.1f}x")
```

**Expected output for income fraud:**

```
Risk Score: 70
Declared: ₦500,000
Observed: ₦180,000
Ratio: 2.8x
Message: MEDIUM RISK: Income mismatch detected
```

### Test Rule 4: Rapid Repeat Applications

```python
from app.rules.lending.rapid_repeat import RapidRepeatApplicationRule

rule = RapidRepeatApplicationRule()
result = rule.evaluate(db, {"user_id": "YOUR_USER_ID"})

print(f"Risk Score: {result.risk_score}")
print(f"Applications (30d): {result.evidence['applications_30d']}")
print(f"Applications (7d): {result.evidence['applications_7d']}")
print(f"Platforms: {result.evidence['platform_list']}")
```

**Expected output for application spam:**

```
Risk Score: 85
Applications (30d): 8
Applications (7d): 4
Platforms: ['PayLater', 'FairMoney', 'Carbon', 'Branch']
Message: HIGH RISK: 8 applications in 30 days across 4 platforms
```

### Test Rule 5: Synthetic Identity

```python
from app.rules.lending.identity_fraud import SyntheticIdentityRule

rule = SyntheticIdentityRule()
result = rule.evaluate(db, {"user_id": "YOUR_USER_ID", "amount": 200000})

print(f"Risk Score: {result.risk_score}")
print(f"BVN age: {result.evidence['bvn_age_days']} days")
print(f"Account age: {result.evidence['account_age_days']} days")
print(f"Gap: {result.evidence['bvn_account_gap_days']} days")
```

**Expected output for synthetic identity:**

```
Risk Score: 90
BVN age: 2400 days (6.5 years)
Account age: 30 days
Gap: 2370 days
Message: CRITICAL: Synthetic identity suspected - BVN 2370 days older than account
```

### Integration Test: Run All Rules

```python
from app.rules.lending import (
    LoanStackingRule,
    SimSwapRule,
    IncomeMismatchRule,
    RapidRepeatApplicationRule,
    SyntheticIdentityRule
)

# Define all rules
rules = [
    LoanStackingRule(),
    SimSwapRule(),
    IncomeMismatchRule(),
    RapidRepeatApplicationRule(),
    SyntheticIdentityRule()
]

# Test data
transaction_data = {
    "user_id": "YOUR_USER_ID",
    "declared_monthly_income": 500000,
    "amount": 200000
}

# Run all rules
print("\n" + "="*60)
print("LENDING FRAUD DETECTION - FULL ANALYSIS")
print("="*60)

for rule in rules:
    result = rule.evaluate(db, transaction_data)

    print(f"\n[{rule.rule_id}] {rule.name}")
    print(f"  Status: {'PASS' if result.passed else 'FAIL'}")
    print(f"  Risk Score: {result.risk_score}/100")
    print(f"  Message: {result.message}")

    if result.risk_score >= 70:
        print(f"  ⚠️  HIGH RISK - Manual review required")
    elif result.risk_score >= 40:
        print(f"  ⚡ MEDIUM RISK - Additional checks recommended")

print("\n" + "="*60)
```

**Expected integrated output:**

```
============================================================
LENDING FRAUD DETECTION - FULL ANALYSIS
============================================================

[LENDING_001] Loan Stacking Detection
  Status: FAIL
  Risk Score: 65/100
  Message: MEDIUM RISK: User has 2 active loans
  ⚡ MEDIUM RISK - Additional checks recommended

[LENDING_002] SIM Swap Detection
  Status: FAIL
  Risk Score: 75/100
  Message: HIGH RISK: Recent phone number change (10 days ago)
  ⚠️  HIGH RISK - Manual review required

[LENDING_003] Income Manipulation Detection
  Status: FAIL
  Risk Score: 70/100
  Message: MEDIUM RISK: Income mismatch detected
  ⚠️  HIGH RISK - Manual review required

[LENDING_004] Rapid Repeat Application Detection
  Status: FAIL
  Risk Score: 85/100
  Message: HIGH RISK: 8 applications in 30 days across 4 platforms
  ⚠️  HIGH RISK - Manual review required

[LENDING_005] Synthetic Identity Detection
  Status: FAIL
  Risk Score: 90/100
  Message: CRITICAL: Synthetic identity suspected
  ⚠️  HIGH RISK - Manual review required

============================================================
```

---

## What We Built Today

### 5 Production-Ready Lending Rules

**Rule 1: Loan Stacking Detection (LENDING_001)**
- Detects multiple simultaneous loans across platforms
- Uses consortium intelligence simulation
- Scores based on active loans, recent applications, and platform diversity
- Critical for Nigerian lending where users can access 50+ lenders

**Rule 2: SIM Swap Detection (LENDING_002)**
- Identifies recent SIM card changes
- Checks phone number age and change history
- Flags suspicious timing (loan app right after swap)
- Addresses major Nigerian fraud vector

**Rule 3: Income Manipulation Detection (LENDING_003)**
- Compares declared vs observed income
- Analyzes spending patterns for consistency
- Detects income inflation fraud
- Handles Nigeria's income verification challenges

**Rule 4: Rapid Repeat Application Detection (LENDING_004)**
- Identifies application spam patterns
- Tracks velocity across multiple platforms
- Distinguishes fraud from credit distress
- Prevents loan stacking before it happens

**Rule 5: Synthetic Identity Detection (LENDING_005)**
- Detects fake identities using real BVNs
- Checks consistency of identity elements
- Identifies bust-out patterns
- Critical for preventing large-scale fraud

### Key Concepts Learned

**1. Nigerian Lending Fraud Landscape**
- Multiple platform exploitation
- Easy SIM replacement vulnerability
- BVN as identity anchor (and attack vector)
- Income verification challenges
- Lack of real-time credit bureau

**2. Risk Scoring Methodology**
- 0-100 scale for consistency
- Additive scoring with caps
- Multiple signals combined
- Clear thresholds (60 = medium, 80 = high)

**3. Evidence-Based Detection**
- Every decision backed by data
- Detailed evidence for manual review
- Transparent scoring logic
- Audit trail for compliance

**4. Consortium Intelligence**
- Simulated cross-platform data sharing
- Real-world would use APIs/data exchanges
- Critical for detecting loan stacking
- Privacy considerations

### Architecture Patterns

**1. Rule Inheritance**
- All rules extend `Rule` base class
- Consistent interface for evaluation
- Standardized result format
- Easy integration with rules engine

**2. Metadata-Driven Detection**
- User and transaction metadata stores fraud signals
- Flexible schema for different data types
- Production would integrate with external APIs

**3. Configurable Thresholds**
- All magic numbers as class constants
- Easy to tune for different risk appetites
- A/B testing friendly

---

## Real-World Deployment Considerations

### For Production Use

**1. External Data Integration**

Our rules simulate consortium data. In production:

```python
# Instead of simulating, call real APIs
def _get_active_loans(self, db: Session, user_id: str):
    # Call credit bureau API
    response = requests.get(
        f"https://creditbureau.ng/api/v1/loans/{user_id}",
        headers={"Authorization": f"Bearer {API_KEY}"}
    )
    return response.json()
```

**2. BVN Verification**

Integrate with NIBSS (Nigerian Interbank Settlement System):

```python
def _verify_bvn(self, bvn: str):
    response = requests.post(
        "https://nibss.ng/api/bvn/verify",
        json={"bvn": bvn},
        headers={"Authorization": f"Bearer {NIBSS_API_KEY}"}
    )
    return response.json()
```

**3. Phone Number Intelligence**

Integrate with telecom providers (MTN, Airtel, Glo, 9mobile):

```python
def _check_phone_age(self, phone: str):
    # MTN API example
    response = requests.get(
        f"https://api.mtn.ng/subscriber/age/{phone}",
        headers={"X-API-Key": MTN_API_KEY}
    )
    return response.json()
```

**4. Email Verification**

Use email intelligence services:

```python
def _check_email_age(self, email: str):
    # EmailRep API
    response = requests.get(
        f"https://emailrep.io/{email}",
        headers={"Key": EMAILREP_API_KEY}
    )
    return response.json()
```

### Performance Optimization

**1. Caching**

Cache expensive API calls:

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def _get_bvn_data(self, bvn: str):
    # BVN data doesn't change often
    return self._call_bvn_api(bvn)
```

**2. Async Processing**

For batch processing, use async:

```python
async def evaluate_async(self, db: Session, transaction_data: Dict):
    # Run multiple checks in parallel
    results = await asyncio.gather(
        self._check_bvn_async(user),
        self._check_phone_async(user),
        self._check_email_async(user)
    )
    return self._calculate_risk(results)
```

**3. Database Indexing**

Ensure fast queries:

```sql
CREATE INDEX idx_transactions_user_created
ON transactions(user_id, created_at);

CREATE INDEX idx_transactions_metadata_type
ON transactions USING GIN(metadata);
```

### Regulatory Compliance

**1. Data Privacy (NDPR - Nigeria Data Protection Regulation)**

```python
# Log access to sensitive data
def _access_bvn(self, user_id: str):
    audit_log.info(
        "BVN_ACCESS",
        user_id=user_id,
        accessed_by=current_user.id,
        purpose="fraud_detection",
        timestamp=datetime.utcnow()
    )
```

**2. Explainability**

Provide clear explanations for decisions:

```python
evidence = {
    "decision": "REJECT",
    "primary_reason": "Multiple active loans detected",
    "supporting_factors": [
        "4 active loans across 3 platforms",
        "Recent SIM swap (5 days ago)",
        "Income mismatch (declared 2.5x observed)"
    ],
    "customer_message": "We need additional verification..."
}
```

---

## Next Steps

### Day 13: More Lending Rules (Part 2)

Tomorrow we'll implement 5 more advanced rules:
- Device fingerprinting
- Geolocation analysis
- Social network fraud
- Repayment behavior patterns
- First-party fraud detection

### Day 14: Rules Orchestration

- Priority-based rule execution
- Rule dependencies
- Performance optimization
- Batch processing

### Immediate Practice

**1. Customize Thresholds**

Adjust risk thresholds for your risk appetite:

```python
# In each rule __init__
self.HIGH_RISK_LOAN_COUNT = 5  # More lenient
self.CRITICAL_WINDOW_DAYS = 3  # More strict
```

**2. Add More Signals**

Extend rules with additional checks:

```python
def _check_social_media_age(self, user: User):
    # Check Facebook/Instagram account age
    # Newly created social accounts = suspicious
    pass
```

**3. Test Edge Cases**

- What about users with no transaction history?
- How do rules handle missing metadata?
- What if external APIs are down?

**4. Build Dashboard**

Create a simple dashboard to visualize rule results:

```python
@app.get("/fraud-dashboard/{user_id}")
def get_fraud_dashboard(user_id: str):
    # Run all rules, return summary
    return {
        "user_id": user_id,
        "overall_risk": "HIGH",
        "risk_score": 85,
        "triggered_rules": [...],
        "recommendations": [...]
    }
```

---

## Key Takeaways

### What Makes These Rules Effective

**1. Nigerian Context**

Every rule is designed for Nigeria's specific fraud landscape:
- BVN centrality
- Telecom vulnerabilities
- Multiple platform access
- Cash economy challenges

**2. Layered Defense**

No single rule catches everything:
- Loan stacking catches multi-platform fraud
- SIM swap catches account takeover
- Income rules catch financial fraud
- Application velocity catches spam
- Identity rules catch sophisticated fraud

**3. Actionable Results**

Rules provide:
- Clear risk scores
- Detailed evidence
- Specific recommendations
- Audit trails

### Production Readiness Checklist

Before deploying to production:

- [ ] Integrate real BVN verification API
- [ ] Connect to credit bureau/consortium
- [ ] Add telecom provider APIs
- [ ] Implement email/phone intelligence
- [ ] Set up comprehensive logging
- [ ] Create manual review workflow
- [ ] Build fraud analyst dashboard
- [ ] Implement A/B testing for thresholds
- [ ] Add performance monitoring
- [ ] Create incident response playbook
- [ ] Document all decision logic
- [ ] Train fraud team on rules

---

## Summary

Today we built **5 production-ready lending fraud detection rules** specifically designed for the Nigerian fintech ecosystem. These rules detect:

1. **Loan stacking** - Multiple simultaneous loans
2. **SIM swap fraud** - Account takeover via phone number
3. **Income manipulation** - Inflated salary declarations
4. **Application spam** - Rapid repeat applications
5. **Synthetic identities** - Fake accounts using real data

Each rule:
- Returns a 0-100 risk score
- Provides detailed evidence
- Integrates with our rules engine from Day 4
- Handles Nigerian-specific fraud patterns

Tomorrow we'll build 5 more advanced rules to create a comprehensive lending fraud defense system!

**Total lines**: ~1,850 (including comprehensive explanations and code)

---

## Additional Resources

**Nigerian Fintech Fraud Research**:
- NIBSS Fraud Reports: https://nibss-plc.com.ng
- CBN Guidelines on Digital Lending
- EFInA Access to Financial Services Survey

**Technical References**:
- FICO Falcon Fraud Platform documentation
- SAS Fraud Detection best practices
- ACFE Fraud Examination Manual

**Regulatory**:
- Nigeria Data Protection Regulation (NDPR)
- CBN Consumer Protection Framework
- Guidelines on Instant Digital Lending

---

*End of Day 12*
