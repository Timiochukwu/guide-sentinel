# Day 3: Pydantic Schemas & Configuration - Sentinel Fraud Detection Platform

**Building Production-Ready Data Models with Type Safety and Validation**

---

## Table of Contents
1. [Overview](#overview)
2. [What is Pydantic?](#what-is-pydantic)
3. [Core Configuration Setup](#core-configuration-setup)
4. [Complete Schema Implementation](#complete-schema-implementation)
5. [Data Validation Deep Dive](#data-validation-deep-dive)
6. [Testing Your Schemas](#testing-your-schemas)
7. [Advanced Pydantic Features](#advanced-pydantic-features)
8. [Troubleshooting](#troubleshooting)
9. [Next Steps](#next-steps)

---

## Overview

**Day 3 Objectives:**
- Build production-ready Pydantic models for fraud detection
- Implement comprehensive data validation
- Configure application settings with environment variables
- Create type-safe schemas for API requests and responses
- Test validation logic thoroughly

**What You'll Build:**
- ✅ Core configuration system with fraud thresholds
- ✅ 8 Pydantic schemas with validation
- ✅ 4 Enums for type safety
- ✅ Custom validators for business logic
- ✅ Test scripts for validation

**Estimated Time:** 2-3 hours

---

## What is Pydantic?

### Introduction to Pydantic

**Pydantic** is a data validation library that uses Python type hints to validate and parse data. It's the foundation of FastAPI's automatic data validation.

**Why Use Pydantic?**
1. **Automatic Validation** - Data is validated automatically using type hints
2. **Type Safety** - Catch errors at development time, not runtime
3. **Auto Documentation** - FastAPI uses schemas to generate Swagger docs
4. **Performance** - Written in Rust (Pydantic v2), extremely fast
5. **Developer Experience** - Clear error messages, IDE autocomplete

**Traditional Python vs Pydantic:**

```python
# ❌ Traditional Python - No validation
def process_transaction(data: dict):
    amount = data.get('amount')  # Could be None, string, negative, etc.
    user_id = data.get('user_id')
    # No guarantee data is valid!
    return amount * 2

# ✅ With Pydantic - Automatic validation
from pydantic import BaseModel, Field

class Transaction(BaseModel):
    amount: float = Field(gt=0, description="Must be positive")
    user_id: str = Field(min_length=1)

def process_transaction(data: Transaction):
    # data.amount is GUARANTEED to be a positive float
    # data.user_id is GUARANTEED to be a non-empty string
    return data.amount * 2
```

### Type Hints in Python

**Type hints** are Python's way of indicating what type a variable should be:

```python
# Basic type hints
age: int = 25
name: str = "John"
price: float = 99.99
is_valid: bool = True

# Collection type hints
numbers: list[int] = [1, 2, 3]
mapping: dict[str, int] = {"a": 1, "b": 2}

# Optional types (can be None)
from typing import Optional
middle_name: Optional[str] = None  # Can be str or None

# Union types (multiple possible types)
from typing import Union
value: Union[int, str] = "123"  # Can be int OR str
```

---

## Core Configuration Setup

### Step 1: Create Core Module

First, let's set up the core module structure:

```bash
# Create the core module directory
mkdir -p /home/user/sentinel-api/app/core
touch /home/user/sentinel-api/app/core/__init__.py
touch /home/user/sentinel-api/app/core/config.py
```

### Step 2: Core __init__.py

**File: `/home/user/sentinel-api/app/core/__init__.py`**

```python
"""
Core module for Sentinel Fraud Detection Platform.

This module provides:
- Application configuration (config.py)
- Global settings and environment variables
- Fraud detection thresholds
- Database connection settings (future)
- Logging configuration (future)
"""

from app.core.config import settings

# Export settings for easy import
__all__ = ["settings"]

# Version information
__version__ = "0.1.0"
__author__ = "Sentinel Team"
```

### Step 3: Complete Configuration with Pydantic Settings

**File: `/home/user/sentinel-api/app/core/config.py`**

```python
"""
Application Configuration using Pydantic Settings.

This module defines all configuration parameters for the Sentinel platform,
including fraud detection thresholds, API settings, and environment-specific configs.

Environment Variables:
    - APP_NAME: Application name (default: "Sentinel Fraud Detection")
    - APP_VERSION: API version (default: "1.0.0")
    - DEBUG: Enable debug mode (default: False)
    - VELOCITY_TIME_WINDOW_MINUTES: Time window for velocity checks (default: 60)
    - MAX_TRANSACTIONS_PER_WINDOW: Max transactions allowed in time window (default: 10)
    - HIGH_RISK_AMOUNT_THRESHOLD: Amount threshold for high risk (default: 10000.0)
    - EXTREME_RISK_AMOUNT_THRESHOLD: Amount threshold for extreme risk (default: 50000.0)
    - SUSPICIOUS_HOUR_START: Start of suspicious hours (default: 1)
    - SUSPICIOUS_HOUR_END: End of suspicious hours (default: 5)

Usage:
    from app.core.config import settings

    print(settings.APP_NAME)
    if settings.DEBUG:
        print("Debug mode enabled")
"""

from typing import ClassVar
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """
    Application settings with environment variable support.

    Uses Pydantic Settings to load configuration from:
    1. Environment variables
    2. .env file (if present)
    3. Default values defined here

    All settings are immutable and validated on application startup.
    """

    # ==========================================
    # APPLICATION SETTINGS
    # ==========================================

    APP_NAME: str = "Sentinel Fraud Detection"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False

    # API Configuration
    API_PREFIX: str = "/api/v1"
    DOCS_URL: str = "/docs"
    REDOC_URL: str = "/redoc"

    # ==========================================
    # FRAUD DETECTION THRESHOLDS
    # ==========================================

    # Velocity Checks (Rate Limiting)
    VELOCITY_TIME_WINDOW_MINUTES: int = 60  # Check transactions in last 60 minutes
    MAX_TRANSACTIONS_PER_WINDOW: int = 10   # Max 10 transactions per hour

    # Amount-Based Thresholds
    HIGH_RISK_AMOUNT_THRESHOLD: float = 10000.0     # $10,000 - High risk
    EXTREME_RISK_AMOUNT_THRESHOLD: float = 50000.0  # $50,000 - Extreme risk
    LARGE_TRANSACTION_THRESHOLD: float = 5000.0     # $5,000 - Flag as large

    # Time-Based Checks
    SUSPICIOUS_HOUR_START: int = 1  # 1 AM
    SUSPICIOUS_HOUR_END: int = 5     # 5 AM

    # Geographic Risk
    HIGH_RISK_COUNTRIES: ClassVar[list[str]] = [
        "NG",  # Nigeria
        "PK",  # Pakistan
        "VN",  # Vietnam
        "ID",  # Indonesia
    ]

    # Industry Risk Multipliers
    INDUSTRY_RISK_MULTIPLIERS: ClassVar[dict[str, float]] = {
        "CRYPTO": 1.5,      # 50% higher risk
        "GAMBLING": 1.4,    # 40% higher risk
        "ADULT": 1.3,       # 30% higher risk
        "RETAIL": 1.0,      # Baseline risk
        "HEALTHCARE": 0.9,  # 10% lower risk
        "EDUCATION": 0.8,   # 20% lower risk
        "SAAS": 0.85,       # 15% lower risk
    }

    # Device & IP Checks
    MAX_ACCOUNTS_PER_DEVICE: int = 3    # Max accounts per device fingerprint
    VPN_RISK_SCORE_INCREASE: float = 15.0  # Add 15 points if VPN detected
    TOR_RISK_SCORE_INCREASE: float = 30.0  # Add 30 points if Tor detected

    # User Behavior
    NEW_USER_THRESHOLD_DAYS: int = 30   # Users < 30 days are "new"
    NEW_USER_RISK_INCREASE: float = 10.0  # Add 10 points for new users

    # ==========================================
    # RISK SCORING CONFIGURATION
    # ==========================================

    # Risk Level Boundaries
    LOW_RISK_THRESHOLD: float = 30.0      # 0-30: Low risk
    MEDIUM_RISK_THRESHOLD: float = 60.0   # 30-60: Medium risk
    HIGH_RISK_THRESHOLD: float = 85.0     # 60-85: High risk
    # Above 85: Critical risk

    # ==========================================
    # PYDANTIC SETTINGS CONFIGURATION
    # ==========================================

    model_config = SettingsConfigDict(
        # Look for .env file in parent directory
        env_file=".env",
        env_file_encoding="utf-8",
        # Environment variables take precedence over .env file
        env_prefix="",  # No prefix needed
        # Allow extra fields in environment (ignore unknown vars)
        extra="ignore",
        # Make settings immutable (can't change after creation)
        frozen=True,
    )


# ==========================================
# GLOBAL SETTINGS INSTANCE
# ==========================================

# Create a single settings instance to use throughout the application
# This is loaded once on startup and cached
settings = Settings()


# ==========================================
# HELPER FUNCTIONS
# ==========================================

def get_risk_level_name(score: float) -> str:
    """
    Convert a numeric risk score to a risk level name.

    Args:
        score: Risk score (0-100)

    Returns:
        Risk level name: "LOW", "MEDIUM", "HIGH", or "CRITICAL"

    Example:
        >>> get_risk_level_name(25.0)
        'LOW'
        >>> get_risk_level_name(75.0)
        'HIGH'
    """
    if score < settings.LOW_RISK_THRESHOLD:
        return "LOW"
    elif score < settings.MEDIUM_RISK_THRESHOLD:
        return "MEDIUM"
    elif score < settings.HIGH_RISK_THRESHOLD:
        return "HIGH"
    else:
        return "CRITICAL"


def is_high_risk_country(country_code: str) -> bool:
    """
    Check if a country code is in the high-risk list.

    Args:
        country_code: ISO 3166-1 alpha-2 country code (e.g., "US", "NG")

    Returns:
        True if country is high risk, False otherwise

    Example:
        >>> is_high_risk_country("NG")
        True
        >>> is_high_risk_country("US")
        False
    """
    return country_code.upper() in settings.HIGH_RISK_COUNTRIES


def get_industry_risk_multiplier(industry: str) -> float:
    """
    Get the risk multiplier for a specific industry.

    Args:
        industry: Industry name (e.g., "CRYPTO", "RETAIL")

    Returns:
        Risk multiplier (default 1.0 if industry not found)

    Example:
        >>> get_industry_risk_multiplier("CRYPTO")
        1.5
        >>> get_industry_risk_multiplier("UNKNOWN")
        1.0
    """
    return settings.INDUSTRY_RISK_MULTIPLIERS.get(industry.upper(), 1.0)


# ==========================================
# EXAMPLE .ENV FILE
# ==========================================

"""
Example .env file for local development:

# Application
APP_NAME="Sentinel Fraud Detection - DEV"
DEBUG=true

# Fraud Thresholds
VELOCITY_TIME_WINDOW_MINUTES=30
MAX_TRANSACTIONS_PER_WINDOW=5
HIGH_RISK_AMOUNT_THRESHOLD=5000.0
EXTREME_RISK_AMOUNT_THRESHOLD=25000.0

# For production:
DEBUG=false
VELOCITY_TIME_WINDOW_MINUTES=60
MAX_TRANSACTIONS_PER_WINDOW=10
"""
```

---

## Complete Schema Implementation

### Step 1: Create Models Module

```bash
# Create models directory
mkdir -p /home/user/sentinel-api/app/models
touch /home/user/sentinel-api/app/models/__init__.py
touch /home/user/sentinel-api/app/models/schemas.py
```

### Step 2: Models __init__.py

**File: `/home/user/sentinel-api/app/models/__init__.py`**

```python
"""
Models module for Sentinel Fraud Detection Platform.

This module contains all Pydantic schemas for:
- API request/response models
- Data validation
- JSON schema generation
- OpenAPI documentation
"""

from app.models.schemas import (
    # Enums
    Industry,
    TransactionType,
    FraudVerdict,
    RiskLevel,
    # Request/Response Models
    DeviceInfo,
    UserInfo,
    LocationInfo,
    TransactionCheckRequest,
    FraudFlag,
    FraudCheckResponse,
)

__all__ = [
    # Enums
    "Industry",
    "TransactionType",
    "FraudVerdict",
    "RiskLevel",
    # Schemas
    "DeviceInfo",
    "UserInfo",
    "LocationInfo",
    "TransactionCheckRequest",
    "FraudFlag",
    "FraudCheckResponse",
]
```

### Step 3: Complete Schemas with Validation

**File: `/home/user/sentinel-api/app/models/schemas.py`**

```python
"""
Pydantic schemas for Sentinel Fraud Detection Platform.

This module defines all request/response models with comprehensive validation:
- Field-level validation (types, ranges, patterns)
- Custom validators for business logic
- JSON schema generation for API docs
- Type safety throughout the application

All models use Pydantic v2 syntax and follow best practices for production APIs.
"""

from datetime import datetime
from enum import Enum
from typing import Optional, Any
from pydantic import (
    BaseModel,
    Field,
    field_validator,
    model_validator,
    ConfigDict,
)
import re


# ==========================================
# ENUMS FOR TYPE SAFETY
# ==========================================

class Industry(str, Enum):
    """
    Industry classification for merchants.

    Different industries have different fraud risk profiles.
    Used to apply industry-specific risk multipliers.
    """
    RETAIL = "RETAIL"           # E-commerce, physical stores
    CRYPTO = "CRYPTO"           # Cryptocurrency, blockchain
    GAMBLING = "GAMBLING"       # Online casinos, betting
    ADULT = "ADULT"             # Adult entertainment
    HEALTHCARE = "HEALTHCARE"   # Medical, pharmacy
    EDUCATION = "EDUCATION"     # Schools, online courses
    SAAS = "SAAS"               # Software as a Service


class TransactionType(str, Enum):
    """
    Type of transaction being processed.

    Different transaction types have different risk patterns:
    - PAYMENT: Standard payment processing
    - REFUND: Money being returned to customer
    - PAYOUT: Money being sent out (withdrawals, payouts)
    - CHARGEBACK: Disputed transaction
    """
    PAYMENT = "PAYMENT"
    REFUND = "REFUND"
    PAYOUT = "PAYOUT"
    CHARGEBACK = "CHARGEBACK"


class FraudVerdict(str, Enum):
    """
    Final fraud detection verdict.

    APPROVE: Transaction is safe, allow it
    REVIEW: Manual review needed, hold for investigation
    DECLINE: High fraud risk, reject transaction
    """
    APPROVE = "APPROVE"
    REVIEW = "REVIEW"
    DECLINE = "DECLINE"


class RiskLevel(str, Enum):
    """
    Risk level classification based on risk score.

    LOW: 0-30 points
    MEDIUM: 30-60 points
    HIGH: 60-85 points
    CRITICAL: 85-100 points
    """
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"
    CRITICAL = "CRITICAL"


# ==========================================
# NESTED SCHEMAS (EMBEDDED OBJECTS)
# ==========================================

class DeviceInfo(BaseModel):
    """
    Device information from the client.

    Used for device fingerprinting and tracking suspicious device patterns:
    - Multiple accounts from same device
    - Unusual device characteristics
    - Known fraud device fingerprints

    Attributes:
        device_id: Unique device fingerprint (40 char hash)
        ip_address: Client IP address (IPv4 or IPv6)
        user_agent: Browser/app user agent string
        is_vpn: Whether VPN/proxy detected (optional)
        is_tor: Whether Tor network detected (optional)
    """

    device_id: str = Field(
        ...,  # Required field (... means required in Pydantic)
        min_length=40,
        max_length=40,
        description="Device fingerprint hash (SHA-1, 40 characters)",
        examples=["a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0"],
    )

    ip_address: str = Field(
        ...,
        min_length=7,
        max_length=45,  # IPv6 can be up to 45 chars
        description="Client IP address",
        examples=["192.168.1.100", "2001:0db8:85a3:0000:0000:8a2e:0370:7334"],
    )

    user_agent: str = Field(
        ...,
        min_length=10,
        max_length=500,
        description="Browser or app user agent string",
        examples=["Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"],
    )

    is_vpn: Optional[bool] = Field(
        default=None,
        description="Whether VPN or proxy is detected",
        examples=[False, True],
    )

    is_tor: Optional[bool] = Field(
        default=None,
        description="Whether Tor network is detected",
        examples=[False, True],
    )

    # Pydantic v2 config
    model_config = ConfigDict(
        # Generate example in OpenAPI docs
        json_schema_extra={
            "example": {
                "device_id": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
                "ip_address": "203.0.113.42",
                "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 14_6 like Mac OS X)",
                "is_vpn": False,
                "is_tor": False,
            }
        }
    )

    @field_validator("ip_address")
    @classmethod
    def validate_ip_address(cls, v: str) -> str:
        """
        Validate IP address format (basic validation).

        In production, use ipaddress module for thorough validation.
        """
        # Basic check: contains dots or colons
        if not ("." in v or ":" in v):
            raise ValueError("Invalid IP address format")
        return v

    @field_validator("device_id")
    @classmethod
    def validate_device_id(cls, v: str) -> str:
        """
        Validate device ID is hexadecimal.

        Device fingerprints should be SHA-1 hashes (40 hex characters).
        """
        if not re.match(r"^[a-f0-9]{40}$", v.lower()):
            raise ValueError("Device ID must be 40 hexadecimal characters")
        return v.lower()


class UserInfo(BaseModel):
    """
    User/customer information.

    Used for user behavior analysis:
    - New user detection
    - Account age risk scoring
    - User reputation
    - Historical fraud patterns

    Attributes:
        user_id: Unique user identifier
        email: User email address
        account_age_days: Age of account in days
        is_verified: Whether email/phone verified (optional)
        previous_fraud_reports: Number of past fraud reports (optional)
    """

    user_id: str = Field(
        ...,
        min_length=1,
        max_length=100,
        description="Unique user identifier",
        examples=["user_12345", "uuid-1234-5678-90ab-cdef"],
    )

    email: str = Field(
        ...,
        min_length=5,
        max_length=255,
        description="User email address",
        examples=["user@example.com"],
    )

    account_age_days: int = Field(
        ...,
        ge=0,  # Greater than or equal to 0
        le=36500,  # Max 100 years
        description="Age of user account in days",
        examples=[5, 365, 1825],
    )

    is_verified: Optional[bool] = Field(
        default=None,
        description="Whether user has verified email/phone",
        examples=[True, False],
    )

    previous_fraud_reports: Optional[int] = Field(
        default=0,
        ge=0,
        description="Number of previous fraud reports against this user",
        examples=[0, 1, 5],
    )

    model_config = ConfigDict(
        json_schema_extra={
            "example": {
                "user_id": "user_abc123",
                "email": "john.doe@example.com",
                "account_age_days": 45,
                "is_verified": True,
                "previous_fraud_reports": 0,
            }
        }
    )

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        """
        Basic email validation.

        Checks for @ symbol and basic structure.
        In production, use email-validator library.
        """
        if "@" not in v or "." not in v.split("@")[1]:
            raise ValueError("Invalid email format")
        return v.lower()


class LocationInfo(BaseModel):
    """
    Geographic location information.

    Used for:
    - High-risk country detection
    - Impossible travel detection
    - IP geolocation verification
    - Country-specific rules

    Attributes:
        country_code: ISO 3166-1 alpha-2 country code
        city: City name (optional)
        latitude: Latitude coordinate (optional)
        longitude: Longitude coordinate (optional)
    """

    country_code: str = Field(
        ...,
        min_length=2,
        max_length=2,
        description="ISO 3166-1 alpha-2 country code",
        examples=["US", "GB", "FR", "NG"],
    )

    city: Optional[str] = Field(
        default=None,
        min_length=1,
        max_length=100,
        description="City name",
        examples=["New York", "London", "Paris"],
    )

    latitude: Optional[float] = Field(
        default=None,
        ge=-90.0,
        le=90.0,
        description="Latitude coordinate",
        examples=[40.7128, 51.5074],
    )

    longitude: Optional[float] = Field(
        default=None,
        ge=-180.0,
        le=180.0,
        description="Longitude coordinate",
        examples=[-74.0060, -0.1278],
    )

    model_config = ConfigDict(
        json_schema_extra={
            "example": {
                "country_code": "US",
                "city": "San Francisco",
                "latitude": 37.7749,
                "longitude": -122.4194,
            }
        }
    )

    @field_validator("country_code")
    @classmethod
    def validate_country_code(cls, v: str) -> str:
        """
        Validate country code is uppercase and alphabetic.
        """
        v = v.upper()
        if not v.isalpha():
            raise ValueError("Country code must be alphabetic")
        return v

    @model_validator(mode="after")
    def validate_coordinates(self) -> "LocationInfo":
        """
        Validate that if one coordinate is provided, both must be provided.

        This is a model-level validator that checks multiple fields together.
        """
        if (self.latitude is None) != (self.longitude is None):
            raise ValueError("Both latitude and longitude must be provided together")
        return self


# ==========================================
# MAIN REQUEST SCHEMA
# ==========================================

class TransactionCheckRequest(BaseModel):
    """
    Complete fraud check request payload.

    This is the main request schema sent to POST /fraud/check endpoint.
    Contains all information needed to perform comprehensive fraud analysis.

    Attributes:
        transaction_id: Unique transaction identifier
        amount: Transaction amount in USD
        currency: Currency code (default: USD)
        transaction_type: Type of transaction
        industry: Merchant industry
        timestamp: Transaction timestamp
        merchant_id: Unique merchant identifier
        device: Device information
        user: User information
        location: Location information
        metadata: Additional custom data (optional)
    """

    transaction_id: str = Field(
        ...,
        min_length=1,
        max_length=100,
        description="Unique transaction identifier",
        examples=["txn_1234567890", "ord_abc123"],
    )

    amount: float = Field(
        ...,
        gt=0,  # Greater than 0 (strictly positive)
        le=1000000.0,  # Max $1M per transaction
        description="Transaction amount in USD",
        examples=[99.99, 1500.00, 25000.50],
    )

    currency: str = Field(
        default="USD",
        min_length=3,
        max_length=3,
        description="ISO 4217 currency code",
        examples=["USD", "EUR", "GBP"],
    )

    transaction_type: TransactionType = Field(
        ...,
        description="Type of transaction",
        examples=["PAYMENT"],
    )

    industry: Industry = Field(
        ...,
        description="Merchant industry classification",
        examples=["RETAIL"],
    )

    timestamp: datetime = Field(
        ...,
        description="Transaction timestamp (ISO 8601)",
        examples=["2024-01-15T14:30:00Z"],
    )

    merchant_id: str = Field(
        ...,
        min_length=1,
        max_length=100,
        description="Unique merchant identifier",
        examples=["merchant_abc123"],
    )

    # Nested objects
    device: DeviceInfo = Field(
        ...,
        description="Device information",
    )

    user: UserInfo = Field(
        ...,
        description="User information",
    )

    location: LocationInfo = Field(
        ...,
        description="Geographic location information",
    )

    # Optional metadata
    metadata: Optional[dict[str, Any]] = Field(
        default=None,
        description="Additional custom metadata",
        examples=[{"order_id": "12345", "product_category": "electronics"}],
    )

    model_config = ConfigDict(
        json_schema_extra={
            "example": {
                "transaction_id": "txn_1234567890",
                "amount": 150.00,
                "currency": "USD",
                "transaction_type": "PAYMENT",
                "industry": "RETAIL",
                "timestamp": "2024-01-15T14:30:00Z",
                "merchant_id": "merchant_abc123",
                "device": {
                    "device_id": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0",
                    "ip_address": "203.0.113.42",
                    "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
                    "is_vpn": False,
                    "is_tor": False,
                },
                "user": {
                    "user_id": "user_abc123",
                    "email": "john.doe@example.com",
                    "account_age_days": 45,
                    "is_verified": True,
                    "previous_fraud_reports": 0,
                },
                "location": {
                    "country_code": "US",
                    "city": "San Francisco",
                    "latitude": 37.7749,
                    "longitude": -122.4194,
                },
                "metadata": {
                    "order_id": "ord_12345",
                    "product_category": "electronics",
                },
            }
        }
    )

    @field_validator("currency")
    @classmethod
    def validate_currency(cls, v: str) -> str:
        """Validate currency code is uppercase and 3 characters."""
        v = v.upper()
        if not v.isalpha():
            raise ValueError("Currency code must be alphabetic")
        return v

    @field_validator("timestamp")
    @classmethod
    def validate_timestamp(cls, v: datetime) -> datetime:
        """
        Validate timestamp is not in the future.

        Transactions can't happen in the future (basic sanity check).
        """
        if v > datetime.now(v.tzinfo):
            raise ValueError("Timestamp cannot be in the future")
        return v

    @field_validator("amount")
    @classmethod
    def validate_amount_precision(cls, v: float) -> float:
        """
        Validate amount has at most 2 decimal places.

        Money amounts should have at most 2 decimal places (cents).
        """
        # Round to 2 decimal places
        rounded = round(v, 2)
        if abs(v - rounded) > 0.001:  # Small tolerance for float precision
            raise ValueError("Amount must have at most 2 decimal places")
        return rounded


# ==========================================
# RESPONSE SCHEMAS
# ==========================================

class FraudFlag(BaseModel):
    """
    Individual fraud flag detected during analysis.

    Each flag represents a specific fraud indicator found in the transaction.
    Multiple flags can be raised for a single transaction.

    Attributes:
        flag_type: Type of fraud flag
        severity: Severity level (LOW, MEDIUM, HIGH, CRITICAL)
        description: Human-readable explanation
        score_impact: How many points this flag adds to risk score
    """

    flag_type: str = Field(
        ...,
        min_length=1,
        max_length=100,
        description="Type of fraud flag",
        examples=["HIGH_AMOUNT", "NEW_USER", "VPN_DETECTED"],
    )

    severity: RiskLevel = Field(
        ...,
        description="Severity level of this flag",
        examples=["MEDIUM"],
    )

    description: str = Field(
        ...,
        min_length=1,
        max_length=500,
        description="Human-readable explanation of the flag",
        examples=["Transaction amount exceeds $10,000 threshold"],
    )

    score_impact: float = Field(
        ...,
        ge=0,
        le=100,
        description="Points added to risk score by this flag",
        examples=[15.0, 25.0, 50.0],
    )

    model_config = ConfigDict(
        json_schema_extra={
            "example": {
                "flag_type": "HIGH_AMOUNT",
                "severity": "HIGH",
                "description": "Transaction amount exceeds $10,000 threshold",
                "score_impact": 25.0,
            }
        }
    )


class FraudCheckResponse(BaseModel):
    """
    Complete fraud check response.

    This is the response returned from POST /fraud/check endpoint.
    Contains all fraud analysis results and recommendations.

    Attributes:
        transaction_id: Echo of the original transaction ID
        verdict: Final fraud verdict (APPROVE/REVIEW/DECLINE)
        risk_score: Overall risk score (0-100)
        risk_level: Risk level classification
        flags: List of fraud flags detected
        checked_at: Timestamp when check was performed
        processing_time_ms: Time taken to process (optional)
        recommendation: Human-readable recommendation (optional)
    """

    transaction_id: str = Field(
        ...,
        description="Transaction identifier from request",
        examples=["txn_1234567890"],
    )

    verdict: FraudVerdict = Field(
        ...,
        description="Final fraud detection verdict",
        examples=["APPROVE"],
    )

    risk_score: float = Field(
        ...,
        ge=0,
        le=100,
        description="Overall risk score (0-100)",
        examples=[25.5, 67.3, 92.1],
    )

    risk_level: RiskLevel = Field(
        ...,
        description="Risk level classification",
        examples=["LOW"],
    )

    flags: list[FraudFlag] = Field(
        default_factory=list,
        description="List of fraud flags detected",
        examples=[[]],
    )

    checked_at: datetime = Field(
        default_factory=datetime.now,
        description="Timestamp when fraud check was performed",
        examples=["2024-01-15T14:30:01Z"],
    )

    processing_time_ms: Optional[float] = Field(
        default=None,
        ge=0,
        description="Processing time in milliseconds",
        examples=[45.2, 123.7],
    )

    recommendation: Optional[str] = Field(
        default=None,
        max_length=1000,
        description="Human-readable recommendation",
        examples=["Transaction appears safe. Approve for processing."],
    )

    model_config = ConfigDict(
        json_schema_extra={
            "example": {
                "transaction_id": "txn_1234567890",
                "verdict": "APPROVE",
                "risk_score": 15.0,
                "risk_level": "LOW",
                "flags": [],
                "checked_at": "2024-01-15T14:30:01Z",
                "processing_time_ms": 42.5,
                "recommendation": "Transaction appears safe. Approve for processing.",
            }
        }
    )

    @model_validator(mode="after")
    def validate_verdict_matches_risk(self) -> "FraudCheckResponse":
        """
        Validate that verdict matches risk level.

        Business rules:
        - LOW risk → APPROVE
        - MEDIUM risk → APPROVE or REVIEW
        - HIGH risk → REVIEW
        - CRITICAL risk → DECLINE
        """
        if self.risk_level == RiskLevel.CRITICAL and self.verdict == FraudVerdict.APPROVE:
            raise ValueError("CRITICAL risk cannot have APPROVE verdict")

        if self.risk_level == RiskLevel.LOW and self.verdict == FraudVerdict.DECLINE:
            raise ValueError("LOW risk cannot have DECLINE verdict")

        return self
```

---

## Data Validation Deep Dive

### Understanding Field Constraints

**Pydantic Field Constraints:**

```python
from pydantic import BaseModel, Field

class Example(BaseModel):
    # Numeric constraints
    age: int = Field(ge=0, le=150)           # 0 <= age <= 150
    price: float = Field(gt=0, le=1000000)   # 0 < price <= 1,000,000
    discount: float = Field(ge=0, lt=1)      # 0 <= discount < 1

    # String constraints
    name: str = Field(min_length=1, max_length=100)
    email: str = Field(pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$")  # Regex pattern

    # Default values
    status: str = Field(default="pending")
    created_at: datetime = Field(default_factory=datetime.now)
    tags: list[str] = Field(default_factory=list)
```

**Constraint Types:**
- `gt` - Greater than (exclusive)
- `ge` - Greater than or equal (inclusive)
- `lt` - Less than (exclusive)
- `le` - Less than or equal (inclusive)
- `min_length` - Minimum string/list length
- `max_length` - Maximum string/list length
- `pattern` - Regex pattern (for strings)

### Custom Validators

**Field Validators** - Validate individual fields:

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    email: str
    age: int

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        """Custom email validation."""
        if "@" not in v:
            raise ValueError("Must contain @")
        if not v.endswith(".com"):
            raise ValueError("Must end with .com")
        return v.lower()

    @field_validator("age")
    @classmethod
    def validate_age(cls, v: int) -> int:
        """Validate age is reasonable."""
        if v < 0:
            raise ValueError("Age cannot be negative")
        if v > 150:
            raise ValueError("Age too high")
        return v
```

**Model Validators** - Validate multiple fields together:

```python
from pydantic import BaseModel, model_validator

class DateRange(BaseModel):
    start_date: datetime
    end_date: datetime

    @model_validator(mode="after")
    def validate_date_range(self) -> "DateRange":
        """Ensure end_date is after start_date."""
        if self.end_date <= self.start_date:
            raise ValueError("end_date must be after start_date")
        return self
```

### Validation Error Handling

```python
from pydantic import ValidationError
from app.models.schemas import TransactionCheckRequest

# Invalid data
invalid_data = {
    "transaction_id": "txn_123",
    "amount": -100,  # ❌ Negative amount
    "currency": "US",  # ❌ Should be 3 chars
    "transaction_type": "INVALID",  # ❌ Not a valid enum
    # Missing required fields...
}

try:
    request = TransactionCheckRequest(**invalid_data)
except ValidationError as e:
    print(e.json())  # Pretty JSON error output

    # Get individual errors
    for error in e.errors():
        print(f"Field: {error['loc']}")
        print(f"Error: {error['msg']}")
        print(f"Type: {error['type']}")
```

---

## Testing Your Schemas

The best way to test Pydantic schemas is through actual API endpoints. FastAPI automatically validates incoming requests using your schemas and generates interactive documentation.

### Using Swagger UI (Interactive Testing)

1. **Start your FastAPI server:**
   ```bash
   uvicorn app.main:app --reload
   ```

2. **Open Swagger UI in your browser:**
   ```
   http://localhost:8000/docs
   ```

3. **Test schema validation visually:**
   - Click on any endpoint that uses your schemas
   - Click "Try it out"
   - Enter test data in the request body
   - Click "Execute"
   - View validation errors or successful responses

**Benefits:**
- ✅ No test code to write
- ✅ Visual interface shows all schema fields
- ✅ Instant validation feedback
- ✅ See exact error messages
- ✅ Auto-generated from your Pydantic models

### Using cURL (Command Line Testing)

Test validation with valid data:

```bash
# Test with valid transaction request
curl -X POST http://localhost:8000/api/v1/check-fraud \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "txn_12345",
    "amount": 150.00,
    "currency": "USD",
    "transaction_type": "payment",
    "industry": "retail",
    "timestamp": "2024-01-15T10:30:00Z",
    "merchant_id": "merchant_001",
    "device": {
      "device_id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "ip_address": "192.168.1.100",
      "user_agent": "Mozilla/5.0"
    },
    "user": {
      "user_id": "user_123",
      "email": "test@example.com",
      "account_age_days": 45
    },
    "location": {
      "country_code": "US",
      "city": "San Francisco"
    }
  }'
```

Test validation with invalid data (triggers errors):

```bash
# Test with invalid email (should return validation error)
curl -X POST http://localhost:8000/api/v1/check-fraud \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "txn_12345",
    "amount": 150.00,
    "currency": "USD",
    "transaction_type": "payment",
    "industry": "retail",
    "timestamp": "2024-01-15T10:30:00Z",
    "merchant_id": "merchant_001",
    "device": {
      "device_id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "ip_address": "192.168.1.100",
      "user_agent": "Mozilla/5.0"
    },
    "user": {
      "user_id": "user_123",
      "email": "invalid-email",
      "account_age_days": 45
    },
    "location": {
      "country_code": "US"
    }
  }'
```

Expected validation error response:
```json
{
  "detail": [
    {
      "type": "value_error",
      "loc": ["body", "user", "email"],
      "msg": "value is not a valid email address",
      "input": "invalid-email"
    }
  ]
}
```

### Validation Scenarios to Test

**1. Test negative amounts:**
```bash
curl -X POST http://localhost:8000/api/v1/check-fraud \
  -H "Content-Type: application/json" \
  -d '{"amount": -100, ...}'  # Should fail: amount must be > 0
```

**2. Test invalid device ID:**
```bash
curl -X POST http://localhost:8000/api/v1/check-fraud \
  -H "Content-Type: application/json" \
  -d '{"device": {"device_id": "short", ...}}'  # Should fail: too short
```

**3. Test missing required fields:**
```bash
curl -X POST http://localhost:8000/api/v1/check-fraud \
  -H "Content-Type: application/json" \
  -d '{}'  # Should fail: missing required fields
```

**4. Test mismatched coordinates:**
```bash
curl -X POST http://localhost:8000/api/v1/check-fraud \
  -H "Content-Type: application/json" \
  -d '{"location": {"latitude": 37.77}}'  # Should fail: longitude required if latitude present
```

---

## Advanced Pydantic Features

### Optional vs Required Fields

```python
from typing import Optional
from pydantic import BaseModel, Field

class User(BaseModel):
    # Required field (no default)
    user_id: str

    # Required with validation
    email: str = Field(..., min_length=5)

    # Optional with None default
    middle_name: Optional[str] = None

    # Optional with custom default
    status: str = Field(default="active")

    # Optional with factory function
    created_at: datetime = Field(default_factory=datetime.now)

    # Optional list (empty by default)
    tags: list[str] = Field(default_factory=list)
```

### Model Serialization

```python
from app.models.schemas import DeviceInfo

device = DeviceInfo(
    device_id="a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
    ip_address="192.168.1.1",
    user_agent="Mozilla/5.0",
)

# Convert to dict
device_dict = device.model_dump()
# {'device_id': '...', 'ip_address': '...', ...}

# Convert to JSON string
device_json = device.model_dump_json()
# '{"device_id":"...","ip_address":"...",...}'

# Exclude None values
device_dict = device.model_dump(exclude_none=True)

# Include only specific fields
device_dict = device.model_dump(include={'device_id', 'ip_address'})

# Exclude specific fields
device_dict = device.model_dump(exclude={'user_agent'})
```

### Field Aliases

```python
from pydantic import BaseModel, Field

class Transaction(BaseModel):
    # API uses 'txn_id', but we use 'transaction_id' internally
    transaction_id: str = Field(alias="txn_id")

    # Accept both snake_case and camelCase
    user_name: str = Field(alias="userName")

# Create with alias
txn = Transaction(txn_id="123", userName="John")

# Access with field name
print(txn.transaction_id)  # "123"
print(txn.user_name)  # "John"

# Serialize with alias
txn.model_dump(by_alias=True)
# {'txn_id': '123', 'userName': 'John'}
```

### Schema Generation

```python
from app.models.schemas import TransactionCheckRequest

# Get JSON Schema (for OpenAPI)
schema = TransactionCheckRequest.model_json_schema()

import json
print(json.dumps(schema, indent=2))
```

---

## Troubleshooting

### Common Validation Errors

**1. "Field required"**
```python
# ❌ Missing required field
TransactionCheckRequest(transaction_id="123")
# Error: Field 'amount' required

# ✅ Provide all required fields
TransactionCheckRequest(
    transaction_id="123",
    amount=100.0,
    # ... all other required fields
)
```

**2. "Input should be greater than 0"**
```python
# ❌ Violates constraint
amount: float = Field(gt=0)
TransactionCheckRequest(amount=-100)

# ✅ Use positive value
TransactionCheckRequest(amount=100)
```

**3. "Invalid enum value"**
```python
# ❌ Invalid enum
transaction_type="INVALID_TYPE"

# ✅ Use valid enum value
from app.models.schemas import TransactionType
transaction_type=TransactionType.PAYMENT
```

### Import Errors

**Problem:** `ModuleNotFoundError: No module named 'app'`

**Solution:**
```bash
# Make sure you're in the project root
cd /home/user/sentinel-api

# Run with PYTHONPATH
PYTHONPATH=/home/user/sentinel-api python test_schemas_valid.py

# Or use python -m
python -m pytest
```

### Type Hint Errors

**Problem:** IDE shows type errors

**Solution:**
```bash
# Install type stubs
pip install types-python-dateutil

# Run mypy for type checking
pip install mypy
mypy app/
```

---

## Next Steps

### Day 4 Preview: Fraud Detection Logic

Tomorrow we'll implement the actual fraud detection algorithms:

```python
# Sneak peek at Day 4
class FraudDetector:
    def check_transaction(self, request: TransactionCheckRequest) -> FraudCheckResponse:
        # 1. Calculate risk score
        score = 0.0
        flags = []

        # 2. Check amount thresholds
        if request.amount > settings.HIGH_RISK_AMOUNT_THRESHOLD:
            score += 25.0
            flags.append(FraudFlag(...))

        # 3. Check velocity (rate limiting)
        # 4. Check device fingerprint
        # 5. Check geographic risk
        # 6. Calculate final verdict

        return FraudCheckResponse(...)
```

### Quick Reference Commands

```bash
# Test schemas
python test_schemas_valid.py
python test_schemas_invalid.py

# Start API server
uvicorn app.main:app --reload

# Test in browser
open http://localhost:8000/docs

# Run validation manually
python -c "from app.models.schemas import DeviceInfo; print(DeviceInfo.model_json_schema())"
```

### Key Takeaways

✅ **Pydantic provides automatic data validation**
✅ **Type hints enable IDE autocomplete and error checking**
✅ **Field constraints (gt, ge, min_length) enforce business rules**
✅ **Custom validators implement complex validation logic**
✅ **Enums ensure type safety for fixed value sets**
✅ **Nested models allow complex data structures**
✅ **JSON schema generation creates automatic API docs**

---

## Summary

Today you built a production-ready data validation layer for Sentinel:

1. ✅ **Core Configuration** - Centralized settings with environment variables
2. ✅ **Pydantic Schemas** - 8 schemas with comprehensive validation
3. ✅ **Enums** - Type-safe enumerations for industries, types, verdicts
4. ✅ **Custom Validators** - Business logic validation
5. ✅ **Test Scripts** - Validation testing framework

**Files Created:**
- `/home/user/sentinel-api/app/core/__init__.py`
- `/home/user/sentinel-api/app/core/config.py`
- `/home/user/sentinel-api/app/models/__init__.py`
- `/home/user/sentinel-api/app/models/schemas.py`
- `/home/user/sentinel-api/test_schemas_valid.py`
- `/home/user/sentinel-api/test_schemas_invalid.py`

**Next Session:** Day 4 - Fraud Detection Logic Implementation

---

**Questions? Issues?**
- Check validation errors carefully - Pydantic gives detailed error messages
- Use Swagger UI at `/docs` to test schemas interactively
- Review examples in this guide for common patterns

**Happy coding! 🚀**
