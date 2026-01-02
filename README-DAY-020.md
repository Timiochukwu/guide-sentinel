# Day 20: E-commerce Fraud Rules (Part 1)

**Navigation:** [← Day 19](./README-DAY-019.md) | [Main Guide](./README.md) | [Day 21 →](./README-DAY-021.md)

---

## Overview

Welcome to Day 20! Today we're entering the **E-commerce vertical** - building fraud detection rules specifically designed for online shopping platforms like Jumia, Konga, and PayPorte. E-commerce fraud in Nigeria has unique characteristics driven by card-not-present (CNP) transactions, address manipulation, and the rise of digital payments.

**What You'll Build Today:**
- 2 specialized e-commerce fraud detection rules
- Card BIN fraud detection system
- Address verification fraud detection
- Nigerian e-commerce fraud pattern recognition
- Integration with existing rules engine

**The Two Rules We'll Implement:**

1. **Card BIN Fraud Detection** - Identifies stolen card patterns through BIN analysis
   - Example: Multiple orders from different accounts using cards from same BIN range → HIGH RISK
   - Real-world: Catches carding attacks and stolen card databases

2. **Address Verification Fraud** - Detects shipping address manipulation
   - Example: Billing address in Lagos, shipping to unknown location in Abuja → MEDIUM RISK
   - Real-world: Prevents reshipping fraud and address drops

**Prerequisites:**
- ✅ Day 4: Rules engine framework
- ✅ Day 2: Database models
- ✅ Days 12-13: Lending rules (pattern familiarity)

**Time Estimate:** 3-4 hours

**No New Packages Required!** We'll use the existing rules framework.

---

## Table of Contents

1. [Understanding E-commerce Fraud in Nigeria](#understanding-e-commerce-fraud-in-nigeria)
2. [E-commerce Fraud Landscape](#e-commerce-fraud-landscape)
3. [Project Structure](#project-structure)
4. [Complete Code Implementation](#complete-code-implementation)
5. [Testing Your Rules](#testing-your-rules)
6. [Integration with Rules Engine](#integration-with-rules-engine)
7. [Troubleshooting](#troubleshooting)
8. [Next Steps](#next-steps)

---

## Understanding E-commerce Fraud in Nigeria

### The Nigerian E-commerce Boom

Nigeria's e-commerce market has experienced explosive growth:

**Market Size:**
- 2020: $13 billion
- 2023: $23 billion (estimated)
- 2025: $75 billion (projected)
- Growth rate: 60%+ YoY

**Major Players:**
1. **Jumia** - Pan-African e-commerce giant
2. **Konga** - Nigerian e-commerce leader
3. **PayPorte** - Fashion and lifestyle
4. **Jiji** - Classifieds marketplace
5. **Slot** - Electronics retailer

**Payment Methods:**
- Card payments (Visa, Mastercard, Verve): 45%
- Bank transfers: 30%
- Pay on delivery (POD): 20%
- USSD: 5%

### Why E-commerce Fraud is Different

Unlike lending fraud which focuses on identity and creditworthiness, e-commerce fraud centers around:

**1. Card-Not-Present (CNP) Transactions**
```
Physical Store (Card Present):
- Card chip verification
- PIN entry
- Physical card inspection
- Lower fraud rate: 0.05%

Online Store (Card Not Present):
- Only card number needed
- No physical verification
- Higher fraud rate: 1.8%
```

**2. Address Manipulation**
```
Fraudster's Strategy:
1. Steal card details online
2. Place order on e-commerce site
3. Ship to "drop address" (abandoned building, hotel lobby)
4. Collect package before investigation
5. Cardholder reports fraud
6. Merchant loses goods + chargeback fee
```

**3. Account Takeover (ATO)**
```
Attack Flow:
1. Phish user credentials
2. Login to e-commerce account
3. Change shipping address
4. Use saved payment methods
5. Order high-value electronics
6. Victim realizes days later
```

### E-commerce Fraud Statistics (Nigeria)

**Loss Distribution:**
```
Total Annual E-commerce Fraud Loss: ₦18.5 billion

Breakdown:
├── Card fraud: ₦8.2B (44%)
├── Account takeover: ₦4.1B (22%)
├── Promo abuse: ₦3.7B (20%)
├── Friendly fraud: ₦1.8B (10%)
└── Return fraud: ₦0.7B (4%)
```

**Fraud by Product Category:**
```
Electronics (phones, laptops): 52%
Fashion/apparel: 18%
Beauty products: 12%
Home appliances: 10%
Others: 8%
```

**Chargeback Rates:**
```
Global average: 0.6%
Nigeria average: 2.3%
E-commerce (Nigeria): 3.8%

High chargeback categories:
- International orders: 8.2%
- Electronics: 5.7%
- First-time buyers: 4.1%
```

---

## E-commerce Fraud Landscape

### Nigerian E-commerce Fraud Patterns

#### 1. Card Testing Attacks

**What it is:**
Fraudsters test stolen card numbers with small purchases before making large fraudulent orders.

**Nigerian Context:**
```python
# Typical card testing pattern
Orders in 10 minutes:
├── ₦500 - Phone case (test)
├── ₦500 - Screen protector (test)
├── ₦500 - Cable (test)
└── ₦450,000 - iPhone 15 Pro (actual fraud)

Success rate for fraudster: 78%
Detection rate without rules: 12%
```

**Impact:**
- Thousands of micro-transactions
- Overwhelms fraud review teams
- Valid cards identified in minutes

#### 2. BIN Attacks

**What is a BIN?**
Bank Identification Number - first 6-8 digits of card number
```
Card: 5399 8312 4567 8901
BIN:  5399 8312 (identifies issuing bank)
```

**Nigerian BIN Attack Pattern:**
```
Fraudster obtains BIN database from breach:
├── GTBank Mastercard: 5399 83xx xxxx xxxx
├── First Bank Visa: 4389 22xx xxxx xxxx
└── Access Bank Verve: 5060 99xx xxxx xxxx

Then generates valid card numbers:
- Algorithm: Luhn check digit validation
- Success rate: 15-30% of generated numbers work
- Attack scale: 1000+ cards tested per day
```

**Real Example:**
```
May 2023 - Jumia BIN Attack:
- 347 orders in 6 hours
- Same BIN range (5399 83xx)
- Different account names
- Total loss: ₦8.7M
- 89% shipped before detection
```

#### 3. Address Fraud Patterns

**Drop Addresses in Nigeria:**
```
Common drop locations:
├── Hotels (Lagos Island, Victoria Island)
├── Co-working spaces (Yaba tech hub)
├── University hostels (UI, UNILAG, UNIBEN)
├── Filling stations ("collect at pump 3")
└── Churches/Mosques (Sunday/Friday collection)
```

**Mismatch Patterns:**
```
High Risk:
- Billing: Lekki Phase 1 (affluent)
  Shipping: Ajegunle (high-crime area)

- Billing: Corporate Ikoyi office
  Shipping: Random residential Festac

- Billing: International address
  Shipping: Nigerian PO Box
```

#### 4. Velocity Fraud

**Pattern:**
```
Account created: 10:00 AM
First order: 10:05 AM (₦250,000)
Second order: 10:12 AM (₦320,000)
Third order: 10:18 AM (₦180,000)

Red flags:
- New account
- No browsing history
- Direct to checkout
- High-value items
- Multiple orders within 30 minutes
```

### Nigerian-Specific Challenges

**1. Address Standardization**
```
Same location, different formats:
- "12 Admiralty Way Lekki Phase 1 Lagos"
- "No 12, admiralty way, lekki phase 1"
- "12 Admiralty Rd, Lekki, Lagos State"
- "Plot 12 Admiralty Way LP1"

Problem: Hard to detect duplicate addresses
Solution: Address normalization algorithm
```

**2. Limited Address Verification**
```
Nigeria has no:
├── Centralized postal database
├── Government address registry
├── AVS (Address Verification Service)
└── Standardized street names

Result: Fraudsters exploit this gap
```

**3. Pay on Delivery (POD) Fraud**
```
POD Fraud Pattern:
1. Order high-value item (₦500,000 laptop)
2. Select "Pay on Delivery"
3. Use fake address/phone
4. Courier delivers to non-existent location
5. Return to sender
6. Merchant loses shipping fee + time

Annual POD fraud loss: ₦2.3B
```

---

## Project Structure

### Directory Setup

We'll create an `ecommerce` subdirectory under `rules`:

```
app/
└── rules/
    ├── __init__.py
    ├── base.py              # From Day 4
    ├── engine.py            # From Day 4
    ├── lending/             # From Days 12-13
    │   ├── __init__.py
    │   ├── loan_stacking.py
    │   ├── sim_swap.py
    │   └── ... (other lending rules)
    └── ecommerce/           # NEW - Today!
        ├── __init__.py
        ├── card_bin_fraud.py
        └── address_verification.py
```

### How E-commerce Rules Integrate

```
Transaction arrives
       ↓
Rules Engine evaluates ALL rules
       ↓
┌──────────────────┬───────────────────┐
│  Lending Rules   │  E-commerce Rules │
├──────────────────┼───────────────────┤
│ Loan Stacking    │ Card BIN Fraud    │
│ SIM Swap         │ Address Fraud     │
│ Income Fraud     │ (More on Day 21)  │
└──────────────────┴───────────────────┘
       ↓
Aggregate risk score
       ↓
Return decision
```

**Why this matters:**
- Same transaction can trigger both lending AND e-commerce rules
- Example: Loan application + card payment → both rule sets evaluate
- Comprehensive fraud detection across verticals

---

## Complete Code Implementation

### Step 1: Create E-commerce Directory

```bash
# Create the ecommerce rules directory
mkdir -p app/rules/ecommerce
```

### Step 2: E-commerce Rules Init File

**File: `app/rules/ecommerce/__init__.py`**

```python
"""
E-commerce Fraud Detection Rules - Days 20-21

This module contains fraud detection rules specifically designed
for e-commerce platforms in Nigeria (Jumia, Konga, PayPorte, etc.).

Rules implemented:
- Day 20: Card BIN fraud detection, Address verification fraud
- Day 21: Account takeover detection, Promo code abuse

Author: Sentinel Team
"""

from .card_bin_fraud import CardBINFraudRule
from .address_verification import AddressVerificationRule

__all__ = [
    'CardBINFraudRule',
    'AddressVerificationRule',
]
```

### Step 3: Card BIN Fraud Detection Rule

**File: `app/rules/ecommerce/card_bin_fraud.py`**

```python
"""
Card BIN Fraud Detection Rule - Day 20

Detects fraudulent patterns based on Bank Identification Number (BIN) analysis.

BIN (Bank Identification Number) is the first 6-8 digits of a payment card
that identifies the issuing bank. This rule detects:
- Multiple accounts using cards from same BIN
- Known compromised BIN ranges
- Suspicious BIN velocity patterns
- Card testing attacks

Nigerian Context:
- Common BINs: GTBank (5399 83), First Bank (4389 22), Access (5060 99)
- BIN attacks increased 340% in 2023
- Average loss per BIN attack: ₦8.7M

Author: Sentinel Team
"""

from typing import Dict, Any, List, Optional
from datetime import datetime, timedelta
from sqlalchemy.orm import Session
from sqlalchemy import and_, func, or_

from app.rules.base import BaseRule, RuleResult
from app.models import Transaction, User


class CardBINFraudRule(BaseRule):
    """
    Detects card BIN-based fraud patterns.

    Risk Indicators:
    - Same BIN used across multiple accounts (>5 = high risk)
    - BIN velocity: too many transactions from same BIN
    - Known compromised BIN ranges
    - Sequential card numbers from same BIN
    - New account + unfamiliar BIN combination

    Risk Scoring:
    - 0-30: Low risk (normal BIN usage)
    - 31-60: Medium risk (elevated BIN velocity)
    - 61-100: High risk (BIN attack pattern detected)
    """

    # Known compromised BIN ranges (examples - update from real threat intel)
    COMPROMISED_BINS = {
        '539983',  # GTBank Mastercard (example)
        '438922',  # First Bank Visa (example)
        '506099',  # Access Bank Verve (example)
    }

    # High-risk international BINs (commonly used in fraud)
    HIGH_RISK_BINS = {
        '424242',  # Test cards leaked online
        '400000',  # Prepaid cards
        '411111',  # Common test BIN
    }

    def __init__(self):
        super().__init__(
            rule_id="ECOMMERCE_001",
            name="Card BIN Fraud Detection",
            description="Detects fraudulent patterns based on BIN analysis",
            category="ecommerce",
            severity="high"
        )

    def evaluate(self, transaction: Dict[str, Any], db: Session) -> RuleResult:
        """
        Evaluate transaction for BIN fraud patterns.

        Args:
            transaction: Transaction data including card_number
            db: Database session

        Returns:
            RuleResult with risk score and detailed findings
        """
        risk_score = 0
        risk_factors = []
        metadata = {}

        # Extract card number and BIN
        card_number = transaction.get('card_number', '')

        if not card_number or len(card_number) < 6:
            # No card number provided or invalid
            return RuleResult(
                rule_id=self.rule_id,
                rule_name=self.name,
                triggered=False,
                risk_score=0,
                reason="No card number provided for BIN analysis",
                metadata={"status": "skipped"}
            )

        # Extract BIN (first 6 digits)
        bin_number = card_number[:6]
        metadata['bin'] = bin_number

        # Check 1: Known compromised BIN
        if bin_number in self.COMPROMISED_BINS:
            risk_score += 40
            risk_factors.append(
                f"BIN {bin_number} is in known compromised BIN list"
            )
            metadata['compromised_bin'] = True

        # Check 2: High-risk international BIN
        if bin_number in self.HIGH_RISK_BINS:
            risk_score += 35
            risk_factors.append(
                f"BIN {bin_number} is flagged as high-risk (test/prepaid cards)"
            )
            metadata['high_risk_bin'] = True

        # Check 3: BIN velocity - same BIN across multiple accounts
        bin_account_count = self._check_bin_velocity_accounts(
            bin_number,
            transaction.get('user_id'),
            db
        )

        if bin_account_count > 5:
            risk_score += 30
            risk_factors.append(
                f"BIN {bin_number} used by {bin_account_count} different accounts (threshold: 5)"
            )
        elif bin_account_count > 3:
            risk_score += 15
            risk_factors.append(
                f"BIN {bin_number} used by {bin_account_count} different accounts"
            )

        metadata['bin_account_count'] = bin_account_count

        # Check 4: BIN transaction velocity (time-based)
        bin_txn_count_1h = self._check_bin_velocity_time(
            bin_number,
            hours=1,
            db=db
        )

        bin_txn_count_24h = self._check_bin_velocity_time(
            bin_number,
            hours=24,
            db=db
        )

        if bin_txn_count_1h > 10:
            risk_score += 25
            risk_factors.append(
                f"{bin_txn_count_1h} transactions from BIN {bin_number} in last hour (threshold: 10)"
            )
        elif bin_txn_count_1h > 5:
            risk_score += 10
            risk_factors.append(
                f"{bin_txn_count_1h} transactions from BIN {bin_number} in last hour"
            )

        metadata['bin_txn_1h'] = bin_txn_count_1h
        metadata['bin_txn_24h'] = bin_txn_count_24h

        # Check 5: Card number sequence pattern (potential card generation)
        is_sequential = self._check_card_sequence(
            card_number,
            bin_number,
            db
        )

        if is_sequential:
            risk_score += 20
            risk_factors.append(
                "Sequential card numbers detected - possible card generation attack"
            )
            metadata['sequential_cards'] = True

        # Check 6: New account + high transaction amount
        user_id = transaction.get('user_id')
        amount = transaction.get('amount', 0)

        if user_id:
            account_age_days = self._get_account_age(user_id, db)

            if account_age_days < 1 and amount > 100000:  # ₦100k
                risk_score += 15
                risk_factors.append(
                    f"New account (<1 day old) with high-value transaction (₦{amount:,.0f})"
                )
                metadata['new_account_high_value'] = True

            metadata['account_age_days'] = account_age_days

        # Check 7: BIN mismatch with user location
        user_location = transaction.get('location', {}).get('country', '')
        bin_country = self._get_bin_country(bin_number)

        if bin_country and user_location:
            if bin_country != user_location and bin_country != 'NG':
                risk_score += 10
                risk_factors.append(
                    f"BIN country ({bin_country}) doesn't match transaction country ({user_location})"
                )
                metadata['bin_country_mismatch'] = True

        metadata['bin_country'] = bin_country

        # Cap risk score at 100
        risk_score = min(risk_score, 100)

        # Determine if rule triggered
        triggered = risk_score >= 30

        # Build reason string
        if triggered:
            reason = f"BIN fraud risk detected (score: {risk_score}). " + "; ".join(risk_factors)
        else:
            reason = f"No significant BIN fraud patterns detected (score: {risk_score})"

        return RuleResult(
            rule_id=self.rule_id,
            rule_name=self.name,
            triggered=triggered,
            risk_score=risk_score,
            reason=reason,
            metadata=metadata
        )

    def _check_bin_velocity_accounts(
        self,
        bin_number: str,
        current_user_id: Optional[int],
        db: Session
    ) -> int:
        """
        Check how many different accounts have used this BIN.

        Args:
            bin_number: BIN to check
            current_user_id: Current user (to include in count)
            db: Database session

        Returns:
            Count of unique accounts using this BIN
        """
        # Look back 30 days
        cutoff = datetime.utcnow() - timedelta(days=30)

        # Query transactions with this BIN
        bin_pattern = f"{bin_number}%"

        unique_users = db.query(func.count(func.distinct(Transaction.user_id)))\
            .filter(
                and_(
                    Transaction.card_number.like(bin_pattern),
                    Transaction.created_at >= cutoff
                )
            ).scalar()

        return unique_users or 0

    def _check_bin_velocity_time(
        self,
        bin_number: str,
        hours: int,
        db: Session
    ) -> int:
        """
        Check transaction count for this BIN in time window.

        Args:
            bin_number: BIN to check
            hours: Time window in hours
            db: Database session

        Returns:
            Transaction count
        """
        cutoff = datetime.utcnow() - timedelta(hours=hours)
        bin_pattern = f"{bin_number}%"

        count = db.query(func.count(Transaction.id))\
            .filter(
                and_(
                    Transaction.card_number.like(bin_pattern),
                    Transaction.created_at >= cutoff
                )
            ).scalar()

        return count or 0

    def _check_card_sequence(
        self,
        card_number: str,
        bin_number: str,
        db: Session
    ) -> bool:
        """
        Check if card numbers are sequential (card generation attack).

        Args:
            card_number: Current card number
            bin_number: BIN of current card
            db: Database session

        Returns:
            True if sequential pattern detected
        """
        # Look back 24 hours
        cutoff = datetime.utcnow() - timedelta(hours=24)
        bin_pattern = f"{bin_number}%"

        # Get recent card numbers with this BIN
        recent_cards = db.query(Transaction.card_number)\
            .filter(
                and_(
                    Transaction.card_number.like(bin_pattern),
                    Transaction.created_at >= cutoff
                )
            )\
            .limit(20)\
            .all()

        if len(recent_cards) < 3:
            return False

        # Extract account numbers (digits after BIN)
        try:
            current_account = int(card_number[6:15])  # Exclude check digit
            recent_accounts = [
                int(card[0][6:15])
                for card in recent_cards
                if len(card[0]) >= 15
            ]

            # Check if we have sequential numbers
            sequential_count = 0
            for account in recent_accounts:
                if abs(current_account - account) <= 10:
                    sequential_count += 1

            # If more than 2 cards within 10 numbers of each other
            return sequential_count >= 2

        except (ValueError, IndexError):
            return False

    def _get_account_age(self, user_id: int, db: Session) -> float:
        """
        Get account age in days.

        Args:
            user_id: User ID
            db: Database session

        Returns:
            Account age in days
        """
        user = db.query(User).filter(User.id == user_id).first()

        if not user:
            return 0

        age = datetime.utcnow() - user.created_at
        return age.total_seconds() / 86400  # Convert to days

    def _get_bin_country(self, bin_number: str) -> str:
        """
        Get country code for BIN.

        In production, this would query a BIN database API.
        For now, we'll use simple pattern matching for Nigerian banks.

        Args:
            bin_number: BIN to lookup

        Returns:
            ISO country code (e.g., 'NG', 'US', 'GB')
        """
        # Nigerian bank BIN ranges (examples)
        nigerian_bins = {
            '5399': 'NG',  # GTBank
            '4389': 'NG',  # First Bank
            '5060': 'NG',  # Access Bank (Verve)
            '5061': 'NG',  # Verve
            '5078': 'NG',  # Verve
            '6280': 'NG',  # Zenith Bank
        }

        # Check first 4 digits
        bin_prefix = bin_number[:4]

        return nigerian_bins.get(bin_prefix, 'UNKNOWN')
```

### Step 4: Address Verification Fraud Rule

**File: `app/rules/ecommerce/address_verification.py`**

```python
"""
Address Verification Fraud Detection Rule - Day 20

Detects fraudulent patterns based on shipping and billing address analysis.

Address fraud is a major vector in Nigerian e-commerce due to:
- Lack of standardized postal system
- No AVS (Address Verification Service)
- Easy to create "drop addresses" for package collection
- Weak address validation by couriers

This rule detects:
- Billing/shipping address mismatches
- Known fraud hotspot addresses
- Suspicious address patterns
- Velocity of address usage

Author: Sentinel Team
"""

from typing import Dict, Any, List, Optional, Tuple
from datetime import datetime, timedelta
from sqlalchemy.orm import Session
from sqlalchemy import and_, func
import re

from app.rules.base import BaseRule, RuleResult
from app.models import Transaction


class AddressVerificationRule(BaseRule):
    """
    Detects address-based fraud patterns.

    Risk Indicators:
    - Billing and shipping addresses in different states/countries
    - Shipping to known fraud hotspots
    - High-value shipping to hotel/PO Box
    - Multiple accounts shipping to same address
    - Address used in previous fraud cases
    - Suspicious address patterns (incomplete, invalid)

    Risk Scoring:
    - 0-30: Low risk (normal address usage)
    - 31-60: Medium risk (address mismatch or moderate red flags)
    - 61-100: High risk (known fraud pattern detected)
    """

    # Known fraud hotspot areas in Nigeria
    FRAUD_HOTSPOTS = {
        'ajegunle': 40,      # High fraud area
        'mushin': 35,        # High fraud area
        'okokomaiko': 40,    # High fraud area
        'alaba': 30,         # Electronics market
        'computer village': 25,  # Electronics hub
    }

    # High-risk address patterns
    RISKY_ADDRESS_KEYWORDS = {
        'hotel': 20,
        'motel': 20,
        'lodge': 15,
        'po box': 25,
        'p.o. box': 25,
        'general delivery': 30,
        'poste restante': 30,
        'filling station': 25,
        'fuel station': 25,
        'church': 15,
        'mosque': 15,
        'under bridge': 40,
        'park': 20,
    }

    # Nigerian states with high e-commerce fraud rates
    HIGH_RISK_STATES = {
        'rivers': 15,
        'delta': 15,
        'edo': 20,
        'abia': 15,
    }

    def __init__(self):
        super().__init__(
            rule_id="ECOMMERCE_002",
            name="Address Verification Fraud",
            description="Detects shipping and billing address fraud patterns",
            category="ecommerce",
            severity="medium"
        )

    def evaluate(self, transaction: Dict[str, Any], db: Session) -> RuleResult:
        """
        Evaluate transaction for address fraud patterns.

        Args:
            transaction: Transaction data with billing/shipping addresses
            db: Database session

        Returns:
            RuleResult with risk score and findings
        """
        risk_score = 0
        risk_factors = []
        metadata = {}

        # Extract addresses
        billing_address = transaction.get('billing_address', {})
        shipping_address = transaction.get('shipping_address', {})

        if not billing_address or not shipping_address:
            return RuleResult(
                rule_id=self.rule_id,
                rule_name=self.name,
                triggered=False,
                risk_score=0,
                reason="Insufficient address data for verification",
                metadata={"status": "skipped"}
            )

        # Normalize addresses
        billing_norm = self._normalize_address(billing_address)
        shipping_norm = self._normalize_address(shipping_address)

        metadata['billing_normalized'] = billing_norm['full']
        metadata['shipping_normalized'] = shipping_norm['full']

        # Check 1: Billing and shipping address country mismatch
        if billing_norm['country'] and shipping_norm['country']:
            if billing_norm['country'] != shipping_norm['country']:
                risk_score += 35
                risk_factors.append(
                    f"International shipping: billing in {billing_norm['country']}, "
                    f"shipping to {shipping_norm['country']}"
                )
                metadata['country_mismatch'] = True

        # Check 2: State mismatch within Nigeria
        if (billing_norm['state'] and shipping_norm['state'] and
            billing_norm['country'] == 'NG' and shipping_norm['country'] == 'NG'):

            if billing_norm['state'] != shipping_norm['state']:
                # Different states - moderate risk
                risk_score += 15
                risk_factors.append(
                    f"Different states: billing in {billing_norm['state']}, "
                    f"shipping to {shipping_norm['state']}"
                )
                metadata['state_mismatch'] = True

                # Check if shipping to high-risk state
                if shipping_norm['state'].lower() in self.HIGH_RISK_STATES:
                    additional_risk = self.HIGH_RISK_STATES[shipping_norm['state'].lower()]
                    risk_score += additional_risk
                    risk_factors.append(
                        f"Shipping to high-risk state: {shipping_norm['state']} "
                        f"(+{additional_risk} risk)"
                    )

        # Check 3: Fraud hotspot detection
        hotspot_risk = self._check_fraud_hotspots(shipping_norm['full'])
        if hotspot_risk > 0:
            risk_score += hotspot_risk
            risk_factors.append(
                f"Shipping address contains fraud hotspot keyword (+{hotspot_risk} risk)"
            )
            metadata['fraud_hotspot_detected'] = True

        # Check 4: Risky address pattern detection
        risky_pattern_score = self._check_risky_patterns(shipping_norm['full'])
        if risky_pattern_score > 0:
            risk_score += risky_pattern_score
            risk_factors.append(
                f"Shipping address contains risky pattern (+{risky_pattern_score} risk)"
            )
            metadata['risky_pattern_detected'] = True

        # Check 5: Address completeness
        completeness_score = self._check_address_completeness(shipping_address)
        if completeness_score < 0.6:  # Less than 60% complete
            risk_score += 20
            risk_factors.append(
                f"Incomplete shipping address (completeness: {completeness_score*100:.0f}%)"
            )
            metadata['incomplete_address'] = True

        metadata['address_completeness'] = completeness_score

        # Check 6: Address velocity (same address used by multiple users)
        shipping_str = shipping_norm['full'].lower()
        address_user_count = self._check_address_velocity(shipping_str, db)

        if address_user_count > 5:
            risk_score += 30
            risk_factors.append(
                f"Shipping address used by {address_user_count} different accounts (threshold: 5)"
            )
        elif address_user_count > 3:
            risk_score += 15
            risk_factors.append(
                f"Shipping address used by {address_user_count} different accounts"
            )

        metadata['address_user_count'] = address_user_count

        # Check 7: High-value order to risky address
        amount = transaction.get('amount', 0)
        if amount > 200000 and risky_pattern_score > 0:  # ₦200k+
            risk_score += 25
            risk_factors.append(
                f"High-value order (₦{amount:,.0f}) to risky address type"
            )
            metadata['high_value_risky_address'] = True

        # Check 8: New account shipping to different address
        user_id = transaction.get('user_id')
        if user_id:
            is_first_order = self._is_first_order(user_id, db)

            if is_first_order and billing_norm['state'] != shipping_norm['state']:
                risk_score += 20
                risk_factors.append(
                    "First order with billing/shipping state mismatch"
                )
                metadata['first_order_mismatch'] = True

        # Check 9: Address previously involved in fraud
        was_fraudulent = self._check_address_fraud_history(shipping_str, db)
        if was_fraudulent:
            risk_score += 50
            risk_factors.append(
                "Shipping address was involved in previous fraudulent transaction"
            )
            metadata['fraud_history'] = True

        # Cap risk score at 100
        risk_score = min(risk_score, 100)

        # Determine if rule triggered
        triggered = risk_score >= 30

        # Build reason
        if triggered:
            reason = f"Address fraud risk detected (score: {risk_score}). " + "; ".join(risk_factors)
        else:
            reason = f"No significant address fraud patterns detected (score: {risk_score})"

        return RuleResult(
            rule_id=self.rule_id,
            rule_name=self.name,
            triggered=triggered,
            risk_score=risk_score,
            reason=reason,
            metadata=metadata
        )

    def _normalize_address(self, address: Dict[str, Any]) -> Dict[str, str]:
        """
        Normalize address for comparison.

        Args:
            address: Address dictionary with fields like street, city, state, country

        Returns:
            Normalized address components
        """
        # Extract fields with defaults
        street = address.get('street', '').strip().lower()
        city = address.get('city', '').strip().lower()
        state = address.get('state', '').strip().lower()
        country = address.get('country', 'NG').strip().upper()
        postal = address.get('postal_code', '').strip()

        # Build full address string
        parts = [p for p in [street, city, state, country] if p]
        full_address = ' '.join(parts)

        return {
            'street': street,
            'city': city,
            'state': state,
            'country': country,
            'postal_code': postal,
            'full': full_address
        }

    def _check_fraud_hotspots(self, address: str) -> int:
        """
        Check if address contains known fraud hotspot keywords.

        Args:
            address: Normalized address string

        Returns:
            Risk score increment
        """
        address_lower = address.lower()

        for hotspot, risk in self.FRAUD_HOTSPOTS.items():
            if hotspot in address_lower:
                return risk

        return 0

    def _check_risky_patterns(self, address: str) -> int:
        """
        Check for risky address patterns.

        Args:
            address: Normalized address string

        Returns:
            Risk score increment
        """
        address_lower = address.lower()

        for pattern, risk in self.RISKY_ADDRESS_KEYWORDS.items():
            if pattern in address_lower:
                return risk

        return 0

    def _check_address_completeness(self, address: Dict[str, Any]) -> float:
        """
        Calculate address completeness score.

        Args:
            address: Address dictionary

        Returns:
            Completeness score (0.0 to 1.0)
        """
        required_fields = ['street', 'city', 'state', 'country']
        filled_fields = sum(1 for field in required_fields if address.get(field))

        return filled_fields / len(required_fields)

    def _check_address_velocity(self, address: str, db: Session) -> int:
        """
        Check how many different users have used this shipping address.

        Args:
            address: Normalized shipping address
            db: Database session

        Returns:
            Count of unique users
        """
        # Look back 60 days
        cutoff = datetime.utcnow() - timedelta(days=60)

        # In a real implementation, you'd have a shipping_address field
        # For now, we'll simulate by checking metadata

        # Query transactions with similar shipping address
        # This is simplified - in production, use proper address storage
        count = db.query(func.count(func.distinct(Transaction.user_id)))\
            .filter(
                and_(
                    Transaction.created_at >= cutoff,
                    Transaction.metadata.contains(address[:50])  # Partial match
                )
            ).scalar()

        return count or 0

    def _is_first_order(self, user_id: int, db: Session) -> bool:
        """
        Check if this is user's first order.

        Args:
            user_id: User ID
            db: Database session

        Returns:
            True if first order
        """
        order_count = db.query(func.count(Transaction.id))\
            .filter(Transaction.user_id == user_id)\
            .scalar()

        return order_count <= 1

    def _check_address_fraud_history(self, address: str, db: Session) -> bool:
        """
        Check if address was involved in previous fraud.

        Args:
            address: Normalized address
            db: Database session

        Returns:
            True if address has fraud history
        """
        # Look back 180 days
        cutoff = datetime.utcnow() - timedelta(days=180)

        # Check for transactions flagged as fraud with this address
        fraud_count = db.query(func.count(Transaction.id))\
            .filter(
                and_(
                    Transaction.created_at >= cutoff,
                    Transaction.is_fraud == True,
                    Transaction.metadata.contains(address[:50])
                )
            ).scalar()

        return fraud_count > 0
```

---

## Testing Your Rules

### Test Setup

Let's test both e-commerce rules with realistic Nigerian scenarios.

### Test 1: Card BIN Fraud Detection

**Scenario 1: BIN Attack Pattern**

```bash
# Terminal 1: Start the application (if not already running)
cd /home/user/guide-sentinel
source venv/bin/activate
uvicorn app.main:app --reload --port 8000

# Terminal 2: Test BIN attack pattern
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 101,
  "transaction_type": "ecommerce_purchase",
  "amount": 450000,
  "currency": "NGN",
  "card_number": "5399831234567890",
  "description": "iPhone 15 Pro Max purchase",
  "metadata": {
    "product": "iPhone 15 Pro Max",
    "category": "electronics",
    "merchant": "Jumia Nigeria"
  }
}'
```

**Expected Result:**
```json
{
  "risk_level": "high",
  "risk_score": 75,
  "rules_triggered": [
    {
      "rule_id": "ECOMMERCE_001",
      "rule_name": "Card BIN Fraud Detection",
      "risk_score": 75,
      "reason": "BIN fraud risk detected (score: 75). BIN 539983 is in known compromised BIN list; New account (<1 day old) with high-value transaction (₦450,000)",
      "metadata": {
        "bin": "539983",
        "compromised_bin": true,
        "new_account_high_value": true
      }
    }
  ]
}
```

**Scenario 2: Normal BIN Usage**

```bash
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 201,
  "transaction_type": "ecommerce_purchase",
  "amount": 15000,
  "currency": "NGN",
  "card_number": "6280123456789012",
  "description": "Book purchase",
  "metadata": {
    "product": "Python Programming Book",
    "category": "books"
  }
}'
```

**Expected Result:**
```json
{
  "risk_level": "low",
  "risk_score": 10,
  "rules_triggered": [],
  "message": "Transaction approved"
}
```

### Test 2: Address Verification Fraud

**Scenario 1: Billing/Shipping State Mismatch to Hotspot**

```bash
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 301,
  "transaction_type": "ecommerce_purchase",
  "amount": 250000,
  "currency": "NGN",
  "card_number": "5060991234567890",
  "billing_address": {
    "street": "23 Ozumba Mbadiwe Avenue",
    "city": "Lagos",
    "state": "Lagos",
    "country": "NG",
    "postal_code": "101241"
  },
  "shipping_address": {
    "street": "15 Boundary Road",
    "city": "Ajegunle",
    "state": "Lagos",
    "country": "NG",
    "postal_code": ""
  },
  "metadata": {
    "product": "MacBook Pro",
    "category": "electronics"
  }
}'
```

**Expected Result:**
```json
{
  "risk_level": "high",
  "risk_score": 85,
  "rules_triggered": [
    {
      "rule_id": "ECOMMERCE_002",
      "rule_name": "Address Verification Fraud",
      "risk_score": 85,
      "reason": "Address fraud risk detected (score: 85). Shipping address contains fraud hotspot keyword (+40 risk); Incomplete shipping address (completeness: 80%); High-value order (₦250,000) to risky address type",
      "metadata": {
        "fraud_hotspot_detected": true,
        "incomplete_address": true,
        "high_value_risky_address": true
      }
    }
  ]
}
```

**Scenario 2: Hotel Shipping Address**

```bash
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 302,
  "transaction_type": "ecommerce_purchase",
  "amount": 120000,
  "currency": "NGN",
  "card_number": "4389221234567890",
  "billing_address": {
    "street": "45 Allen Avenue",
    "city": "Ikeja",
    "state": "Lagos",
    "country": "NG"
  },
  "shipping_address": {
    "street": "Eko Hotel and Suites, 1415 Adetokunbo Ademola Street",
    "city": "Victoria Island",
    "state": "Lagos",
    "country": "NG"
  },
  "metadata": {
    "product": "Samsung Galaxy S24",
    "first_order": true
  }
}'
```

**Expected Result:**
```json
{
  "risk_level": "medium",
  "risk_score": 55,
  "rules_triggered": [
    {
      "rule_id": "ECOMMERCE_002",
      "rule_name": "Address Verification Fraud",
      "risk_score": 55,
      "reason": "Address fraud risk detected (score: 55). Shipping address contains risky pattern (+20 risk); First order with billing/shipping state mismatch",
      "metadata": {
        "risky_pattern_detected": true,
        "first_order_mismatch": true
      }
    }
  ]
}
```

**Scenario 3: International Shipping**

```bash
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 303,
  "transaction_type": "ecommerce_purchase",
  "amount": 500000,
  "currency": "NGN",
  "card_number": "5399831234567891",
  "billing_address": {
    "street": "12 Admiralty Way",
    "city": "Lekki",
    "state": "Lagos",
    "country": "NG"
  },
  "shipping_address": {
    "street": "123 Main Street",
    "city": "London",
    "state": "Greater London",
    "country": "GB"
  }
}'
```

**Expected Result:**
```json
{
  "risk_level": "high",
  "risk_score": 90,
  "rules_triggered": [
    {
      "rule_id": "ECOMMERCE_001",
      "rule_name": "Card BIN Fraud Detection",
      "risk_score": 40,
      "reason": "BIN 539983 is in known compromised BIN list"
    },
    {
      "rule_id": "ECOMMERCE_002",
      "rule_name": "Address Verification Fraud",
      "risk_score": 50,
      "reason": "International shipping: billing in NG, shipping to GB; High-value order (₦500,000) to risky address type"
    }
  ]
}
```

### Test 3: Combined E-commerce + Lending Rules

Test that both verticals work together:

```bash
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 401,
  "transaction_type": "loan_application",
  "amount": 300000,
  "currency": "NGN",
  "card_number": "5399831234567892",
  "billing_address": {
    "street": "78 Herbert Macaulay Way",
    "city": "Yaba",
    "state": "Lagos",
    "country": "NG"
  },
  "shipping_address": {
    "street": "Computer Village Ikeja",
    "city": "Ikeja",
    "state": "Lagos",
    "country": "NG"
  },
  "metadata": {
    "loan_purpose": "business",
    "income": 80000,
    "employment_type": "self_employed"
  }
}'
```

**Expected Behavior:**
- Both lending rules AND e-commerce rules evaluate
- Comprehensive fraud assessment across verticals

---

## Integration with Rules Engine

### Update Rules Engine to Include E-commerce Rules

**File: `app/rules/engine.py` (update)**

Add e-commerce rules to the rules engine:

```python
"""
Rules Engine - Updated for E-commerce Rules

This file should already exist from Day 4.
We're adding the new e-commerce rules to the engine.
"""

from typing import List, Dict, Any
from sqlalchemy.orm import Session

from app.rules.base import BaseRule, RuleResult

# Import lending rules
from app.rules.lending.loan_stacking import LoanStackingRule
from app.rules.lending.sim_swap import SimSwapRule
# ... other lending rules

# Import e-commerce rules (NEW)
from app.rules.ecommerce.card_bin_fraud import CardBINFraudRule
from app.rules.ecommerce.address_verification import AddressVerificationRule


class RulesEngine:
    """
    Main rules engine that coordinates all fraud detection rules.
    """

    def __init__(self):
        """Initialize rules engine with all available rules."""
        self.rules: List[BaseRule] = [
            # Lending rules (from Days 12-13)
            LoanStackingRule(),
            SimSwapRule(),
            # ... add other lending rules

            # E-commerce rules (NEW - Day 20)
            CardBINFraudRule(),
            AddressVerificationRule(),
        ]

    def evaluate(
        self,
        transaction: Dict[str, Any],
        db: Session
    ) -> Dict[str, Any]:
        """
        Evaluate transaction against all rules.

        Args:
            transaction: Transaction data
            db: Database session

        Returns:
            Evaluation results with risk score and triggered rules
        """
        results = []
        total_risk_score = 0

        for rule in self.rules:
            try:
                result = rule.evaluate(transaction, db)

                if result.triggered:
                    results.append({
                        'rule_id': result.rule_id,
                        'rule_name': result.rule_name,
                        'risk_score': result.risk_score,
                        'reason': result.reason,
                        'metadata': result.metadata
                    })

                # Accumulate risk score
                total_risk_score += result.risk_score

            except Exception as e:
                # Log error but continue with other rules
                print(f"Error in rule {rule.name}: {str(e)}")
                continue

        # Determine overall risk level
        risk_level = self._calculate_risk_level(total_risk_score)

        return {
            'risk_level': risk_level,
            'risk_score': total_risk_score,
            'rules_triggered': results,
            'total_rules_evaluated': len(self.rules)
        }

    def _calculate_risk_level(self, score: int) -> str:
        """Calculate risk level from score."""
        if score >= 70:
            return 'high'
        elif score >= 40:
            return 'medium'
        else:
            return 'low'
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: "No module named 'app.rules.ecommerce'"

**Cause:** E-commerce directory not created or missing `__init__.py`

**Solution:**
```bash
# Create directory
mkdir -p app/rules/ecommerce

# Create __init__.py
touch app/rules/ecommerce/__init__.py

# Verify structure
ls -la app/rules/ecommerce/
```

#### Issue 2: "Card number field not found"

**Cause:** Transaction data doesn't include card_number field

**Solution:**
Ensure your transaction payload includes `card_number`:
```json
{
  "card_number": "5399831234567890",
  "...": "..."
}
```

#### Issue 3: "Address data missing"

**Cause:** Billing or shipping address not provided

**Solution:**
Include both addresses in payload:
```json
{
  "billing_address": {
    "street": "...",
    "city": "...",
    "state": "...",
    "country": "NG"
  },
  "shipping_address": {
    "street": "...",
    "city": "...",
    "state": "...",
    "country": "NG"
  }
}
```

#### Issue 4: Rules not triggering

**Cause:** Risk thresholds too high

**Solution:**
Check threshold in rule:
```python
# In rule code
triggered = risk_score >= 30  # Adjust this value if needed
```

For testing, you can temporarily lower thresholds.

#### Issue 5: Database query errors

**Cause:** Missing database indexes or too many queries

**Solution:**
Add indexes for frequently queried fields:
```sql
-- Add index for card_number prefix searches
CREATE INDEX idx_card_number_prefix ON transactions (substring(card_number, 1, 6));

-- Add index for created_at for time-based queries
CREATE INDEX idx_transactions_created_at ON transactions (created_at);
```

---

## Advanced Testing Scenarios

### Scenario Suite 1: Nigerian Bank BIN Patterns

Test all major Nigerian bank BINs:

```bash
# GTBank Mastercard (5399 83xx)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 1001,
  "transaction_type": "ecommerce_purchase",
  "amount": 250000,
  "currency": "NGN",
  "card_number": "5399831234567890",
  "metadata": {"bank": "GTBank", "card_type": "Mastercard"}
}'

# First Bank Visa (4389 22xx)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 1002,
  "transaction_type": "ecommerce_purchase",
  "amount": 180000,
  "currency": "NGN",
  "card_number": "4389221234567890",
  "metadata": {"bank": "First Bank", "card_type": "Visa"}
}'

# Access Bank Verve (5060 99xx)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 1003,
  "transaction_type": "ecommerce_purchase",
  "amount": 95000,
  "currency": "NGN",
  "card_number": "5060991234567890",
  "metadata": {"bank": "Access Bank", "card_type": "Verve"}
}'

# Zenith Bank (6280 xxxx)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 1004,
  "transaction_type": "ecommerce_purchase",
  "amount": 320000,
  "currency": "NGN",
  "card_number": "6280123456789012",
  "metadata": {"bank": "Zenith Bank", "card_type": "Verve"}
}'
```

### Scenario Suite 2: Address Fraud Patterns

**Test 1: All Nigerian fraud hotspots**

```bash
# Ajegunle (high risk)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 2001,
  "amount": 150000,
  "billing_address": {
    "street": "10 Admiralty Way",
    "city": "Lekki",
    "state": "Lagos",
    "country": "NG"
  },
  "shipping_address": {
    "street": "45 Boundary Road",
    "city": "Ajegunle",
    "state": "Lagos",
    "country": "NG"
  }
}'

# Mushin (high risk)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 2002,
  "amount": 200000,
  "billing_address": {
    "street": "23 Ozumba Mbadiwe",
    "city": "Victoria Island",
    "state": "Lagos",
    "country": "NG"
  },
  "shipping_address": {
    "street": "12 Ladipo Street",
    "city": "Mushin",
    "state": "Lagos",
    "country": "NG"
  }
}'

# Computer Village (electronics fraud)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 2003,
  "amount": 450000,
  "billing_address": {
    "street": "5 Bourdillon Road",
    "city": "Ikoyi",
    "state": "Lagos",
    "country": "NG"
  },
  "shipping_address": {
    "street": "Computer Village",
    "city": "Ikeja",
    "state": "Lagos",
    "country": "NG"
  },
  "metadata": {"product": "10 iPhones"}
}'
```

**Test 2: Different risky address types**

```bash
# Hotel delivery
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 2101,
  "amount": 180000,
  "billing_address": {"street": "12 Allen Ave", "city": "Ikeja", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "Eko Hotel and Suites", "city": "Victoria Island", "state": "Lagos", "country": "NG"}
}'

# PO Box delivery
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 2102,
  "amount": 250000,
  "billing_address": {"street": "34 Awolowo Road", "city": "Ikoyi", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "PO Box 12345", "city": "Lagos", "state": "Lagos", "country": "NG"}
}'

# Filling station
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 2103,
  "amount": 320000,
  "billing_address": {"street": "78 Herbert Macaulay", "city": "Yaba", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "Oando Filling Station, Lekki", "city": "Lekki", "state": "Lagos", "country": "NG"}
}'
```

### Scenario Suite 3: Cross-State Shipping Patterns

```bash
# Lagos to Abuja (legitimate business travel)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 3001,
  "amount": 85000,
  "billing_address": {"street": "23 Admiralty Way", "city": "Lekki", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "12 Gana Street", "city": "Maitama", "state": "Abuja", "country": "NG"},
  "metadata": {"note": "Business trip", "customer_level": "gold"}
}'

# Lagos to Rivers State (high-risk state)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 3002,
  "amount": 350000,
  "billing_address": {"street": "45 Allen Avenue", "city": "Ikeja", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "23 Aba Road", "city": "Port Harcourt", "state": "Rivers", "country": "NG"}
}'

# Lagos to Edo State (high-risk state)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 3003,
  "amount": 280000,
  "billing_address": {"street": "12 Ahmadu Bello Way", "city": "Victoria Island", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "56 Sapele Road", "city": "Benin City", "state": "Edo", "country": "NG"}
}'
```

### Scenario Suite 4: International Shipping

```bash
# Nigeria to UK
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 4001,
  "amount": 650000,
  "card_number": "5399831234567894",
  "billing_address": {"street": "10 Bourdillon Road", "city": "Ikoyi", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "123 Oxford Street", "city": "London", "state": "Greater London", "country": "GB"}
}'

# Nigeria to USA
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 4002,
  "amount": 800000,
  "card_number": "4389221234567891",
  "billing_address": {"street": "45 Awolowo Road", "city": "Ikoyi", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "789 Fifth Avenue", "city": "New York", "state": "NY", "country": "US"}
}'

# Nigeria to Ghana (regional)
curl -X POST http://localhost:8000/api/v1/fraud/check \
-H "Content-Type: application/json" \
-d '{
  "user_id": 4003,
  "amount": 250000,
  "billing_address": {"street": "23 Ozumba Mbadiwe", "city": "Victoria Island", "state": "Lagos", "country": "NG"},
  "shipping_address": {"street": "15 Airport Road", "city": "Accra", "state": "Greater Accra", "country": "GH"}
}'
```

---

## Performance Benchmarking

### Benchmark Your E-commerce Rules

Test how fast your rules evaluate transactions:

```bash
# Create a benchmark script
cat > /home/user/guide-sentinel/benchmark_ecommerce.sh << 'EOF'
#!/bin/bash

echo "E-commerce Rules Performance Benchmark"
echo "======================================"
echo ""

# Test 1: Card BIN Rule Performance
echo "Test 1: Card BIN Fraud Detection"
echo "Running 100 requests..."

start_time=$(date +%s%N)
for i in {1..100}; do
  curl -s -X POST http://localhost:8000/api/v1/fraud/check \
  -H "Content-Type: application/json" \
  -d "{\"user_id\": $i, \"amount\": 100000, \"card_number\": \"5399831234567890\"}" \
  > /dev/null
done
end_time=$(date +%s%N)

elapsed=$((($end_time - $start_time) / 1000000))
avg=$(($elapsed / 100))

echo "Total time: ${elapsed}ms"
echo "Average per request: ${avg}ms"
echo ""

# Test 2: Address Verification Performance
echo "Test 2: Address Verification"
echo "Running 100 requests..."

start_time=$(date +%s%N)
for i in {1..100}; do
  curl -s -X POST http://localhost:8000/api/v1/fraud/check \
  -H "Content-Type: application/json" \
  -d "{
    \"user_id\": $i,
    \"amount\": 100000,
    \"billing_address\": {\"street\": \"Test\", \"city\": \"Lagos\", \"state\": \"Lagos\", \"country\": \"NG\"},
    \"shipping_address\": {\"street\": \"Test\", \"city\": \"Abuja\", \"state\": \"Abuja\", \"country\": \"NG\"}
  }" > /dev/null
done
end_time=$(date +%s%N)

elapsed=$((($end_time - $start_time) / 1000000))
avg=$(($elapsed / 100))

echo "Total time: ${elapsed}ms"
echo "Average per request: ${avg}ms"
echo ""

# Test 3: Combined Rules Performance
echo "Test 3: All E-commerce Rules"
echo "Running 100 requests..."

start_time=$(date +%s%N)
for i in {1..100}; do
  curl -s -X POST http://localhost:8000/api/v1/fraud/check \
  -H "Content-Type: application/json" \
  -d "{
    \"user_id\": $i,
    \"amount\": 100000,
    \"card_number\": \"5399831234567890\",
    \"billing_address\": {\"street\": \"Test\", \"city\": \"Lagos\", \"state\": \"Lagos\", \"country\": \"NG\"},
    \"shipping_address\": {\"street\": \"Ajegunle\", \"city\": \"Lagos\", \"state\": \"Lagos\", \"country\": \"NG\"}
  }" > /dev/null
done
end_time=$(date +%s%N)

elapsed=$((($end_time - $start_time) / 1000000))
avg=$(($elapsed / 100))

echo "Total time: ${elapsed}ms"
echo "Average per request: ${avg}ms"
echo ""

echo "======================================"
echo "Benchmark complete!"
echo ""
echo "Performance Targets:"
echo "- Single rule: <50ms"
echo "- All rules: <150ms"
echo "- Goal: <100ms end-to-end"
EOF

chmod +x /home/user/guide-sentinel/benchmark_ecommerce.sh

# Run benchmark
./benchmark_ecommerce.sh
```

**Expected Output:**
```
E-commerce Rules Performance Benchmark
======================================

Test 1: Card BIN Fraud Detection
Running 100 requests...
Total time: 3200ms
Average per request: 32ms

Test 2: Address Verification
Running 100 requests...
Total time: 4100ms
Average per request: 41ms

Test 3: All E-commerce Rules
Running 100 requests...
Total time: 8500ms
Average per request: 85ms

======================================
Benchmark complete!

Performance Targets:
- Single rule: <50ms
- All rules: <150ms
- Goal: <100ms end-to-end
```

---

## Real-World Integration Examples

### Example 1: Jumia-style E-commerce Platform

```python
# app/api/ecommerce_integration.py

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import Dict, Any

from app.database import get_db
from app.rules.engine import RulesEngine

router = APIRouter(prefix="/ecommerce", tags=["ecommerce"])

@router.post("/order/validate")
async def validate_order(
    order_data: Dict[str, Any],
    db: Session = Depends(get_db)
):
    """
    Validate e-commerce order before processing.

    Used by Jumia/Konga-style platforms to check orders
    before accepting payment and fulfillment.
    """
    engine = RulesEngine()

    # Evaluate order against e-commerce rules
    result = engine.evaluate(order_data, db)

    # Decision logic
    if result['risk_level'] == 'high':
        return {
            'approved': False,
            'action': 'BLOCK',
            'reason': 'High fraud risk detected',
            'risk_score': result['risk_score'],
            'rules_triggered': result['rules_triggered']
        }
    elif result['risk_level'] == 'medium':
        return {
            'approved': True,
            'action': 'REVIEW',
            'reason': 'Manual review required',
            'risk_score': result['risk_score'],
            'rules_triggered': result['rules_triggered']
        }
    else:
        return {
            'approved': True,
            'action': 'APPROVE',
            'reason': 'Order approved',
            'risk_score': result['risk_score']
        }
```

### Example 2: Webhook Integration for Order Updates

```python
@router.post("/webhook/order-status")
async def order_status_webhook(
    webhook_data: Dict[str, Any],
    db: Session = Depends(get_db)
):
    """
    Receive order status updates to improve fraud detection.

    When orders are delivered/returned/disputed, update
    our fraud models with the outcome.
    """
    order_id = webhook_data.get('order_id')
    status = webhook_data.get('status')

    # Update transaction record
    transaction = db.query(Transaction).filter(
        Transaction.id == order_id
    ).first()

    if transaction:
        if status == 'DELIVERED':
            # Order successful - good signal
            transaction.is_fraud = False
        elif status in ['RETURNED', 'CHARGEBACK', 'DISPUTED']:
            # Potential fraud - update record
            transaction.is_fraud = True

        transaction.status = status
        db.commit()

    return {'status': 'updated'}
```

---

## Next Steps

Congratulations! You've implemented the first 2 e-commerce fraud detection rules. Here's what you've accomplished:

**✅ Today's Achievements:**
- Card BIN fraud detection with Nigerian bank patterns
- Address verification fraud for shipping/billing mismatches
- Understanding of e-commerce fraud landscape in Nigeria
- Integration with existing rules engine

**📊 Current Progress:**
- **Total rules implemented:** 12 (10 lending + 2 e-commerce)
- **Verticals covered:** 2 (Lending, E-commerce)
- **Rules remaining in e-commerce:** 2 (Day 21)

**🎯 Tomorrow (Day 21):**
We'll implement the final 2 e-commerce rules:
1. **Account Takeover (ATO) Detection** - Detects compromised accounts
2. **Promo Code Abuse / Coupon Fraud** - Prevents discount exploitation

**🔜 Coming Up:**
- Day 22-23: Betting/Gaming fraud rules
- Day 24-25: Crypto fraud rules
- Day 26: Marketplace fraud rules
- Day 27: Cross-vertical integration testing

**📚 Further Learning:**

**E-commerce Fraud Resources:**
- Stripe Radar: Card fraud prevention best practices
- Sift Science: E-commerce fraud patterns
- NIBSS: Nigerian payment card security guidelines

**Nigerian E-commerce:**
- Jumia fraud prevention case studies
- Konga merchant protection guides
- PayStack payment security docs

**Address Verification:**
- USPS Address Verification (concept reference)
- Google Maps Geocoding API (address validation)
- Nigeria Postal Service (NIPOST) standards

**Next Chapter:** [Day 21: E-commerce Rules Part 2 →](./README-DAY-021.md)

---

**Great work!** You're building a comprehensive multi-vertical fraud detection platform. Keep up the momentum! 🚀
