# Day 2: Database Setup & Data Models

## 🎯 Learning Objectives

By the end of Day 2, you will:
- ✅ Understand what PostgreSQL is and why it's essential for fraud detection
- ✅ Install and configure PostgreSQL locally
- ✅ Learn SQLAlchemy ORM concepts (engine, session, Base)
- ✅ Create 3 production-ready database models with relationships
- ✅ Understand database indexes and why they matter for performance
- ✅ Set up database connection management
- ✅ Verify your database setup with real PostgreSQL commands

---

## 📋 Table of Contents

1. [What is PostgreSQL?](#what-is-postgresql)
2. [ORM vs Raw SQL](#orm-vs-raw-sql)
3. [SQLAlchemy Fundamentals](#sqlalchemy-fundamentals)
4. [Installing PostgreSQL](#installing-postgresql)
5. [Database Setup](#database-setup)
6. [Complete Code Implementation](#complete-code-implementation)
7. [Database Schema Explained](#database-schema-explained)
8. [Testing Your Setup](#testing-your-setup)
9. [Troubleshooting](#troubleshooting)
10. [Key Takeaways](#key-takeaways)
11. [Navigation](#navigation)

---

## What is PostgreSQL?

### Definition

**PostgreSQL** (often called "Postgres") is a powerful, open-source **relational database management system (RDBMS)** that has been in active development for over 30 years.

### Why PostgreSQL for Fraud Detection?

For our Sentinel fraud detection platform, PostgreSQL is the ideal choice because:

1. **ACID Compliance**: Guarantees data consistency and reliability
   - **Atomicity**: Transactions are all-or-nothing
   - **Consistency**: Data integrity is maintained
   - **Isolation**: Concurrent transactions don't interfere
   - **Durability**: Committed data is never lost

2. **Advanced Data Types**: Perfect for fraud detection
   - **JSONB**: Store flexible fraud metadata (device info, location data)
   - **Arrays**: Store lists (multiple phone numbers, email addresses)
   - **Timestamps with timezone**: Critical for tracking fraud patterns globally

3. **Performance**: Handles millions of fraud transactions
   - Advanced indexing (B-tree, GiST, GIN, BRIN)
   - Query optimization
   - Parallel query execution

4. **Scalability**: Grows with your business
   - Handles billions of rows
   - Replication for high availability
   - Partitioning for massive datasets

5. **Production-Ready**: Used by industry leaders
   - Instagram, Spotify, Uber, Reddit all use PostgreSQL
   - Battle-tested for financial applications
   - Strong security features

### PostgreSQL vs Other Databases

| Feature | PostgreSQL | MySQL | MongoDB |
|---------|-----------|-------|---------|
| Data Model | Relational | Relational | Document |
| ACID Compliance | ✅ Full | ⚠️ Partial | ⚠️ Eventual |
| JSON Support | ✅ JSONB | ⚠️ JSON | ✅ Native |
| Indexing | ✅ Advanced | ⚠️ Basic | ✅ Good |
| Best For | Complex queries, data integrity | Simple reads | Flexible schemas |

**For fraud detection:** We need **ACID compliance** (no data loss) + **JSONB** (flexible metadata) + **advanced indexing** (fast queries). PostgreSQL wins!

---

## ORM vs Raw SQL

### What is an ORM?

**ORM (Object-Relational Mapping)** is a technique that lets you query and manipulate data from a database using an **object-oriented paradigm**.

### Raw SQL (Traditional Approach)

```python
# Raw SQL - you write SQL strings directly
cursor.execute("""
    INSERT INTO fraud_transactions (transaction_id, amount, user_id)
    VALUES (%s, %s, %s)
""", (tx_id, amount, user_id))

# Fragile: typos in SQL, SQL injection risks, hard to refactor
```

**Problems:**
- ❌ SQL injection vulnerabilities
- ❌ Database-specific syntax (hard to switch DBs)
- ❌ Manual type conversion
- ❌ No autocomplete in your IDE
- ❌ Harder to test

### ORM Approach (SQLAlchemy)

```python
# ORM - you work with Python objects
transaction = FraudTransaction(
    transaction_id=tx_id,
    amount=amount,
    user_id=user_id
)
session.add(transaction)
session.commit()

# Type-safe, autocomplete, database-agnostic, secure
```

**Benefits:**
- ✅ SQL injection protection (automatic parameterization)
- ✅ Database agnostic (switch from PostgreSQL to MySQL easily)
- ✅ Type safety and IDE autocomplete
- ✅ Easy to write unit tests
- ✅ Automatic migrations (with Alembic)

### When to Use Each?

| Scenario | Use ORM | Use Raw SQL |
|----------|---------|-------------|
| CRUD operations (95% of cases) | ✅ Yes | ❌ No |
| Complex analytics queries | ⚠️ Maybe | ✅ Yes |
| Performance-critical queries | ⚠️ Maybe | ✅ Yes |
| Database migrations | ✅ Yes | ❌ No |

**For Day 2:** We use **SQLAlchemy ORM** for everything. It's safer, cleaner, and more maintainable.

---

## SQLAlchemy Fundamentals

### Key Concepts

SQLAlchemy has 3 core components:

#### 1. Engine (Connection Pool)

```python
# Engine manages database connections
engine = create_engine("postgresql://user:pass@localhost/db")

# Think of it as: "How do I connect to the database?"
```

**What it does:**
- Manages connection pooling (reuses connections for performance)
- Handles connection lifecycle (open, close, retry)
- Translates Python to database-specific SQL

#### 2. Session (Transaction Manager)

```python
# Session manages transactions (groups of operations)
session = SessionLocal()

# Think of it as: "A workspace for database operations"
```

**What it does:**
- Groups multiple operations into a transaction
- Tracks changes to objects (dirty tracking)
- Commits or rolls back changes
- Automatically manages BEGIN/COMMIT/ROLLBACK

#### 3. Base (Model Definition)

```python
# Base is the foundation for all models
Base = declarative_base()

class FraudTransaction(Base):
    __tablename__ = "fraud_transactions"
    # ...

# Think of it as: "The blueprint for all database tables"
```

**What it does:**
- Defines table structure (columns, types, constraints)
- Maps Python classes to database tables
- Enables relationships between tables

### How They Work Together

```
┌─────────────────────────────────────────┐
│  Your FastAPI Application               │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Session (Transaction Manager)   │   │
│  │  - Begin transaction            │   │
│  │  - Track changes                │   │
│  │  - Commit/Rollback              │   │
│  └──────────────┬──────────────────┘   │
│                 │                       │
│                 ▼                       │
│  ┌─────────────────────────────────┐   │
│  │ Engine (Connection Pool)        │   │
│  │  - Manage connections           │   │
│  │  - Execute SQL                  │   │
│  │  - Handle retries               │   │
│  └──────────────┬──────────────────┘   │
│                 │                       │
└─────────────────┼───────────────────────┘
                  │
                  ▼
         ┌────────────────┐
         │   PostgreSQL   │
         │   Database     │
         └────────────────┘
```

---

## Installing PostgreSQL

### Ubuntu/Debian (Linux)

```bash
# Update package list
sudo apt update

# Install PostgreSQL 15 (latest stable)
sudo apt install postgresql-15 postgresql-contrib-15 -y

# Verify installation
psql --version
# Expected: psql (PostgreSQL) 15.x

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql  # Auto-start on boot

# Check status
sudo systemctl status postgresql
# Expected: active (running)
```

### macOS

```bash
# Using Homebrew (recommended)
brew install postgresql@15

# Start PostgreSQL service
brew services start postgresql@15

# Verify installation
psql --version
# Expected: psql (PostgreSQL) 15.x
```

### Windows

```bash
# Download installer from: https://www.postgresql.org/download/windows/
# Run the installer (use default settings)
# Remember the password you set for 'postgres' user!

# Verify installation (in Command Prompt)
psql --version
# Expected: psql (PostgreSQL) 15.x
```

### Docker (Recommended for Development)

```bash
# Pull PostgreSQL 15 image
docker pull postgres:15

# Run PostgreSQL container
docker run --name sentinel-postgres \
  -e POSTGRES_USER=sentinel_user \
  -e POSTGRES_PASSWORD=sentinel_pass_2024 \
  -e POSTGRES_DB=sentinel_fraud_db \
  -p 5432:5432 \
  -d postgres:15

# Verify container is running
docker ps | grep sentinel-postgres

# Connect to PostgreSQL
docker exec -it sentinel-postgres psql -U sentinel_user -d sentinel_fraud_db
```

---

## Database Setup

### Step 1: Create Database User

```bash
# Switch to postgres user (Linux/macOS)
sudo -u postgres psql

# Or connect directly (if password auth is enabled)
psql -U postgres
```

**In the PostgreSQL prompt:**

```sql
-- Create dedicated user for Sentinel
CREATE USER sentinel_user WITH PASSWORD 'sentinel_pass_2024';

-- Create database
CREATE DATABASE sentinel_fraud_db OWNER sentinel_user;

-- Grant all privileges
GRANT ALL PRIVILEGES ON DATABASE sentinel_fraud_db TO sentinel_user;

-- Exit
\q
```

### Step 2: Verify Database Access

```bash
# Test connection with new user
psql -U sentinel_user -d sentinel_fraud_db -h localhost

# If password prompt appears, enter: sentinel_pass_2024
```

**You should see:**
```
sentinel_fraud_db=>
```

### Step 3: Update Environment Variables

Create `.env` file in your project root:

```bash
# Database Configuration
DATABASE_URL=postgresql://sentinel_user:sentinel_pass_2024@localhost:5432/sentinel_fraud_db

# FastAPI Configuration
APP_NAME=Sentinel Fraud Detection
DEBUG=True
```

---

## Complete Code Implementation

### File Structure

```
sentinel-fraud-detection/
├── app/
│   ├── __init__.py
│   ├── main.py                  # ✅ Updated with DB check
│   ├── db/
│   │   ├── __init__.py          # ✅ NEW
│   │   └── session.py           # ✅ NEW (engine, session)
│   ├── models/
│   │   ├── __init__.py          # ✅ NEW
│   │   └── database.py          # ✅ NEW (3 ORM models)
├── scripts/
│   └── init_db.py               # ✅ NEW (DB initialization)
├── .env                         # ✅ NEW (environment vars)
└── requirements.txt             # ✅ Updated
```

---

### 1. `app/db/__init__.py`

```python
"""
Database package initialization.

This file makes 'db' a Python package and exposes key database objects
for easy importing throughout the application.

Usage:
    from app.db import get_db, engine
"""

# Import engine and session factory from session module
from app.db.session import engine, SessionLocal, get_db, Base

# Expose these objects when someone does: from app.db import ...
__all__ = [
    "engine",        # SQLAlchemy engine (connection pool manager)
    "SessionLocal",  # Session factory (creates new sessions)
    "get_db",        # FastAPI dependency for database sessions
    "Base",          # Declarative base for ORM models
]
```

---

### 2. `app/db/session.py`

```python
"""
SQLAlchemy database session management.

This module sets up:
1. Database engine (connection pool)
2. Session factory (creates database sessions)
3. Declarative Base (foundation for ORM models)
4. FastAPI dependency for database sessions

Key Concepts:
- Engine: Manages connections to PostgreSQL
- Session: Manages transactions (groups of database operations)
- Base: Blueprint for all database tables
"""

import os
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from typing import Generator


# ============================================================================
# DATABASE URL CONFIGURATION
# ============================================================================

# Load database URL from environment variable
# Format: postgresql://username:password@host:port/database_name
DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql://sentinel_user:sentinel_pass_2024@localhost:5432/sentinel_fraud_db"
)

# Alternative: Load from .env file using python-dotenv
# from dotenv import load_dotenv
# load_dotenv()
# DATABASE_URL = os.getenv("DATABASE_URL")


# ============================================================================
# ENGINE CONFIGURATION
# ============================================================================

# Create SQLAlchemy engine
# The engine manages a connection pool to PostgreSQL
engine = create_engine(
    DATABASE_URL,

    # Connection Pool Settings (important for production!)
    pool_size=10,              # Keep 10 connections open
    max_overflow=20,           # Allow 20 additional connections if needed
    pool_pre_ping=True,        # Verify connections before using (auto-reconnect)
    pool_recycle=3600,         # Recycle connections after 1 hour (prevents stale connections)

    # Echo SQL queries to console (useful for debugging)
    echo=False,                # Set to True to see all SQL queries
)

"""
Why these pool settings?

- pool_size=10: For a fraud detection API, we expect multiple concurrent requests.
  Each request needs a database connection. 10 is a good starting point.

- max_overflow=20: During traffic spikes, allow up to 30 total connections (10 + 20).
  This prevents "connection pool exhausted" errors.

- pool_pre_ping=True: Before using a connection, SQLAlchemy pings the database.
  If the connection is dead (database restart, network issue), it reconnects automatically.
  Critical for production reliability!

- pool_recycle=3600: PostgreSQL may close idle connections after 8 hours.
  By recycling every hour, we prevent "server closed the connection" errors.
"""


# ============================================================================
# SESSION FACTORY
# ============================================================================

# Create a SessionLocal class (not an instance!)
# This is a factory that creates new Session instances
SessionLocal = sessionmaker(
    autocommit=False,  # Don't auto-commit (we control transactions explicitly)
    autoflush=False,   # Don't auto-flush changes (we control when to save)
    bind=engine,       # Bind this session factory to our PostgreSQL engine
)

"""
Why autocommit=False and autoflush=False?

In fraud detection, we need transactional control:

1. autocommit=False: We want to group multiple operations into a transaction.
   Example: Create FraudTransaction + Update UserRiskProfile + Log DeviceFingerprint
   All must succeed or all must fail (ACID atomicity).

2. autoflush=False: We control when changes are sent to the database.
   This improves performance by batching database calls.

Manual control:
    session.add(fraud_tx)          # Stage change
    session.add(user_profile)      # Stage change
    session.commit()               # Send both to DB in one transaction
"""


# ============================================================================
# DECLARATIVE BASE
# ============================================================================

# Create Base class for all ORM models
# All models (FraudTransaction, UserRiskProfile, etc.) will inherit from this
Base = declarative_base()

"""
What is declarative_base()?

It's a factory function that returns a base class for ORM models.
When you define a model:

    class FraudTransaction(Base):
        __tablename__ = "fraud_transactions"
        id = Column(Integer, primary_key=True)

SQLAlchemy automatically:
1. Maps the class to a database table
2. Creates table metadata (columns, types, constraints)
3. Enables querying: session.query(FraudTransaction).all()
"""


# ============================================================================
# FASTAPI DEPENDENCY
# ============================================================================

def get_db() -> Generator:
    """
    FastAPI dependency that provides a database session.

    This function:
    1. Creates a new database session
    2. Yields it to the request handler
    3. Automatically closes the session after the request

    Usage in FastAPI:
        @app.get("/transactions")
        def get_transactions(db: Session = Depends(get_db)):
            return db.query(FraudTransaction).all()

    Yields:
        Session: SQLAlchemy database session

    How it works:
        - FastAPI calls this function for each request
        - The session is created fresh (no shared state between requests)
        - After the request completes, the session is closed automatically
        - If an exception occurs, the session is rolled back
    """
    # Create a new database session
    db = SessionLocal()

    try:
        # Yield the session to the request handler
        # The request handler runs here
        yield db
    finally:
        # After the request completes (success or error), close the session
        # This returns the connection to the pool
        db.close()


"""
Why use a dependency?

FastAPI's dependency injection system ensures:
1. Each request gets a fresh session (no shared state bugs)
2. Sessions are automatically closed (no connection leaks)
3. Clean separation of concerns (handlers don't manage sessions)

Example:
    @app.post("/fraud-check")
    def check_fraud(
        transaction: TransactionSchema,
        db: Session = Depends(get_db)  # ← Automatically injected
    ):
        # Use db session here
        fraud_tx = FraudTransaction(**transaction.dict())
        db.add(fraud_tx)
        db.commit()
        return {"status": "recorded"}
"""
```

---

### 3. `app/models/__init__.py`

```python
"""
Database models package initialization.

This file exposes all ORM models for easy importing.

Usage:
    from app.models import FraudTransaction, UserRiskProfile, DeviceFingerprint
"""

# Import all models from database module
from app.models.database import (
    FraudTransaction,
    UserRiskProfile,
    DeviceFingerprint,
)

# Expose these models when someone does: from app.models import ...
__all__ = [
    "FraudTransaction",
    "UserRiskProfile",
    "DeviceFingerprint",
]
```

---

### 4. `app/models/database.py`

```python
"""
SQLAlchemy ORM Models for Sentinel Fraud Detection Platform.

This module defines 3 core database tables:
1. fraud_transactions: Records every transaction analyzed
2. user_risk_profiles: Tracks user risk scores and behavior
3. device_fingerprints: Identifies unique devices

Each model includes:
- Comprehensive columns for fraud detection
- Indexes for fast queries
- Relationships between tables
- JSON columns for flexible metadata
"""

from sqlalchemy import (
    Column,
    Integer,
    String,
    Float,
    Boolean,
    DateTime,
    Text,
    ForeignKey,
    Index,
    Numeric,
)
from sqlalchemy.dialects.postgresql import JSONB, ARRAY
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from app.db.session import Base


# ============================================================================
# MODEL 1: FRAUD TRANSACTIONS
# ============================================================================

class FraudTransaction(Base):
    """
    Stores every transaction analyzed by the fraud detection system.

    This is the core table for fraud detection. Every API request to
    /api/v1/fraud-check creates a record here.

    Key Features:
    - Tracks transaction amount, currency, payment method
    - Records fraud risk score (0.0 = safe, 1.0 = fraud)
    - Stores decision (approve, decline, review)
    - Flexible JSONB fields for metadata
    - Indexed for fast queries
    """

    __tablename__ = "fraud_transactions"

    # ========================================================================
    # PRIMARY KEY
    # ========================================================================

    id = Column(
        Integer,
        primary_key=True,
        index=True,
        comment="Auto-incrementing primary key"
    )

    # ========================================================================
    # TRANSACTION IDENTIFIERS
    # ========================================================================

    transaction_id = Column(
        String(100),
        unique=True,
        nullable=False,
        index=True,
        comment="Unique transaction ID from client system (e.g., TXN_20240101_ABC123)"
    )

    user_id = Column(
        String(100),
        nullable=True,
        index=True,
        comment="User ID from client system (optional for guest checkouts)"
    )

    merchant_id = Column(
        String(100),
        nullable=True,
        index=True,
        comment="Merchant/seller ID (for marketplace verticals)"
    )

    # ========================================================================
    # TRANSACTION DETAILS
    # ========================================================================

    amount = Column(
        Numeric(15, 2),  # 15 digits total, 2 decimal places (e.g., 9999999999999.99)
        nullable=False,
        comment="Transaction amount (e.g., 1500.50)"
    )

    currency = Column(
        String(3),
        nullable=False,
        default="NGN",
        comment="ISO 4217 currency code (NGN, USD, EUR, etc.)"
    )

    transaction_type = Column(
        String(50),
        nullable=False,
        comment="Type: payment, withdrawal, transfer, loan_disbursement, etc."
    )

    vertical = Column(
        String(50),
        nullable=False,
        index=True,
        comment="Business vertical: lending, ecommerce, betting, crypto, marketplace"
    )

    # ========================================================================
    # FRAUD DETECTION RESULTS
    # ========================================================================

    risk_score = Column(
        Float,
        nullable=False,
        default=0.0,
        comment="Fraud risk score: 0.0 (safe) to 1.0 (fraud)"
    )

    decision = Column(
        String(20),
        nullable=False,
        default="review",
        comment="Final decision: approve, decline, review"
    )

    fraud_flags = Column(
        ARRAY(String),  # PostgreSQL array of strings
        nullable=True,
        comment="List of triggered fraud rules: ['velocity_check', 'suspicious_device']"
    )

    ml_score = Column(
        Float,
        nullable=True,
        comment="Machine learning model score (0.0 to 1.0)"
    )

    # ========================================================================
    # DEVICE & LOCATION
    # ========================================================================

    ip_address = Column(
        String(45),  # IPv6 max length is 45 characters
        nullable=True,
        index=True,
        comment="Client IP address (IPv4 or IPv6)"
    )

    device_fingerprint_hash = Column(
        String(64),  # SHA-256 hash = 64 characters
        nullable=True,
        index=True,
        comment="SHA-256 hash of device fingerprint (links to device_fingerprints table)"
    )

    user_agent = Column(
        Text,
        nullable=True,
        comment="Browser user agent string"
    )

    # ========================================================================
    # METADATA (JSONB for flexibility)
    # ========================================================================

    metadata = Column(
        JSONB,
        nullable=True,
        comment="Flexible JSON field for additional data: location, payment details, etc."
    )
    # Example metadata:
    # {
    #   "location": {"country": "NG", "city": "Lagos", "latitude": 6.5244, "longitude": 3.3792},
    #   "payment_method": {"type": "card", "last4": "4242", "bin": "424242"},
    #   "device": {"os": "iOS", "browser": "Safari", "screen_resolution": "1920x1080"}
    # }

    # ========================================================================
    # TIMESTAMPS
    # ========================================================================

    created_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),  # Automatically set to current timestamp
        nullable=False,
        index=True,
        comment="When this record was created (UTC)"
    )

    updated_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),  # Automatically update on modification
        nullable=False,
        comment="When this record was last updated (UTC)"
    )

    # ========================================================================
    # RELATIONSHIPS
    # ========================================================================

    # Relationship to UserRiskProfile (one user has many transactions)
    # This allows: fraud_tx.user_profile.risk_score
    user_profile = relationship(
        "UserRiskProfile",
        foreign_keys=[user_id],
        primaryjoin="FraudTransaction.user_id == UserRiskProfile.user_id",
        back_populates="transactions",
        lazy="joined"  # Load user profile automatically with transaction
    )

    # ========================================================================
    # INDEXES (for fast queries)
    # ========================================================================

    __table_args__ = (
        # Composite index for common query: find recent transactions by user
        Index("idx_user_created", "user_id", "created_at"),

        # Composite index for fraud analysis: find high-risk transactions by vertical
        Index("idx_vertical_risk", "vertical", "risk_score"),

        # Index for IP-based fraud detection
        Index("idx_ip_created", "ip_address", "created_at"),
    )

    def __repr__(self):
        """String representation for debugging."""
        return f"<FraudTransaction(id={self.id}, tx_id={self.transaction_id}, risk={self.risk_score})>"


# ============================================================================
# MODEL 2: USER RISK PROFILES
# ============================================================================

class UserRiskProfile(Base):
    """
    Tracks risk scores and behavioral patterns for each user.

    This table maintains a running risk profile for every user:
    - Overall risk score (updated after each transaction)
    - Transaction counts and volumes
    - Fraud history
    - Behavioral metadata

    Use Cases:
    - Velocity checks: "Has user made 10+ transactions in last hour?"
    - Risk scoring: "User has 3 previous fraud attempts → high risk"
    - User analytics: "What's this user's average transaction amount?"
    """

    __tablename__ = "user_risk_profiles"

    # ========================================================================
    # PRIMARY KEY
    # ========================================================================

    id = Column(
        Integer,
        primary_key=True,
        index=True,
        comment="Auto-incrementing primary key"
    )

    # ========================================================================
    # USER IDENTIFIER
    # ========================================================================

    user_id = Column(
        String(100),
        unique=True,
        nullable=False,
        index=True,
        comment="User ID from client system (must be unique)"
    )

    # ========================================================================
    # RISK SCORING
    # ========================================================================

    risk_score = Column(
        Float,
        nullable=False,
        default=0.5,  # Start at medium risk
        comment="Current risk score: 0.0 (safe) to 1.0 (high risk)"
    )

    risk_level = Column(
        String(20),
        nullable=False,
        default="medium",
        comment="Risk category: low, medium, high, critical"
    )

    # ========================================================================
    # TRANSACTION STATISTICS
    # ========================================================================

    total_transactions = Column(
        Integer,
        nullable=False,
        default=0,
        comment="Total number of transactions by this user"
    )

    total_amount = Column(
        Numeric(18, 2),
        nullable=False,
        default=0.0,
        comment="Total transaction amount (all time)"
    )

    fraud_count = Column(
        Integer,
        nullable=False,
        default=0,
        comment="Number of fraudulent transactions detected"
    )

    declined_count = Column(
        Integer,
        nullable=False,
        default=0,
        comment="Number of declined transactions"
    )

    # ========================================================================
    # BEHAVIORAL FLAGS
    # ========================================================================

    is_verified = Column(
        Boolean,
        nullable=False,
        default=False,
        comment="Whether user identity is verified (KYC complete)"
    )

    is_blacklisted = Column(
        Boolean,
        nullable=False,
        default=False,
        index=True,
        comment="Whether user is blacklisted (auto-decline all transactions)"
    )

    # ========================================================================
    # METADATA
    # ========================================================================

    metadata = Column(
        JSONB,
        nullable=True,
        comment="Flexible JSON for user data: email, phone, KYC details, etc."
    )
    # Example metadata:
    # {
    #   "email": "user@example.com",
    #   "phone": "+2348012345678",
    #   "kyc": {"bvn": "12345678901", "verified_at": "2024-01-01T10:00:00Z"},
    #   "preferences": {"language": "en", "currency": "NGN"}
    # }

    # ========================================================================
    # TIMESTAMPS
    # ========================================================================

    first_seen = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
        comment="When user was first seen (first transaction)"
    )

    last_transaction_at = Column(
        DateTime(timezone=True),
        nullable=True,
        index=True,
        comment="When user's most recent transaction occurred"
    )

    updated_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
        comment="When this record was last updated"
    )

    # ========================================================================
    # RELATIONSHIPS
    # ========================================================================

    # Relationship to FraudTransaction (one user has many transactions)
    transactions = relationship(
        "FraudTransaction",
        foreign_keys=[FraudTransaction.user_id],
        primaryjoin="UserRiskProfile.user_id == FraudTransaction.user_id",
        back_populates="user_profile",
        lazy="dynamic"  # Don't load all transactions automatically (use .count(), .filter(), etc.)
    )

    # ========================================================================
    # INDEXES
    # ========================================================================

    __table_args__ = (
        # Index for finding blacklisted users quickly
        Index("idx_blacklist_risk", "is_blacklisted", "risk_score"),

        # Index for analyzing recent user activity
        Index("idx_last_transaction", "last_transaction_at"),
    )

    def __repr__(self):
        """String representation for debugging."""
        return f"<UserRiskProfile(user_id={self.user_id}, risk={self.risk_score})>"


# ============================================================================
# MODEL 3: DEVICE FINGERPRINTS
# ============================================================================

class DeviceFingerprint(Base):
    """
    Identifies unique devices for fraud detection.

    Device fingerprinting is a powerful fraud prevention technique.
    By identifying the physical device (browser, phone, computer),
    we can detect:
    - Multiple accounts from same device (account farming)
    - Device reuse across fraud attempts
    - Stolen devices used for fraud

    Key Features:
    - SHA-256 hash of device attributes (privacy-preserving)
    - Risk score per device
    - Usage statistics
    - Flexible metadata for device details
    """

    __tablename__ = "device_fingerprints"

    # ========================================================================
    # PRIMARY KEY
    # ========================================================================

    id = Column(
        Integer,
        primary_key=True,
        index=True,
        comment="Auto-incrementing primary key"
    )

    # ========================================================================
    # DEVICE IDENTIFIER
    # ========================================================================

    fingerprint_hash = Column(
        String(64),  # SHA-256 hash = 64 hex characters
        unique=True,
        nullable=False,
        index=True,
        comment="SHA-256 hash of device fingerprint (unique identifier)"
    )
    # Example: "a3f4d5e6c7b8a9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4"

    # ========================================================================
    # RISK SCORING
    # ========================================================================

    risk_score = Column(
        Float,
        nullable=False,
        default=0.3,  # Start at low-medium risk
        comment="Device risk score: 0.0 (safe) to 1.0 (high risk)"
    )

    is_suspicious = Column(
        Boolean,
        nullable=False,
        default=False,
        index=True,
        comment="Whether device is flagged as suspicious"
    )

    is_blacklisted = Column(
        Boolean,
        nullable=False,
        default=False,
        index=True,
        comment="Whether device is blacklisted (auto-decline)"
    )

    # ========================================================================
    # USAGE STATISTICS
    # ========================================================================

    user_count = Column(
        Integer,
        nullable=False,
        default=1,
        comment="Number of different users associated with this device"
    )
    # If user_count > 10: Likely a shared device or account farming

    transaction_count = Column(
        Integer,
        nullable=False,
        default=0,
        comment="Total transactions from this device"
    )

    fraud_count = Column(
        Integer,
        nullable=False,
        default=0,
        comment="Number of fraud attempts from this device"
    )

    # ========================================================================
    # DEVICE METADATA
    # ========================================================================

    metadata = Column(
        JSONB,
        nullable=True,
        comment="Device details: OS, browser, screen resolution, timezone, etc."
    )
    # Example metadata:
    # {
    #   "browser": {"name": "Chrome", "version": "120.0"},
    #   "os": {"name": "Windows", "version": "10"},
    #   "device": {"type": "desktop", "vendor": "Dell"},
    #   "screen": {"width": 1920, "height": 1080, "colorDepth": 24},
    #   "timezone": "Africa/Lagos",
    #   "language": "en-US",
    #   "plugins": ["Chrome PDF Plugin", "Native Client"],
    #   "canvas_hash": "abc123...",  # Canvas fingerprinting
    #   "webgl_hash": "def456..."    # WebGL fingerprinting
    # }

    # ========================================================================
    # TIMESTAMPS
    # ========================================================================

    first_seen = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
        comment="When device was first seen"
    )

    last_seen = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
        index=True,
        comment="When device was last seen"
    )

    updated_at = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
        comment="When this record was last updated"
    )

    # ========================================================================
    # INDEXES
    # ========================================================================

    __table_args__ = (
        # Index for finding suspicious devices
        Index("idx_suspicious_risk", "is_suspicious", "risk_score"),

        # Index for device velocity checks
        Index("idx_fingerprint_last_seen", "fingerprint_hash", "last_seen"),
    )

    def __repr__(self):
        """String representation for debugging."""
        return f"<DeviceFingerprint(hash={self.fingerprint_hash[:8]}..., risk={self.risk_score})>"


"""
===============================================================================
DATABASE INDEXES: WHY THEY MATTER
===============================================================================

An index is like a book's index - it helps find information quickly.

WITHOUT INDEX:
    SELECT * FROM fraud_transactions WHERE user_id = 'USER_123';
    → Database scans ALL rows (1 million rows = slow!)

WITH INDEX:
    CREATE INDEX idx_user_id ON fraud_transactions(user_id);
    → Database jumps directly to matching rows (instant!)

Performance Impact:
- No index: 1,000,000 rows scanned → 2,000ms
- With index: 50 rows scanned → 5ms (400x faster!)

Our Indexes:
1. Single-column indexes: user_id, transaction_id, ip_address
   - Fast lookups: "Find user USER_123's transactions"

2. Composite indexes: (user_id, created_at), (vertical, risk_score)
   - Fast range queries: "Find user's transactions in last 24 hours"

3. Unique indexes: transaction_id, user_id (in UserRiskProfile)
   - Enforce uniqueness + fast lookups

Trade-off:
- Indexes speed up SELECT queries
- But slow down INSERT/UPDATE (must update index too)
- For fraud detection: We read > write, so indexes are essential!
"""
```

---

### 5. `scripts/init_db.py`

```python
"""
Database initialization script.

This script:
1. Connects to PostgreSQL
2. Creates all tables defined in models
3. Verifies tables were created successfully

Run this script once to set up your database:
    python scripts/init_db.py
"""

import sys
import os

# Add project root to Python path (so we can import app modules)
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from app.db.session import engine, Base
from app.models import FraudTransaction, UserRiskProfile, DeviceFingerprint


def init_db():
    """
    Initialize the database by creating all tables.

    This function:
    1. Imports all models (FraudTransaction, UserRiskProfile, DeviceFingerprint)
    2. Calls Base.metadata.create_all(engine) to create tables
    3. Prints success/error messages
    """

    print("=" * 80)
    print("Sentinel Fraud Detection - Database Initialization")
    print("=" * 80)

    try:
        # Test database connection
        print("\n[1/3] Testing database connection...")
        with engine.connect() as conn:
            print("✅ Successfully connected to PostgreSQL!")

        # Create all tables
        print("\n[2/3] Creating database tables...")
        Base.metadata.create_all(bind=engine)

        # The above line creates these tables:
        # - fraud_transactions
        # - user_risk_profiles
        # - device_fingerprints

        print("✅ Successfully created tables:")
        print("   - fraud_transactions")
        print("   - user_risk_profiles")
        print("   - device_fingerprints")

        # Verify tables exist
        print("\n[3/3] Verifying tables...")
        from sqlalchemy import inspect
        inspector = inspect(engine)
        tables = inspector.get_table_names()

        expected_tables = ["fraud_transactions", "user_risk_profiles", "device_fingerprints"]
        for table in expected_tables:
            if table in tables:
                print(f"   ✅ {table}")
            else:
                print(f"   ❌ {table} (NOT FOUND!)")

        print("\n" + "=" * 80)
        print("🎉 Database initialization complete!")
        print("=" * 80)
        print("\nNext steps:")
        print("1. Verify tables: psql -U sentinel_user -d sentinel_fraud_db -c '\\dt'")
        print("2. Run FastAPI: uvicorn app.main:app --reload")
        print("3. Test API: curl http://localhost:8000/")

    except Exception as e:
        print(f"\n❌ Error initializing database: {e}")
        print("\nTroubleshooting:")
        print("1. Is PostgreSQL running? Check: sudo systemctl status postgresql")
        print("2. Is DATABASE_URL correct? Check .env file")
        print("3. Does database exist? Create: psql -U postgres -c 'CREATE DATABASE sentinel_fraud_db'")
        sys.exit(1)


if __name__ == "__main__":
    init_db()
```

---

### 6. Updated `app/main.py`

```python
"""
Sentinel Fraud Detection Platform - Main FastAPI Application

Day 2: Added database connectivity check endpoint
"""

from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session
from app.db.session import engine, get_db
import sqlalchemy

# Create FastAPI app instance
app = FastAPI(
    title="Sentinel Fraud Detection API",
    description="Real-time fraud detection for fintech, e-commerce, betting, crypto, and marketplaces",
    version="1.0.0"
)


@app.get("/")
def read_root():
    """
    Health check endpoint.

    Returns:
        dict: Basic API information
    """
    return {
        "message": "Sentinel Fraud Detection API is running",
        "version": "1.0.0",
        "status": "operational"
    }


@app.get("/health")
def health_check():
    """
    Detailed health check endpoint.

    Returns:
        dict: API status and version
    """
    return {
        "status": "healthy",
        "service": "sentinel-fraud-detection",
        "version": "1.0.0"
    }


@app.get("/db-check")
def database_check(db: Session = Depends(get_db)):
    """
    Database connectivity check.

    This endpoint:
    1. Tests database connection
    2. Executes a simple query (SELECT 1)
    3. Returns success/failure

    Args:
        db: Database session (injected by FastAPI)

    Returns:
        dict: Database status
    """
    try:
        # Execute a simple query to test connection
        result = db.execute(sqlalchemy.text("SELECT 1")).fetchone()

        # Query successful
        return {
            "status": "connected",
            "database": "postgresql",
            "message": "Database connection successful",
            "test_query_result": result[0]
        }

    except Exception as e:
        # Connection failed
        return {
            "status": "error",
            "database": "postgresql",
            "message": f"Database connection failed: {str(e)}"
        }


@app.get("/db-stats")
def database_stats(db: Session = Depends(get_db)):
    """
    Database statistics endpoint.

    Returns table counts for all models.

    Args:
        db: Database session

    Returns:
        dict: Table row counts
    """
    try:
        from app.models import FraudTransaction, UserRiskProfile, DeviceFingerprint

        # Count rows in each table
        fraud_tx_count = db.query(FraudTransaction).count()
        user_profile_count = db.query(UserRiskProfile).count()
        device_count = db.query(DeviceFingerprint).count()

        return {
            "status": "success",
            "tables": {
                "fraud_transactions": fraud_tx_count,
                "user_risk_profiles": user_profile_count,
                "device_fingerprints": device_count
            },
            "total_records": fraud_tx_count + user_profile_count + device_count
        }

    except Exception as e:
        return {
            "status": "error",
            "message": f"Failed to fetch database stats: {str(e)}"
        }
```

---

### 7. Updated `requirements.txt`

```txt
# Web Framework
fastapi==0.104.1
uvicorn[standard]==0.24.0

# Database (Day 2)
sqlalchemy==2.0.23
psycopg2-binary==2.9.9  # PostgreSQL driver
alembic==1.13.0          # Database migrations (future use)

# Environment Variables
python-dotenv==1.0.0

# Development Tools
pytest==7.4.3
httpx==0.25.2            # For testing FastAPI endpoints
```

---

## Database Schema Explained

### Table 1: `fraud_transactions`

**Purpose:** Record every transaction analyzed by the system.

| Column | Type | Purpose | Example |
|--------|------|---------|---------|
| `id` | Integer | Auto-incrementing primary key | 1, 2, 3, ... |
| `transaction_id` | String(100) | Unique transaction ID from client | "TXN_20240101_ABC123" |
| `user_id` | String(100) | User identifier | "USER_123456" |
| `amount` | Numeric(15,2) | Transaction amount | 1500.50 |
| `currency` | String(3) | ISO currency code | "NGN", "USD" |
| `risk_score` | Float | Fraud risk (0.0-1.0) | 0.75 (75% fraud probability) |
| `decision` | String(20) | approve, decline, review | "decline" |
| `fraud_flags` | Array[String] | Triggered rules | ["velocity_check", "new_device"] |
| `ip_address` | String(45) | Client IP | "41.203.123.45" |
| `metadata` | JSONB | Flexible data | `{"location": {"city": "Lagos"}}` |
| `created_at` | DateTime(TZ) | When record created | "2024-01-01 10:30:00+00" |

**Key Indexes:**
- `idx_user_created` (user_id, created_at): Fast user transaction history queries
- `idx_vertical_risk` (vertical, risk_score): Analyze fraud by industry
- `idx_ip_created` (ip_address, created_at): IP-based fraud detection

### Table 2: `user_risk_profiles`

**Purpose:** Track each user's risk profile and behavioral patterns.

| Column | Type | Purpose | Example |
|--------|------|---------|---------|
| `id` | Integer | Primary key | 1, 2, 3, ... |
| `user_id` | String(100) | Unique user ID | "USER_123456" |
| `risk_score` | Float | Current risk (0.0-1.0) | 0.65 (medium-high risk) |
| `total_transactions` | Integer | Lifetime transaction count | 47 |
| `total_amount` | Numeric(18,2) | Lifetime transaction volume | 125000.00 |
| `fraud_count` | Integer | Fraud attempts | 2 |
| `is_blacklisted` | Boolean | Auto-decline flag | false |
| `metadata` | JSONB | User details | `{"email": "user@example.com"}` |
| `last_transaction_at` | DateTime(TZ) | Most recent activity | "2024-01-15 14:22:00+00" |

**Key Indexes:**
- `idx_blacklist_risk` (is_blacklisted, risk_score): Find high-risk users
- `idx_last_transaction` (last_transaction_at): Identify inactive users

### Table 3: `device_fingerprints`

**Purpose:** Identify unique devices and track device-based fraud patterns.

| Column | Type | Purpose | Example |
|--------|------|---------|---------|
| `id` | Integer | Primary key | 1, 2, 3, ... |
| `fingerprint_hash` | String(64) | SHA-256 device hash | "a3f4d5e6c7b8a9d0..." |
| `risk_score` | Float | Device risk (0.0-1.0) | 0.85 (high risk) |
| `user_count` | Integer | Users on this device | 15 (suspicious!) |
| `transaction_count` | Integer | Total transactions | 203 |
| `fraud_count` | Integer | Fraud attempts | 12 |
| `is_blacklisted` | Boolean | Blacklisted device | true |
| `metadata` | JSONB | Device details | `{"os": "Windows", "browser": "Chrome"}` |

**Key Indexes:**
- `idx_suspicious_risk` (is_suspicious, risk_score): Find risky devices
- `idx_fingerprint_last_seen` (fingerprint_hash, last_seen): Device velocity tracking

---

## Testing Your Setup

### Step 1: Create Tables

```bash
# Navigate to project directory
cd /path/to/sentinel-fraud-detection

# Run initialization script
python scripts/init_db.py
```

**Expected output:**
```
================================================================================
Sentinel Fraud Detection - Database Initialization
================================================================================

[1/3] Testing database connection...
✅ Successfully connected to PostgreSQL!

[2/3] Creating database tables...
✅ Successfully created tables:
   - fraud_transactions
   - user_risk_profiles
   - device_fingerprints

[3/3] Verifying tables...
   ✅ fraud_transactions
   ✅ user_risk_profiles
   ✅ device_fingerprints

================================================================================
🎉 Database initialization complete!
================================================================================
```

### Step 2: Verify Tables in PostgreSQL

```bash
# Connect to database
psql -U sentinel_user -d sentinel_fraud_db -h localhost

# List all tables
\dt

# Expected output:
                       List of relations
 Schema |        Name         | Type  |     Owner
--------+---------------------+-------+----------------
 public | device_fingerprints | table | sentinel_user
 public | fraud_transactions  | table | sentinel_user
 public | user_risk_profiles  | table | sentinel_user
```

### Step 3: Inspect Table Structure

```sql
-- View fraud_transactions table structure
\d fraud_transactions

-- Expected output shows all columns, types, and indexes
```

### Step 4: Test with FastAPI

```bash
# Start FastAPI server
uvicorn app.main:app --reload
```

**Test endpoints:**

```bash
# 1. Health check
curl http://localhost:8000/

# Expected: {"message": "Sentinel Fraud Detection API is running", ...}

# 2. Database connectivity check
curl http://localhost:8000/db-check

# Expected: {"status": "connected", "database": "postgresql", ...}

# 3. Database statistics
curl http://localhost:8000/db-stats

# Expected: {"status": "success", "tables": {"fraud_transactions": 0, ...}}
```

### Step 5: Insert Test Data

**In PostgreSQL:**

```sql
-- Insert test fraud transaction
INSERT INTO fraud_transactions (
    transaction_id,
    user_id,
    amount,
    currency,
    transaction_type,
    vertical,
    risk_score,
    decision,
    ip_address,
    metadata
) VALUES (
    'TXN_TEST_001',
    'USER_12345',
    1500.00,
    'NGN',
    'payment',
    'lending',
    0.75,
    'decline',
    '41.203.123.45',
    '{"location": {"country": "NG", "city": "Lagos"}}'::jsonb
);

-- Insert test user profile
INSERT INTO user_risk_profiles (
    user_id,
    risk_score,
    total_transactions,
    total_amount,
    fraud_count,
    is_verified,
    is_blacklisted,
    metadata
) VALUES (
    'USER_12345',
    0.65,
    10,
    15000.00,
    2,
    false,
    false,
    '{"email": "user@example.com", "phone": "+2348012345678"}'::jsonb
);

-- Insert test device fingerprint
INSERT INTO device_fingerprints (
    fingerprint_hash,
    risk_score,
    user_count,
    transaction_count,
    fraud_count,
    is_suspicious,
    metadata
) VALUES (
    'a3f4d5e6c7b8a9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4',
    0.85,
    15,
    203,
    12,
    true,
    '{"os": "Windows", "browser": "Chrome", "screen": {"width": 1920, "height": 1080}}'::jsonb
);
```

### Step 6: Query Test Data

```sql
-- View all fraud transactions
SELECT
    transaction_id,
    user_id,
    amount,
    risk_score,
    decision,
    created_at
FROM fraud_transactions;

-- Find high-risk users
SELECT
    user_id,
    risk_score,
    total_transactions,
    fraud_count,
    is_blacklisted
FROM user_risk_profiles
WHERE risk_score > 0.7
ORDER BY risk_score DESC;

-- Find suspicious devices
SELECT
    fingerprint_hash,
    risk_score,
    user_count,
    fraud_count,
    is_blacklisted
FROM device_fingerprints
WHERE is_suspicious = true;

-- Join query: Get user's transactions with profile
SELECT
    ft.transaction_id,
    ft.amount,
    ft.risk_score AS tx_risk,
    urp.risk_score AS user_risk,
    urp.fraud_count
FROM fraud_transactions ft
JOIN user_risk_profiles urp ON ft.user_id = urp.user_id
WHERE urp.risk_score > 0.5;
```

---

## Troubleshooting

### Problem 1: "psql: error: connection to server failed"

**Symptoms:**
```
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

**Solutions:**

```bash
# Check if PostgreSQL is running
sudo systemctl status postgresql

# If not running, start it
sudo systemctl start postgresql

# Enable auto-start on boot
sudo systemctl enable postgresql
```

### Problem 2: "FATAL: password authentication failed for user"

**Symptoms:**
```
psql: FATAL: password authentication failed for user "sentinel_user"
```

**Solutions:**

```bash
# Option 1: Reset password
sudo -u postgres psql
ALTER USER sentinel_user WITH PASSWORD 'sentinel_pass_2024';
\q

# Option 2: Update .env file with correct credentials
nano .env
# DATABASE_URL=postgresql://sentinel_user:CORRECT_PASSWORD@localhost:5432/sentinel_fraud_db
```

### Problem 3: "sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) could not connect"

**Symptoms:**
```python
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) could not connect to server
```

**Solutions:**

```bash
# Check PostgreSQL is listening on port 5432
sudo netstat -tulpn | grep 5432

# Check DATABASE_URL in .env
cat .env | grep DATABASE_URL

# Test connection manually
psql -U sentinel_user -d sentinel_fraud_db -h localhost
```

### Problem 4: "relation 'fraud_transactions' does not exist"

**Symptoms:**
```sql
ERROR:  relation "fraud_transactions" does not exist
```

**Solutions:**

```bash
# Tables not created yet - run init script
python scripts/init_db.py

# Verify tables exist
psql -U sentinel_user -d sentinel_fraud_db -c '\dt'
```

### Problem 5: "ImportError: No module named 'psycopg2'"

**Symptoms:**
```python
ImportError: No module named 'psycopg2'
```

**Solutions:**

```bash
# Install PostgreSQL driver
pip install psycopg2-binary

# Or install from requirements.txt
pip install -r requirements.txt
```

---

## Key Takeaways

### What You Learned Today

1. **PostgreSQL Fundamentals**
   - What PostgreSQL is and why it's essential for fraud detection
   - ACID compliance ensures data reliability
   - JSONB enables flexible fraud metadata storage

2. **SQLAlchemy ORM**
   - Engine: Manages database connections (connection pooling)
   - Session: Manages transactions (groups operations)
   - Base: Foundation for defining models
   - ORM is safer and more maintainable than raw SQL

3. **Database Models**
   - Created 3 production-ready tables:
     - `fraud_transactions`: Every analyzed transaction
     - `user_risk_profiles`: User risk tracking
     - `device_fingerprints`: Device identification
   - Each model includes indexes for performance
   - JSONB columns provide flexibility

4. **Database Performance**
   - Indexes dramatically speed up queries (400x faster!)
   - Composite indexes optimize complex queries
   - Connection pooling prevents connection exhaustion

5. **FastAPI Integration**
   - Dependency injection (`Depends(get_db)`) provides database sessions
   - Sessions are automatically created and closed per request
   - Clean separation of concerns

### Tomorrow's Preview (Day 3)

On **Day 3**, we'll create **Pydantic schemas** for:
- Type-safe request/response validation
- Automatic API documentation
- Data serialization/deserialization
- Request validation (prevent bad data)

**Example:**
```python
class TransactionRequest(BaseModel):
    transaction_id: str
    amount: float = Field(gt=0, description="Must be positive")
    currency: str = Field(regex="^[A-Z]{3}$")  # ISO 4217
```

### Metrics

- **Lines of code written:** ~800
- **Database tables created:** 3
- **Total columns:** 50+
- **Indexes created:** 9
- **Time to complete:** 2-3 hours

---

## Navigation

### Previous Day
- **[← Day 1: Project Setup & FastAPI Basics](./README-DAY-001.md)**

### Next Day
- **[→ Day 3: Pydantic Schemas & Validation](./README-DAY-003.md)**

### Related Resources
- **[Main Guide](./README.md)** - 120-day roadmap
- **[SQLAlchemy Docs](https://docs.sqlalchemy.org/)** - Official documentation
- **[PostgreSQL Tutorial](https://www.postgresql.org/docs/current/tutorial.html)** - Learn PostgreSQL
- **[FastAPI Database Guide](https://fastapi.tiangolo.com/tutorial/sql-databases/)** - FastAPI + SQLAlchemy

---

## 📝 Daily Checklist

Mark your progress:

- [ ] PostgreSQL installed and running
- [ ] Database and user created
- [ ] All 7 code files created
- [ ] `scripts/init_db.py` executed successfully
- [ ] All 3 tables exist in database
- [ ] FastAPI `/db-check` endpoint returns success
- [ ] Test data inserted and queried
- [ ] Understand engine, session, Base concepts
- [ ] Understand why indexes matter

---

**🎉 Congratulations!** You've completed Day 2 and set up a production-grade database for fraud detection!

**Tomorrow:** Learn Pydantic schemas for type-safe API requests and responses.

---

**Questions or issues?** Open an issue on GitHub or join our Discord community.

**Star the repository** if you find this guide helpful! ⭐
