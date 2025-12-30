# Day 6: Redis Setup & Caching Layer

**50x Performance Boost with Distributed Caching**

**Navigation:** [← Previous: Day 5](README-DAY-005.md) | [Main Guide](README.md) | [Next: Day 7 →](README-DAY-007.md)

---

## Table of Contents

1. [Overview](#overview)
2. [Why Redis for Fraud Detection?](#why-redis-for-fraud-detection)
3. [Understanding Caching Patterns](#understanding-caching-patterns)
4. [Redis Architecture & Setup](#redis-architecture--setup)
5. [Installing Redis](#installing-redis)
6. [Complete Code Implementation](#complete-code-implementation)
7. [Cache Key Strategy](#cache-key-strategy)
8. [TTL Strategy & Configuration](#ttl-strategy--configuration)
9. [Testing & Performance Measurement](#testing--performance-measurement)
10. [Cache Invalidation Strategies](#cache-invalidation-strategies)
11. [Troubleshooting](#troubleshooting)
12. [Next Steps](#next-steps)

---

## Overview

### Day 6 Objectives

Welcome to Day 6! Today we're supercharging your fraud detection API with **Redis caching** to achieve dramatic performance improvements. You'll learn how to implement enterprise-grade caching that reduces database load and response times by **50x or more**.

**What You'll Build:**

✅ Redis connection with connection pooling
✅ Cache manager with async operations
✅ Cache fraud check results (5-minute TTL)
✅ Cache user risk profiles (1-hour TTL)
✅ Implement cache-aside pattern
✅ Cache invalidation on data updates
✅ Graceful degradation when Redis is down
✅ Performance monitoring and metrics

**Performance Gains You'll See:**

```
WITHOUT CACHING:
  Request 1: 45ms (database query)
  Request 2: 43ms (database query)
  Request 3: 46ms (database query)
  Average: ~45ms

WITH CACHING:
  Request 1: 45ms (database query + cache write)
  Request 2: 0.8ms (cache hit)
  Request 3: 0.9ms (cache hit)
  Average: ~15ms (3x faster)

  Subsequent requests: <1ms (50x faster!) ⚡
```

**Estimated Time:** 2-3 hours

**Prerequisites:**
- ✅ Completed Days 1-5
- ✅ PostgreSQL running and configured
- ✅ FastAPI fraud detection API working
- ✅ Virtual environment activated
- ⬜ Redis installed (we'll do this today)

**New Packages (Day 6 Only):**

```txt
redis==5.0.1
hiredis==2.2.3
```

---

## Why Redis for Fraud Detection?

### What is Redis?

**Redis** (REmote DIctionary Server) is an in-memory data structure store used as:
- **Cache** - Store frequently accessed data in RAM for ultra-fast retrieval
- **Session Store** - Manage user sessions across multiple servers
- **Message Broker** - Pub/sub for real-time notifications
- **Rate Limiter** - Track API usage and enforce limits

**Why Redis vs Other Caching Solutions?**

| Feature | Redis | Memcached | Database Cache |
|---------|-------|-----------|---------------|
| **Speed** | ✅ <1ms | ✅ <1ms | ⚠️ 10-50ms |
| **Data Structures** | ✅ Rich (strings, hashes, sets, sorted sets) | ❌ Strings only | ⚠️ Tables |
| **Persistence** | ✅ Optional (RDB/AOF) | ❌ No | ✅ Yes |
| **Pub/Sub** | ✅ Yes | ❌ No | ❌ No |
| **TTL Support** | ✅ Per-key | ✅ Per-key | ⚠️ Complex |
| **Atomic Operations** | ✅ Yes | ⚠️ Limited | ✅ Yes |
| **Memory Efficiency** | ✅ Excellent | ✅ Excellent | ❌ Poor |

### Why Fraud Detection Needs Caching

**1. Reduce Database Load**
```
Without cache: 1000 requests/sec → 1000 database queries
With cache: 1000 requests/sec → 100 database queries (90% cache hit rate)
Result: 10x less database load!
```

**2. Faster Response Times**
```
Fraud check without cache:
  ├─ Validate request: 1ms
  ├─ Query user profile: 15ms
  ├─ Query transaction history: 20ms
  ├─ Run rules engine: 5ms
  └─ Save result: 10ms
  Total: 51ms

Fraud check with cache:
  ├─ Validate request: 1ms
  ├─ Check cache: 0.5ms (HIT!)
  └─ Return cached result: 0.5ms
  Total: 2ms (25x faster!)
```

**3. Resilience & Scalability**
- **Horizontal scaling**: Add more cache nodes as traffic grows
- **Graceful degradation**: API works even if cache is down
- **Reduced costs**: Fewer database reads = lower cloud costs

**4. Rate Limiting**
```python
# Track user requests in Redis
key = f"rate_limit:user:{user_id}"
request_count = redis.incr(key)
if request_count == 1:
    redis.expire(key, 60)  # 1 minute window
if request_count > 100:
    return "Rate limit exceeded"
```

---

## Understanding Caching Patterns

### 1. Cache-Aside Pattern (Lazy Loading)

This is the pattern we'll implement today. It's the most common caching pattern:

```
┌─────────────────────────────────────────────────┐
│         CLIENT REQUEST                          │
│         "Get user fraud profile"                │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│    1. CHECK CACHE                                │
│    key = "user_profile:user_123"                │
└────────────┬────────────────────────────────────┘
             │
        ┌────┴────┐
        │         │
    CACHE HIT   CACHE MISS
        │         │
        ▼         ▼
┌─────────────┐  ┌─────────────────────────────┐
│  2a. RETURN │  │  2b. QUERY DATABASE         │
│  FROM CACHE │  │  SELECT * FROM profiles ... │
│             │  └────────────┬────────────────┘
│  Time: 1ms  │               │
└─────────────┘               ▼
                  ┌─────────────────────────────┐
                  │  3. WRITE TO CACHE          │
                  │  SET key value EX 3600      │
                  └────────────┬────────────────┘
                               │
                               ▼
                  ┌─────────────────────────────┐
                  │  4. RETURN TO CLIENT        │
                  │  Time: 45ms (first time)    │
                  │  Time: 1ms (subsequent)     │
                  └─────────────────────────────┘
```

**Pseudo-code:**

```python
def get_user_profile(user_id):
    # 1. Try cache first
    cache_key = f"user_profile:{user_id}"
    cached_data = redis.get(cache_key)

    if cached_data:
        # 2a. Cache hit - return immediately
        return json.loads(cached_data)

    # 2b. Cache miss - query database
    profile = db.query(UserProfile).filter_by(user_id=user_id).first()

    # 3. Store in cache for next time
    redis.setex(cache_key, 3600, json.dumps(profile))  # TTL: 1 hour

    # 4. Return data
    return profile
```

**Pros:**
- ✅ Simple to implement
- ✅ Only caches requested data (memory efficient)
- ✅ Cache failures don't break the app

**Cons:**
- ⚠️ First request is always slow (cache miss)
- ⚠️ Cache stampede risk (many requests for expired key)

---

### 2. Write-Through Pattern

Data is written to cache AND database simultaneously:

```python
def create_fraud_check(transaction):
    # 1. Write to database
    db.add(transaction)
    db.commit()

    # 2. Write to cache immediately
    cache_key = f"fraud_check:{transaction.id}"
    redis.setex(cache_key, 300, json.dumps(transaction))

    return transaction
```

**Pros:**
- ✅ Cache always in sync with database
- ✅ No cache misses on reads

**Cons:**
- ⚠️ Every write hits two systems (slower writes)
- ⚠️ Cache may store data that's never read (memory waste)

---

### 3. Write-Behind Pattern (Write-Back)

Writes go to cache first, then async to database:

```python
def update_user_profile(user_id, data):
    # 1. Update cache immediately
    cache_key = f"user_profile:{user_id}"
    redis.setex(cache_key, 3600, json.dumps(data))

    # 2. Queue database write for later
    background_queue.enqueue(write_to_db, user_id, data)

    return data
```

**Pros:**
- ✅ Extremely fast writes
- ✅ Reduces database load

**Cons:**
- ⚠️ Risk of data loss if cache crashes before DB write
- ⚠️ Complex to implement correctly

---

### 4. Time-To-Live (TTL) Strategy

Every cached item should have an expiration time:

```python
# Short TTL (5 minutes) - Frequently changing data
redis.setex("fraud_check:tx_123", 300, data)  # 5 minutes

# Medium TTL (1 hour) - Moderately stable data
redis.setex("user_profile:user_456", 3600, data)  # 1 hour

# Long TTL (24 hours) - Rarely changing data
redis.setex("fraud_rules:v1", 86400, data)  # 24 hours
```

**Choosing TTL:**
- **Too short**: More cache misses, higher database load
- **Too long**: Stale data, memory waste
- **Rule of thumb**: TTL = How long can you tolerate stale data?

---

## Redis Architecture & Setup

### Architecture with Redis

```
┌────────────────────────────────────────────────────────────┐
│                    CLIENT REQUESTS                         │
│              (Mobile App, Web, API Consumers)              │
└───────────────────────┬────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────────┐
│               FASTAPI APPLICATION                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  API Routes (fraud.py)                               │  │
│  │  POST /api/v1/check-fraud                            │  │
│  └────────────┬─────────────────────────────────────────┘  │
│               │                                            │
│               ▼                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Service Layer (fraud_service.py)                    │  │
│  │  - check_fraud()                                     │  │
│  │  - get_user_profile()                                │  │
│  └────┬───────────────────────────────┬─────────────────┘  │
│       │                               │                    │
│       ▼                               ▼                    │
│  ┌──────────────┐              ┌──────────────┐            │
│  │ CacheManager │              │ Rules Engine │            │
│  │ (NEW!)       │              │ (Day 4)      │            │
│  └──────┬───────┘              └──────────────┘            │
│         │                                                  │
└─────────┼──────────────────────────────────────────────────┘
          │
    ┌─────┴──────┐
    │            │
    ▼            ▼
┌─────────┐  ┌─────────────┐
│  REDIS  │  │  POSTGRESQL │
│  CACHE  │  │  DATABASE   │
│         │  │             │
│ Port:   │  │ Port: 5432  │
│ 6379    │  │             │
└─────────┘  └─────────────┘

FLOW:
1. Request arrives → Service layer
2. Service checks cache (CacheManager)
   - Hit? Return cached data (1ms)
   - Miss? Query database (45ms)
3. On miss: Store result in cache
4. Return response to client
```

### Connection Pooling

Redis connection pooling prevents creating new connections for every request:

```python
# WITHOUT CONNECTION POOLING (❌ Bad)
def get_user():
    redis_client = redis.Redis(host='localhost', port=6379)  # New connection!
    data = redis_client.get('user:123')
    redis_client.close()  # Close connection
    return data

# Result: 100 requests = 100 connections = slow + resource waste


# WITH CONNECTION POOLING (✅ Good)
pool = redis.ConnectionPool(host='localhost', port=6379, max_connections=10)

def get_user():
    redis_client = redis.Redis(connection_pool=pool)  # Reuse connection!
    data = redis_client.get('user:123')
    # Connection automatically returned to pool
    return data

# Result: 100 requests = 10 pooled connections = fast + efficient
```

---

## Installing Redis

### Option 1: Ubuntu/Debian

```bash
# Update package list
sudo apt update

# Install Redis
sudo apt install redis-server -y

# Start Redis service
sudo systemctl start redis-server

# Enable Redis to start on boot
sudo systemctl enable redis-server

# Verify Redis is running
sudo systemctl status redis-server

# Expected output:
# ● redis-server.service - Advanced key-value store
#    Loaded: loaded (/lib/systemd/system/redis-server.service; enabled)
#    Active: active (running) since Mon 2024-01-15 10:00:00 UTC
```

### Option 2: macOS

```bash
# Install using Homebrew
brew install redis

# Start Redis
brew services start redis

# Verify
redis-cli ping
# Expected: PONG
```

### Option 3: Docker (Recommended for Development)

```bash
# Run Redis in Docker
docker run -d \
  --name sentinel-redis \
  -p 6379:6379 \
  redis:7-alpine

# Verify
docker ps | grep redis

# Test connection
docker exec -it sentinel-redis redis-cli ping
# Expected: PONG
```

### Test Redis Connection

```bash
# Connect to Redis CLI
redis-cli

# Test basic commands
127.0.0.1:6379> PING
PONG

127.0.0.1:6379> SET test "Hello Redis"
OK

127.0.0.1:6379> GET test
"Hello Redis"

127.0.0.1:6379> DEL test
(integer) 1

127.0.0.1:6379> EXIT
```

---

## Complete Code Implementation

### Step 1: Install Redis Python Client

```bash
# Ensure you're in your project directory
cd /home/user/sentinel-api

# Activate virtual environment
source venv/bin/activate

# Install Redis packages
pip install redis==5.0.1 hiredis==2.2.3

# Verify installation
pip list | grep redis
# Expected output:
# redis        5.0.1
# hiredis      2.2.3
```

**Update requirements.txt:**

```bash
# Add to requirements.txt
echo "redis==5.0.1" >> requirements.txt
echo "hiredis==2.2.3" >> requirements.txt
```

**Why hiredis?**

`hiredis` is a C parser for Redis that provides:
- **3x faster** parsing than pure Python
- **Lower memory usage**
- **Better performance** under high load

---

### Step 2: Create Cache Directory Structure

```bash
# Create cache directory and files
mkdir -p /home/user/sentinel-api/app/cache
touch /home/user/sentinel-api/app/cache/__init__.py
touch /home/user/sentinel-api/app/cache/redis.py
touch /home/user/sentinel-api/app/cache/keys.py
```

**Verify structure:**

```bash
tree /home/user/sentinel-api/app/cache

# Expected output:
# app/cache/
# ├── __init__.py
# ├── redis.py
# └── keys.py
```

---

### Step 3: Configure Redis Connection

**File: `/home/user/sentinel-api/.env`**

Add Redis configuration to your existing `.env` file:

```bash
# Existing database configuration
DATABASE_URL=postgresql://sentinel_user:sentinel_pass@localhost:5432/sentinel_db

# Redis configuration (NEW)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=
REDIS_MAX_CONNECTIONS=10
REDIS_SOCKET_TIMEOUT=5
REDIS_SOCKET_CONNECT_TIMEOUT=5

# Cache TTL settings (in seconds)
CACHE_TTL_FRAUD_CHECK=300        # 5 minutes
CACHE_TTL_USER_PROFILE=3600      # 1 hour
CACHE_TTL_DEVICE_FINGERPRINT=7200  # 2 hours
CACHE_TTL_FRAUD_RULES=86400      # 24 hours

# Cache enabled flag (for easy disable in development)
CACHE_ENABLED=true
```

**Update config.py to load Redis settings:**

**File: `/home/user/sentinel-api/app/core/config.py`**

```python
"""
Application configuration management.

Loads settings from environment variables and .env file.
Uses Pydantic for validation and type safety.
"""

import os
from typing import Optional
from pydantic import BaseModel, validator
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()


class Settings(BaseModel):
    """Application settings loaded from environment variables."""

    # Application
    app_name: str = "Sentinel Fraud Detection API"
    app_version: str = "0.1.0"
    debug: bool = False

    # Database
    database_url: str = os.getenv(
        "DATABASE_URL",
        "postgresql://sentinel_user:sentinel_pass@localhost:5432/sentinel_db"
    )

    # Redis Configuration (NEW)
    redis_host: str = os.getenv("REDIS_HOST", "localhost")
    redis_port: int = int(os.getenv("REDIS_PORT", "6379"))
    redis_db: int = int(os.getenv("REDIS_DB", "0"))
    redis_password: Optional[str] = os.getenv("REDIS_PASSWORD", None)
    redis_max_connections: int = int(os.getenv("REDIS_MAX_CONNECTIONS", "10"))
    redis_socket_timeout: int = int(os.getenv("REDIS_SOCKET_TIMEOUT", "5"))
    redis_socket_connect_timeout: int = int(os.getenv("REDIS_SOCKET_CONNECT_TIMEOUT", "5"))

    # Cache TTL settings (in seconds)
    cache_ttl_fraud_check: int = int(os.getenv("CACHE_TTL_FRAUD_CHECK", "300"))
    cache_ttl_user_profile: int = int(os.getenv("CACHE_TTL_USER_PROFILE", "3600"))
    cache_ttl_device_fingerprint: int = int(os.getenv("CACHE_TTL_DEVICE_FINGERPRINT", "7200"))
    cache_ttl_fraud_rules: int = int(os.getenv("CACHE_TTL_FRAUD_RULES", "86400"))

    # Cache enabled flag
    cache_enabled: bool = os.getenv("CACHE_ENABLED", "true").lower() == "true"

    @validator("redis_password", pre=True)
    def empty_string_to_none(cls, v):
        """Convert empty string to None for optional password."""
        if v == "":
            return None
        return v

    class Config:
        """Pydantic configuration."""
        case_sensitive = False


# Create global settings instance
settings = Settings()
```

---

### Step 4: Implement Cache Key Strategy

**File: `/home/user/sentinel-api/app/cache/keys.py`**

```python
"""
Cache key generation and management.

Provides consistent naming conventions for cache keys across the application.
This ensures cache keys are:
- Predictable
- Collision-free
- Easy to debug
- Easy to invalidate

Key Format: {namespace}:{entity_type}:{identifier}:{version}
Example: sentinel:fraud_check:tx_abc123:v1
"""

from typing import Optional


class CacheKeys:
    """
    Centralized cache key generation.

    Benefits:
    1. Consistent naming across the application
    2. Easy to change key format in one place
    3. Type-safe key generation
    4. Version support for schema changes
    """

    # Namespace prefix for all cache keys
    NAMESPACE = "sentinel"

    # Key version (increment when schema changes)
    VERSION = "v1"

    @classmethod
    def fraud_check(cls, transaction_id: str) -> str:
        """
        Generate cache key for fraud check result.

        Args:
            transaction_id: Unique transaction identifier

        Returns:
            Cache key: sentinel:fraud_check:tx_abc123:v1

        Example:
            key = CacheKeys.fraud_check("tx_abc123")
            # Returns: "sentinel:fraud_check:tx_abc123:v1"
        """
        return f"{cls.NAMESPACE}:fraud_check:{transaction_id}:{cls.VERSION}"

    @classmethod
    def user_profile(cls, user_id: str) -> str:
        """
        Generate cache key for user risk profile.

        Args:
            user_id: Unique user identifier

        Returns:
            Cache key: sentinel:user_profile:user_123:v1
        """
        return f"{cls.NAMESPACE}:user_profile:{user_id}:{cls.VERSION}"

    @classmethod
    def device_fingerprint(cls, device_id: str) -> str:
        """
        Generate cache key for device fingerprint.

        Args:
            device_id: Unique device identifier

        Returns:
            Cache key: sentinel:device:dev_abc123:v1
        """
        return f"{cls.NAMESPACE}:device:{device_id}:{cls.VERSION}"

    @classmethod
    def user_transactions(cls, user_id: str, period_days: int = 30) -> str:
        """
        Generate cache key for user transaction history.

        Args:
            user_id: Unique user identifier
            period_days: Number of days to look back

        Returns:
            Cache key: sentinel:user_txs:user_123:30d:v1
        """
        return f"{cls.NAMESPACE}:user_txs:{user_id}:{period_days}d:{cls.VERSION}"

    @classmethod
    def fraud_rules(cls, rule_set_name: str = "default") -> str:
        """
        Generate cache key for fraud detection rules.

        Args:
            rule_set_name: Name of the rule set

        Returns:
            Cache key: sentinel:rules:default:v1
        """
        return f"{cls.NAMESPACE}:rules:{rule_set_name}:{cls.VERSION}"

    @classmethod
    def rate_limit(cls, user_id: str, window: str = "1m") -> str:
        """
        Generate cache key for rate limiting.

        Args:
            user_id: Unique user identifier
            window: Time window (e.g., "1m", "1h", "1d")

        Returns:
            Cache key: sentinel:rate_limit:user_123:1m:v1
        """
        return f"{cls.NAMESPACE}:rate_limit:{user_id}:{window}:{cls.VERSION}"

    @classmethod
    def pattern_match(cls, pattern: str) -> str:
        """
        Generate pattern for matching multiple keys.

        Args:
            pattern: Pattern to match (e.g., "user_profile:*")

        Returns:
            Pattern: sentinel:user_profile:*:v1

        Example:
            # Get all user profiles
            pattern = CacheKeys.pattern_match("user_profile:*")
            keys = redis.keys(pattern)
        """
        return f"{cls.NAMESPACE}:{pattern}:{cls.VERSION}"

    @classmethod
    def invalidate_user(cls, user_id: str) -> list[str]:
        """
        Get list of patterns to invalidate all user-related cache.

        Args:
            user_id: User ID to invalidate

        Returns:
            List of patterns to delete

        Example:
            patterns = CacheKeys.invalidate_user("user_123")
            for pattern in patterns:
                cache_manager.delete_pattern(pattern)
        """
        return [
            cls.user_profile(user_id),
            cls.user_transactions(user_id, period_days=30),
            cls.user_transactions(user_id, period_days=90),
            cls.rate_limit(user_id, window="1m"),
            cls.rate_limit(user_id, window="1h"),
        ]


# Example usage and testing
if __name__ == "__main__":
    # Test key generation
    print("Fraud Check Key:", CacheKeys.fraud_check("tx_abc123"))
    print("User Profile Key:", CacheKeys.user_profile("user_456"))
    print("Device Key:", CacheKeys.device_fingerprint("dev_789"))
    print("User Transactions Key:", CacheKeys.user_transactions("user_456", 30))
    print("Fraud Rules Key:", CacheKeys.fraud_rules("default"))
    print("Rate Limit Key:", CacheKeys.rate_limit("user_456", "1m"))
    print("Pattern Match:", CacheKeys.pattern_match("user_profile:*"))
    print("Invalidate User:", CacheKeys.invalidate_user("user_456"))
```

---

### Step 5: Implement Redis Cache Manager

**File: `/home/user/sentinel-api/app/cache/redis.py`**

```python
"""
Redis cache manager with connection pooling and error handling.

Provides a high-level interface for caching operations with:
- Connection pooling for performance
- Automatic serialization/deserialization
- TTL support
- Error handling and graceful degradation
- Async operations support
"""

import json
import logging
from typing import Optional, Any, List
from datetime import timedelta

import redis
from redis.connection import ConnectionPool

from app.core.config import settings

# Configure logging
logger = logging.getLogger(__name__)


class CacheManager:
    """
    Redis cache manager with connection pooling.

    Features:
    - Connection pooling for performance
    - Automatic JSON serialization
    - Configurable TTL
    - Error handling with graceful degradation
    - Pattern-based deletion

    Usage:
        cache = CacheManager()

        # Set value
        cache.set("key", {"data": "value"}, ttl=300)

        # Get value
        data = cache.get("key")

        # Delete value
        cache.delete("key")
    """

    def __init__(self):
        """Initialize Redis connection pool."""
        self._pool: Optional[ConnectionPool] = None
        self._client: Optional[redis.Redis] = None
        self._initialize_pool()

    def _initialize_pool(self) -> None:
        """
        Create Redis connection pool.

        Connection pooling benefits:
        - Reuse connections instead of creating new ones
        - Reduce connection overhead
        - Better performance under load
        """
        try:
            self._pool = ConnectionPool(
                host=settings.redis_host,
                port=settings.redis_port,
                db=settings.redis_db,
                password=settings.redis_password,
                max_connections=settings.redis_max_connections,
                socket_timeout=settings.redis_socket_timeout,
                socket_connect_timeout=settings.redis_socket_connect_timeout,
                decode_responses=True,  # Automatically decode bytes to strings
            )

            self._client = redis.Redis(connection_pool=self._pool)

            # Test connection
            self._client.ping()
            logger.info(
                f"Redis connection pool initialized: "
                f"{settings.redis_host}:{settings.redis_port}"
            )

        except redis.ConnectionError as e:
            logger.error(f"Failed to connect to Redis: {e}")
            logger.warning("Cache will be disabled. Application will work without caching.")
            self._client = None
        except Exception as e:
            logger.error(f"Unexpected error initializing Redis: {e}")
            self._client = None

    def _is_available(self) -> bool:
        """
        Check if Redis is available.

        Returns:
            True if cache is enabled and Redis is connected
        """
        if not settings.cache_enabled:
            return False

        if self._client is None:
            return False

        try:
            self._client.ping()
            return True
        except redis.ConnectionError:
            logger.warning("Redis connection lost. Operating without cache.")
            return False

    def set(
        self,
        key: str,
        value: Any,
        ttl: Optional[int] = None
    ) -> bool:
        """
        Store value in cache with optional TTL.

        Args:
            key: Cache key
            value: Value to cache (will be JSON serialized)
            ttl: Time to live in seconds (None = no expiration)

        Returns:
            True if cached successfully, False otherwise

        Example:
            cache.set("user:123", {"name": "John"}, ttl=3600)
        """
        if not self._is_available():
            return False

        try:
            # Serialize value to JSON
            serialized_value = json.dumps(value)

            if ttl:
                # Set with expiration
                self._client.setex(key, ttl, serialized_value)
            else:
                # Set without expiration
                self._client.set(key, serialized_value)

            logger.debug(f"Cache SET: {key} (TTL: {ttl}s)")
            return True

        except (redis.RedisError, json.JSONEncodeError) as e:
            logger.error(f"Failed to cache key {key}: {e}")
            return False

    def get(self, key: str) -> Optional[Any]:
        """
        Retrieve value from cache.

        Args:
            key: Cache key

        Returns:
            Cached value (deserialized from JSON) or None if not found

        Example:
            data = cache.get("user:123")
            if data:
                print(f"Cache hit: {data}")
            else:
                print("Cache miss")
        """
        if not self._is_available():
            return None

        try:
            value = self._client.get(key)

            if value is None:
                logger.debug(f"Cache MISS: {key}")
                return None

            logger.debug(f"Cache HIT: {key}")

            # Deserialize JSON
            return json.loads(value)

        except (redis.RedisError, json.JSONDecodeError) as e:
            logger.error(f"Failed to retrieve key {key}: {e}")
            return None

    def delete(self, key: str) -> bool:
        """
        Delete key from cache.

        Args:
            key: Cache key to delete

        Returns:
            True if deleted, False otherwise

        Example:
            cache.delete("user:123")
        """
        if not self._is_available():
            return False

        try:
            result = self._client.delete(key)
            logger.debug(f"Cache DELETE: {key} (deleted: {result > 0})")
            return result > 0

        except redis.RedisError as e:
            logger.error(f"Failed to delete key {key}: {e}")
            return False

    def delete_pattern(self, pattern: str) -> int:
        """
        Delete all keys matching pattern.

        Args:
            pattern: Pattern to match (e.g., "user:*")

        Returns:
            Number of keys deleted

        Example:
            # Delete all user caches
            deleted = cache.delete_pattern("sentinel:user_profile:*")
            print(f"Deleted {deleted} keys")
        """
        if not self._is_available():
            return 0

        try:
            # Find all matching keys
            keys = self._client.keys(pattern)

            if not keys:
                logger.debug(f"No keys found for pattern: {pattern}")
                return 0

            # Delete all matching keys
            deleted = self._client.delete(*keys)
            logger.info(f"Deleted {deleted} keys matching pattern: {pattern}")
            return deleted

        except redis.RedisError as e:
            logger.error(f"Failed to delete pattern {pattern}: {e}")
            return 0

    def exists(self, key: str) -> bool:
        """
        Check if key exists in cache.

        Args:
            key: Cache key

        Returns:
            True if key exists, False otherwise
        """
        if not self._is_available():
            return False

        try:
            return self._client.exists(key) > 0
        except redis.RedisError as e:
            logger.error(f"Failed to check existence of key {key}: {e}")
            return False

    def ttl(self, key: str) -> int:
        """
        Get remaining TTL for key.

        Args:
            key: Cache key

        Returns:
            Remaining seconds (-1 if no expiration, -2 if key doesn't exist)
        """
        if not self._is_available():
            return -2

        try:
            return self._client.ttl(key)
        except redis.RedisError as e:
            logger.error(f"Failed to get TTL for key {key}: {e}")
            return -2

    def increment(self, key: str, amount: int = 1) -> Optional[int]:
        """
        Increment value by amount (atomic operation).

        Args:
            key: Cache key
            amount: Amount to increment

        Returns:
            New value after increment, or None on error

        Example:
            # Rate limiting
            count = cache.increment("rate_limit:user:123")
            if count == 1:
                cache.expire("rate_limit:user:123", 60)  # 1 minute window
            if count > 100:
                raise RateLimitExceeded()
        """
        if not self._is_available():
            return None

        try:
            return self._client.incrby(key, amount)
        except redis.RedisError as e:
            logger.error(f"Failed to increment key {key}: {e}")
            return None

    def expire(self, key: str, seconds: int) -> bool:
        """
        Set expiration time for key.

        Args:
            key: Cache key
            seconds: Expiration time in seconds

        Returns:
            True if expiration was set, False otherwise
        """
        if not self._is_available():
            return False

        try:
            return self._client.expire(key, seconds)
        except redis.RedisError as e:
            logger.error(f"Failed to set expiration for key {key}: {e}")
            return False

    def flush_all(self) -> bool:
        """
        Delete all keys in current database.

        WARNING: Use with caution! This deletes ALL cache data.

        Returns:
            True if successful, False otherwise
        """
        if not self._is_available():
            return False

        try:
            self._client.flushdb()
            logger.warning("All cache keys deleted (FLUSHDB)")
            return True
        except redis.RedisError as e:
            logger.error(f"Failed to flush database: {e}")
            return False

    def get_stats(self) -> dict:
        """
        Get Redis server statistics.

        Returns:
            Dictionary with Redis stats

        Example:
            stats = cache.get_stats()
            print(f"Total keys: {stats['db0']['keys']}")
            print(f"Memory used: {stats['used_memory_human']}")
        """
        if not self._is_available():
            return {"error": "Cache not available"}

        try:
            info = self._client.info()
            return {
                "redis_version": info.get("redis_version"),
                "used_memory_human": info.get("used_memory_human"),
                "connected_clients": info.get("connected_clients"),
                "total_commands_processed": info.get("total_commands_processed"),
                "keyspace_hits": info.get("keyspace_hits", 0),
                "keyspace_misses": info.get("keyspace_misses", 0),
            }
        except redis.RedisError as e:
            logger.error(f"Failed to get Redis stats: {e}")
            return {"error": str(e)}

    def close(self) -> None:
        """
        Close Redis connection pool.

        Should be called when application shuts down.
        """
        if self._pool:
            self._pool.disconnect()
            logger.info("Redis connection pool closed")


# Create global cache manager instance
cache_manager = CacheManager()
```

---

### Step 6: Update Cache __init__.py

**File: `/home/user/sentinel-api/app/cache/__init__.py`**

```python
"""
Cache module for distributed caching with Redis.

This module provides caching functionality to improve performance
and reduce database load.

Components:
- CacheManager: Redis client with connection pooling
- CacheKeys: Consistent cache key generation

Usage:
    from app.cache import cache_manager, CacheKeys

    # Cache fraud check result
    key = CacheKeys.fraud_check("tx_123")
    cache_manager.set(key, result, ttl=300)

    # Retrieve from cache
    cached = cache_manager.get(key)
"""

from app.cache.redis import cache_manager, CacheManager
from app.cache.keys import CacheKeys

__all__ = ["cache_manager", "CacheManager", "CacheKeys"]
```

---

### Step 7: Update Fraud Service with Caching

**File: `/home/user/sentinel-api/app/services/fraud_service.py`**

Replace the entire file with this cached version:

```python
"""
Fraud detection service layer with Redis caching.

This service encapsulates fraud detection business logic and integrates:
- Rules engine for fraud detection
- Database operations
- Redis caching for performance
- User risk profile management

Caching strategy:
- Fraud checks: 5-minute TTL (frequently queried, can tolerate some staleness)
- User profiles: 1-hour TTL (changes infrequently)
- Device fingerprints: 2-hour TTL (very stable)
"""

import logging
from datetime import datetime, timedelta
from typing import List, Optional
from sqlalchemy.orm import Session

from app.schemas.fraud import (
    TransactionCheckRequest,
    FraudCheckResponse,
    FraudFlag,
    RiskLevel
)
from app.db.models import FraudTransaction, UserRiskProfile, DeviceFingerprint
from app.rules.engine import RulesEngine
from app.cache import cache_manager, CacheKeys
from app.core.config import settings

logger = logging.getLogger(__name__)


class FraudService:
    """
    Service layer for fraud detection operations.

    Implements cache-aside pattern:
    1. Check cache first
    2. On miss, query database
    3. Store result in cache
    4. Return result
    """

    def __init__(self):
        """Initialize fraud service with rules engine."""
        self.rules_engine = RulesEngine()

    def check_fraud(
        self,
        request: TransactionCheckRequest,
        db: Session
    ) -> FraudCheckResponse:
        """
        Check transaction for fraud with caching.

        Cache strategy:
        - First check cache for existing fraud check result
        - If not found, run fraud detection and cache result
        - TTL: 5 minutes (balance freshness vs performance)

        Args:
            request: Transaction data to check
            db: Database session

        Returns:
            Fraud check response with score and risk level
        """
        # 1. Try to get cached result
        cache_key = CacheKeys.fraud_check(request.transaction_id)
        cached_result = cache_manager.get(cache_key)

        if cached_result:
            logger.info(f"Cache HIT for fraud check: {request.transaction_id}")
            return FraudCheckResponse(**cached_result)

        logger.info(f"Cache MISS for fraud check: {request.transaction_id}")

        # 2. Cache miss - run fraud detection
        # Run rules engine
        rule_results = self.rules_engine.evaluate(request)

        # Calculate fraud score (sum of triggered rules)
        fraud_score = sum(r.score for r in rule_results if r.triggered)

        # Determine risk level
        risk_level = self._determine_risk_level(fraud_score)

        # Create fraud flags from triggered rules
        fraud_flags = [
            FraudFlag(
                flag_type=r.rule_name,
                severity=self._score_to_severity(r.score),
                description=r.reason
            )
            for r in rule_results if r.triggered
        ]

        # Determine if fraudulent (high risk = fraudulent)
        is_fraudulent = risk_level == RiskLevel.HIGH

        # 3. Save to database
        fraud_tx = FraudTransaction(
            transaction_id=request.transaction_id,
            user_id=request.user_id,
            amount=float(request.amount),
            currency=request.currency,
            transaction_type=request.transaction_type.value,
            fraud_score=fraud_score,
            risk_level=risk_level.value,
            is_flagged=is_fraudulent,
            device_id=request.device_info.device_id if request.device_info else None,
            ip_address=request.device_info.ip_address if request.device_info else None,
            country=request.location_info.country if request.location_info else None,
            city=request.location_info.city if request.location_info else None,
        )
        db.add(fraud_tx)

        # Update user risk profile
        self._update_user_profile(request.user_id, fraud_score, db)

        # Update device fingerprint if provided
        if request.device_info:
            self._update_device_fingerprint(request.device_info.device_id, fraud_score, db)

        # Commit transaction
        db.commit()

        # 4. Build response
        response = FraudCheckResponse(
            transaction_id=request.transaction_id,
            fraud_score=fraud_score,
            risk_level=risk_level,
            is_fraudulent=is_fraudulent,
            fraud_flags=fraud_flags,
            timestamp=datetime.utcnow(),
            recommendation=self._get_recommendation(risk_level)
        )

        # 5. Cache the result
        cache_manager.set(
            cache_key,
            response.dict(),
            ttl=settings.cache_ttl_fraud_check  # 5 minutes
        )

        return response

    def get_fraud_check(
        self,
        transaction_id: str,
        db: Session
    ) -> Optional[FraudCheckResponse]:
        """
        Retrieve fraud check result with caching.

        Cache strategy:
        - Check cache first
        - If not cached, query database and cache result

        Args:
            transaction_id: Transaction ID to retrieve
            db: Database session

        Returns:
            Fraud check response or None if not found
        """
        # 1. Try cache first
        cache_key = CacheKeys.fraud_check(transaction_id)
        cached_result = cache_manager.get(cache_key)

        if cached_result:
            logger.info(f"Cache HIT for get_fraud_check: {transaction_id}")
            return FraudCheckResponse(**cached_result)

        logger.info(f"Cache MISS for get_fraud_check: {transaction_id}")

        # 2. Query database
        fraud_tx = db.query(FraudTransaction).filter(
            FraudTransaction.transaction_id == transaction_id
        ).first()

        if not fraud_tx:
            return None

        # 3. Build response
        response = FraudCheckResponse(
            transaction_id=fraud_tx.transaction_id,
            fraud_score=fraud_tx.fraud_score,
            risk_level=RiskLevel(fraud_tx.risk_level),
            is_fraudulent=fraud_tx.is_flagged,
            fraud_flags=[],  # Could reconstruct from database if needed
            timestamp=fraud_tx.created_at,
            recommendation=self._get_recommendation(RiskLevel(fraud_tx.risk_level))
        )

        # 4. Cache for future requests
        cache_manager.set(
            cache_key,
            response.dict(),
            ttl=settings.cache_ttl_fraud_check
        )

        return response

    def get_user_fraud_history(
        self,
        user_id: str,
        period_days: int,
        db: Session
    ) -> dict:
        """
        Get user fraud history with caching.

        Args:
            user_id: User ID
            period_days: Number of days to look back
            db: Database session

        Returns:
            User fraud statistics
        """
        # 1. Try cache first
        cache_key = CacheKeys.user_transactions(user_id, period_days)
        cached_result = cache_manager.get(cache_key)

        if cached_result:
            logger.info(f"Cache HIT for user fraud history: {user_id}")
            return cached_result

        logger.info(f"Cache MISS for user fraud history: {user_id}")

        # 2. Calculate date range
        since_date = datetime.utcnow() - timedelta(days=period_days)

        # 3. Query transactions
        transactions = db.query(FraudTransaction).filter(
            FraudTransaction.user_id == user_id,
            FraudTransaction.created_at >= since_date
        ).all()

        # 4. Calculate statistics
        total_transactions = len(transactions)
        flagged_transactions = sum(1 for tx in transactions if tx.is_flagged)
        total_amount = sum(tx.amount for tx in transactions)
        avg_fraud_score = (
            sum(tx.fraud_score for tx in transactions) / total_transactions
            if total_transactions > 0 else 0
        )

        # 5. Get user profile
        user_profile = db.query(UserRiskProfile).filter(
            UserRiskProfile.user_id == user_id
        ).first()

        # 6. Build response
        result = {
            "user_id": user_id,
            "period_days": period_days,
            "total_transactions": total_transactions,
            "flagged_transactions": flagged_transactions,
            "total_amount": total_amount,
            "average_fraud_score": round(avg_fraud_score, 2),
            "current_risk_score": user_profile.risk_score if user_profile else 0,
            "account_status": user_profile.status if user_profile else "ACTIVE"
        }

        # 7. Cache result
        cache_manager.set(
            cache_key,
            result,
            ttl=settings.cache_ttl_user_profile  # 1 hour
        )

        return result

    def invalidate_fraud_check_cache(self, transaction_id: str) -> None:
        """
        Invalidate fraud check cache.

        Call this when fraud check result needs to be recalculated.

        Args:
            transaction_id: Transaction ID to invalidate
        """
        cache_key = CacheKeys.fraud_check(transaction_id)
        cache_manager.delete(cache_key)
        logger.info(f"Invalidated cache for transaction: {transaction_id}")

    def invalidate_user_cache(self, user_id: str) -> None:
        """
        Invalidate all cache entries for a user.

        Call this when user profile or history changes significantly.

        Args:
            user_id: User ID to invalidate
        """
        patterns = CacheKeys.invalidate_user(user_id)
        for pattern in patterns:
            cache_manager.delete(pattern)

        logger.info(f"Invalidated all cache for user: {user_id}")

    def _update_user_profile(
        self,
        user_id: str,
        fraud_score: int,
        db: Session
    ) -> None:
        """
        Update user risk profile and invalidate cache.

        Args:
            user_id: User ID
            fraud_score: Latest fraud score
            db: Database session
        """
        profile = db.query(UserRiskProfile).filter(
            UserRiskProfile.user_id == user_id
        ).first()

        if profile:
            # Update existing profile
            profile.transaction_count += 1
            profile.total_fraud_score += fraud_score
            profile.risk_score = profile.total_fraud_score // profile.transaction_count
            profile.last_transaction_at = datetime.utcnow()

            # Update status based on risk score
            if profile.risk_score > 70:
                profile.status = "BLOCKED"
            elif profile.risk_score > 40:
                profile.status = "REVIEW"
            else:
                profile.status = "ACTIVE"
        else:
            # Create new profile
            profile = UserRiskProfile(
                user_id=user_id,
                risk_score=fraud_score,
                transaction_count=1,
                total_fraud_score=fraud_score,
                status="ACTIVE",
                last_transaction_at=datetime.utcnow()
            )
            db.add(profile)

        # Invalidate user cache after update
        self.invalidate_user_cache(user_id)

    def _update_device_fingerprint(
        self,
        device_id: str,
        fraud_score: int,
        db: Session
    ) -> None:
        """Update device fingerprint."""
        fingerprint = db.query(DeviceFingerprint).filter(
            DeviceFingerprint.device_id == device_id
        ).first()

        if fingerprint:
            fingerprint.fraud_check_count += 1
            fingerprint.total_fraud_score += fraud_score
            fingerprint.average_fraud_score = (
                fingerprint.total_fraud_score / fingerprint.fraud_check_count
            )
            fingerprint.last_seen_at = datetime.utcnow()
        else:
            fingerprint = DeviceFingerprint(
                device_id=device_id,
                fraud_check_count=1,
                total_fraud_score=fraud_score,
                average_fraud_score=fraud_score,
                last_seen_at=datetime.utcnow()
            )
            db.add(fingerprint)

    def _determine_risk_level(self, fraud_score: int) -> RiskLevel:
        """Determine risk level based on fraud score."""
        if fraud_score >= 70:
            return RiskLevel.HIGH
        elif fraud_score >= 40:
            return RiskLevel.MEDIUM
        else:
            return RiskLevel.LOW

    def _score_to_severity(self, score: int) -> str:
        """Convert rule score to severity level."""
        if score >= 30:
            return "HIGH"
        elif score >= 15:
            return "MEDIUM"
        else:
            return "LOW"

    def _get_recommendation(self, risk_level: RiskLevel) -> str:
        """Get action recommendation based on risk level."""
        recommendations = {
            RiskLevel.LOW: "APPROVE - Transaction appears legitimate",
            RiskLevel.MEDIUM: "REVIEW - Manual review recommended",
            RiskLevel.HIGH: "BLOCK - High fraud risk detected"
        }
        return recommendations[risk_level]
```

---

## Cache Key Strategy

### Why Cache Keys Matter

Cache keys are like URLs - they need to be:
- **Unique**: No collisions between different data
- **Consistent**: Same data always generates same key
- **Versioned**: Support schema changes
- **Hierarchical**: Easy to invalidate related keys

### Key Naming Convention

We use this format:
```
{namespace}:{entity_type}:{identifier}:{version}
```

**Examples:**

```python
# Fraud check result
"sentinel:fraud_check:tx_abc123:v1"
# ↑        ↑           ↑         ↑
# │        │           │         └─ Version (for schema changes)
# │        │           └─────────── Transaction ID
# │        └─────────────────────── Entity type
# └──────────────────────────────── Namespace (app name)

# User profile
"sentinel:user_profile:user_456:v1"

# Device fingerprint
"sentinel:device:dev_789:v1"

# User transactions (30 days)
"sentinel:user_txs:user_456:30d:v1"
```

### Pattern Matching for Bulk Operations

```python
# Delete all fraud checks
cache_manager.delete_pattern("sentinel:fraud_check:*:v1")

# Delete all user data
cache_manager.delete_pattern("sentinel:user_profile:*:v1")
cache_manager.delete_pattern("sentinel:user_txs:*:v1")

# Delete specific user's cache
patterns = CacheKeys.invalidate_user("user_123")
for pattern in patterns:
    cache_manager.delete(pattern)
```

---

## TTL Strategy & Configuration

### What is TTL?

**TTL (Time To Live)** determines how long data stays in cache before expiring.

```python
# Set value with 5-minute TTL
cache.set("key", value, ttl=300)

# After 5 minutes, key automatically deleted
# Next request will be cache miss → query database
```

### Choosing Appropriate TTL

| Data Type | TTL | Rationale |
|-----------|-----|-----------|
| **Fraud Check Result** | 5 minutes | Results rarely change; balance freshness vs performance |
| **User Profile** | 1 hour | Profile changes infrequently; longer cache OK |
| **Device Fingerprint** | 2 hours | Very stable data; can cache longer |
| **Fraud Rules** | 24 hours | Rules change rarely; long cache acceptable |
| **Rate Limiting** | 1-5 minutes | Must be accurate; short TTL required |

### TTL Best Practices

**1. Shorter TTL for Critical Data**
```python
# Payment verification - 1 minute TTL
cache.set("payment:verification:123", data, ttl=60)

# Fraud score - 5 minutes TTL
cache.set("fraud:score:123", data, ttl=300)
```

**2. Longer TTL for Static Data**
```python
# Country list - 24 hours
cache.set("countries:list", data, ttl=86400)

# System configuration - 1 hour
cache.set("config:fraud_thresholds", data, ttl=3600)
```

**3. No TTL for Permanent Data**
```python
# API keys (until manually invalidated)
cache.set("api:key:abc123", data)  # No TTL
```

**4. Sliding Window for Active Data**
```python
# Refresh TTL on each access
data = cache.get("user:session:123")
if data:
    cache.expire("user:session:123", 1800)  # Reset to 30 minutes
```

---

## Testing & Performance Measurement

### Step 1: Start Your Application

```bash
# Ensure you're in project directory
cd /home/user/sentinel-api

# Activate virtual environment
source venv/bin/activate

# Start FastAPI server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Expected output:
# INFO:     Started server process
# INFO:     Waiting for application startup.
# INFO:     Application startup complete.
# INFO:     Uvicorn running on http://0.0.0.0:8000
```

---

### Step 2: Verify Redis Connection

```bash
# Open new terminal
redis-cli

# Test basic commands
127.0.0.1:6379> PING
PONG

# Check current keys (should be empty)
127.0.0.1:6379> KEYS sentinel:*
(empty array)

# Monitor Redis in real-time (optional)
127.0.0.1:6379> MONITOR
OK
# Now you'll see all Redis commands as they happen
```

---

### Step 3: Test Fraud Check with Caching

**First Request (Cache Miss):**

```bash
# Time the request
time curl -X POST "http://localhost:8000/api/v1/check-fraud" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_cache_test_001",
    "user_id": "user_cache_001",
    "amount": 150000,
    "currency": "NGN",
    "transaction_type": "WITHDRAWAL",
    "user_info": {
      "email": "cache@example.com",
      "phone_number": "+2348012345678",
      "account_age_days": 45
    },
    "device_info": {
      "device_id": "dev_cache_001",
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
  }'
```

**Expected Response:**

```json
{
  "transaction_id": "tx_cache_test_001",
  "fraud_score": 30,
  "risk_level": "LOW",
  "is_fraudulent": false,
  "fraud_flags": [
    {
      "flag_type": "large_amount",
      "severity": "HIGH",
      "description": "Transaction amount ₦150,000.00 exceeds ₦100,000 threshold"
    }
  ],
  "timestamp": "2024-01-15T10:30:00Z",
  "recommendation": "APPROVE - Transaction appears legitimate"
}
```

**Check application logs:**

```
INFO: Cache MISS for fraud check: tx_cache_test_001
INFO: Fraud check completed: tx_cache_test_001 (score: 30, risk: LOW)
```

**Time: ~45ms** (includes database query)

---

**Second Request (Cache Hit):**

```bash
# Same request again
time curl -X POST "http://localhost:8000/api/v1/check-fraud" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_cache_test_001",
    "user_id": "user_cache_001",
    "amount": 150000,
    "currency": "NGN",
    "transaction_type": "WITHDRAWAL",
    "user_info": {
      "email": "cache@example.com",
      "phone_number": "+2348012345678",
      "account_age_days": 45
    },
    "device_info": {
      "device_id": "dev_cache_001",
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
  }'
```

**Check application logs:**

```
INFO: Cache HIT for fraud check: tx_cache_test_001
```

**Time: ~2ms** (50x faster! ⚡)

---

### Step 4: Inspect Cache in Redis

```bash
# Connect to Redis CLI
redis-cli

# List all Sentinel cache keys
127.0.0.1:6379> KEYS sentinel:*
1) "sentinel:fraud_check:tx_cache_test_001:v1"

# Get cached value
127.0.0.1:6379> GET sentinel:fraud_check:tx_cache_test_001:v1
"{\"transaction_id\":\"tx_cache_test_001\",\"fraud_score\":30,\"risk_level\":\"LOW\",...}"

# Check TTL (time remaining)
127.0.0.1:6379> TTL sentinel:fraud_check:tx_cache_test_001:v1
(integer) 245  # 245 seconds remaining (out of 300)

# Check key type
127.0.0.1:6379> TYPE sentinel:fraud_check:tx_cache_test_001:v1
string

# Pretty print the cached JSON (using redis-cli)
127.0.0.1:6379> GET sentinel:fraud_check:tx_cache_test_001:v1 | jq .
```

---

### Step 5: Test Cache Expiration

```bash
# Set a test key with 10-second TTL
redis-cli

127.0.0.1:6379> SET test_key "test_value" EX 10
OK

127.0.0.1:6379> TTL test_key
(integer) 8  # 8 seconds remaining

# Wait 10 seconds...

127.0.0.1:6379> GET test_key
(nil)  # Key expired and auto-deleted

# Fraud check cache expires after 5 minutes (300 seconds)
127.0.0.1:6379> TTL sentinel:fraud_check:tx_cache_test_001:v1
(integer) 180  # 3 minutes remaining
```

---

### Step 6: Performance Comparison

**Test without cache (disabled):**

```bash
# Temporarily disable cache in .env
# CACHE_ENABLED=false

# Restart server
# Then run 10 requests

for i in {1..10}; do
  time curl -X POST "http://localhost:8000/api/v1/check-fraud" \
    -H "Content-Type: application/json" \
    -d "{\"transaction_id\":\"tx_perf_$i\", ...}" > /dev/null 2>&1
done

# Average time: ~45ms per request
```

**Test with cache (enabled):**

```bash
# Re-enable cache in .env
# CACHE_ENABLED=true

# Restart server

# First request (cache miss)
curl -X POST "http://localhost:8000/api/v1/check-fraud" -d '{...}'
# Time: 45ms

# Next 9 requests (cache hits)
for i in {2..10}; do
  time curl -X POST "http://localhost:8000/api/v1/check-fraud" -d '{...}' > /dev/null 2>&1
done

# Average time for cached requests: ~2ms
# Speedup: 45ms / 2ms = 22.5x faster!
```

---

### Step 7: Test Cache Invalidation

```bash
# Make a fraud check
curl -X POST "http://localhost:8000/api/v1/check-fraud" \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "tx_invalidate_test",
    "user_id": "user_001",
    ...
  }'

# Verify it's cached
redis-cli

127.0.0.1:6379> EXISTS sentinel:fraud_check:tx_invalidate_test:v1
(integer) 1  # Key exists

# Manually invalidate (simulate user profile update)
127.0.0.1:6379> DEL sentinel:fraud_check:tx_invalidate_test:v1
(integer) 1  # Key deleted

# Next request will be cache miss
127.0.0.1:6379> EXISTS sentinel:fraud_check:tx_invalidate_test:v1
(integer) 0  # Key gone
```

---

### Step 8: Test GET Endpoint Caching

```bash
# GET request (first time - cache miss)
time curl -X GET "http://localhost:8000/api/v1/transactions/tx_cache_test_001"

# Expected response:
{
  "transaction_id": "tx_cache_test_001",
  "fraud_score": 30,
  "risk_level": "LOW",
  ...
}

# Time: ~40ms (database query)

# GET request again (cache hit)
time curl -X GET "http://localhost:8000/api/v1/transactions/tx_cache_test_001"

# Time: ~2ms (cache hit!)
```

---

### Step 9: Test User History Caching

```bash
# Get user fraud history (cache miss)
time curl -X GET "http://localhost:8000/api/v1/users/user_cache_001/fraud-history?period_days=30"

# Expected response:
{
  "user_id": "user_cache_001",
  "period_days": 30,
  "total_transactions": 1,
  "flagged_transactions": 0,
  "total_amount": 150000,
  "average_fraud_score": 30,
  "current_risk_score": 30,
  "account_status": "ACTIVE"
}

# Time: ~50ms (multiple database queries)

# Same request (cache hit)
time curl -X GET "http://localhost:8000/api/v1/users/user_cache_001/fraud-history?period_days=30"

# Time: ~2ms (50/2 = 25x faster!)
```

---

### Step 10: Monitor Cache Hit Rate

```bash
# Get Redis statistics
redis-cli

127.0.0.1:6379> INFO stats
# Keyspace
keyspace_hits:150
keyspace_misses:10

# Calculate hit rate
# Hit Rate = hits / (hits + misses) = 150 / 160 = 93.75%
# Excellent! 93.75% of requests served from cache
```

**Good cache hit rates:**
- 90-95%: Excellent
- 80-90%: Good
- 70-80%: Fair (tune TTL)
- <70%: Poor (review caching strategy)

---

## Cache Invalidation Strategies

### The Two Hardest Things in Computer Science

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

Cache invalidation is critical for data consistency. Here are the strategies:

---

### 1. Time-Based Invalidation (TTL)

**Automatic expiration after TTL:**

```python
# Set with TTL
cache.set("user:123", data, ttl=3600)  # Expires in 1 hour

# Redis automatically deletes after 1 hour
# Next request = cache miss = fresh data from database
```

**Pros:**
- ✅ Automatic (no manual intervention)
- ✅ Simple to implement
- ✅ Prevents stale data

**Cons:**
- ⚠️ Data may be stale for up to TTL duration
- ⚠️ All cached data expires even if unchanged

**When to use:** Data that changes infrequently or when eventual consistency is acceptable.

---

### 2. Write-Through Invalidation

**Update cache when data changes:**

```python
def update_user_profile(user_id, new_data, db):
    # 1. Update database
    db.query(UserProfile).filter_by(user_id=user_id).update(new_data)
    db.commit()

    # 2. Invalidate cache immediately
    cache_key = CacheKeys.user_profile(user_id)
    cache.delete(cache_key)

    # Next read will fetch fresh data from database
```

**Pros:**
- ✅ Immediate consistency
- ✅ No stale data
- ✅ Predictable behavior

**Cons:**
- ⚠️ Requires code changes everywhere data is written
- ⚠️ More complex to implement

**When to use:** Critical data that must be immediately consistent.

---

### 3. Event-Based Invalidation

**Invalidate cache based on events:**

```python
# When fraud check is updated
@app.event_handler("fraud_check_updated")
def on_fraud_check_updated(transaction_id):
    # Invalidate fraud check cache
    cache.delete(CacheKeys.fraud_check(transaction_id))

    # Also invalidate related user cache
    cache.delete_pattern(f"sentinel:user_txs:*")
```

**Pros:**
- ✅ Decoupled from business logic
- ✅ Can invalidate related caches
- ✅ Flexible

**Cons:**
- ⚠️ Requires event system
- ⚠️ More complex architecture

**When to use:** Large applications with many interdependent caches.

---

### 4. Pattern-Based Invalidation

**Invalidate multiple related keys:**

```python
def update_user_data(user_id):
    # Update database
    # ...

    # Invalidate all user-related caches
    patterns = [
        f"sentinel:user_profile:{user_id}:*",
        f"sentinel:user_txs:{user_id}:*",
        f"sentinel:rate_limit:{user_id}:*",
    ]

    for pattern in patterns:
        cache.delete_pattern(pattern)
```

**Pros:**
- ✅ Invalidates related data
- ✅ Simple to use
- ✅ Prevents stale related data

**Cons:**
- ⚠️ Can be expensive (KEYS command scans all keys)
- ⚠️ May invalidate more than necessary

**When to use:** When updating data affects multiple cache entries.

---

### 5. Lazy Invalidation (Refresh on Read)

**Check data version on read:**

```python
def get_user_profile(user_id, db):
    cache_key = CacheKeys.user_profile(user_id)
    cached_data = cache.get(cache_key)

    if cached_data:
        # Check if cached data is from current version
        db_version = db.query(UserProfile.version).filter_by(user_id=user_id).scalar()

        if cached_data['version'] == db_version:
            return cached_data  # Still valid

        # Version mismatch - cache is stale
        cache.delete(cache_key)

    # Fetch fresh data
    fresh_data = db.query(UserProfile).filter_by(user_id=user_id).first()
    cache.set(cache_key, fresh_data, ttl=3600)
    return fresh_data
```

**Pros:**
- ✅ No stale data served
- ✅ Only invalidates when actually changed
- ✅ Efficient

**Cons:**
- ⚠️ Requires version field in database
- ⚠️ Extra database query on cache hit
- ⚠️ More complex logic

**When to use:** When you need strong consistency but want cache benefits.

---

### Cache Invalidation in Sentinel

**Our implementation:**

```python
# In fraud_service.py

def _update_user_profile(self, user_id, fraud_score, db):
    """Update user profile and invalidate cache."""
    # Update database
    profile = db.query(UserRiskProfile).filter_by(user_id=user_id).first()
    profile.risk_score = new_score
    db.commit()

    # Invalidate all user caches
    self.invalidate_user_cache(user_id)

def invalidate_user_cache(self, user_id):
    """Invalidate all cache entries for a user."""
    patterns = CacheKeys.invalidate_user(user_id)
    for pattern in patterns:
        cache_manager.delete(pattern)
```

**When we invalidate:**
1. When user profile is updated (risk score changes)
2. When new transaction is added (history changes)
3. Manual invalidation via admin API (future feature)

---

## Troubleshooting

### Issue 1: Redis Connection Failed

**Error:**

```
ERROR: Failed to connect to Redis: Connection refused
WARNING: Cache will be disabled. Application will work without caching.
```

**Solution:**

```bash
# Check if Redis is running
sudo systemctl status redis-server

# If not running, start it
sudo systemctl start redis-server

# For Docker
docker ps | grep redis
# If not running
docker start sentinel-redis

# Test connection
redis-cli ping
# Expected: PONG

# Check Redis logs
sudo journalctl -u redis-server -n 50
# Or for Docker
docker logs sentinel-redis
```

---

### Issue 2: Redis Module Not Found

**Error:**

```
ModuleNotFoundError: No module named 'redis'
```

**Solution:**

```bash
# Verify virtual environment is activated
which python
# Should show: /home/user/sentinel-api/venv/bin/python

# Install Redis
pip install redis==5.0.1 hiredis==2.2.3

# Verify installation
pip list | grep redis
# Expected:
# redis        5.0.1
# hiredis      2.2.3
```

---

### Issue 3: Cache Not Working

**Symptom:** All requests show "Cache MISS" in logs

**Solution 1: Check if cache is enabled**

```bash
# Check .env file
cat .env | grep CACHE_ENABLED
# Should be: CACHE_ENABLED=true

# If false, enable it
echo "CACHE_ENABLED=true" >> .env

# Restart server
```

**Solution 2: Check Redis connectivity**

```python
# Test in Python
from app.cache import cache_manager

# Try to set a value
result = cache_manager.set("test", {"data": "test"}, ttl=60)
print(f"Set result: {result}")  # Should be True

# Try to get it
value = cache_manager.get("test")
print(f"Get result: {value}")  # Should be {'data': 'test'}
```

**Solution 3: Check logs**

```bash
# Look for cache-related errors
tail -f /var/log/syslog | grep -i redis
# Or in application logs
# Look for "Redis connection pool initialized" on startup
```

---

### Issue 4: Cache Returning Stale Data

**Symptom:** Updated data in database but API returns old cached data

**Solution 1: Clear specific cache**

```bash
# Connect to Redis
redis-cli

# Delete specific key
127.0.0.1:6379> DEL sentinel:fraud_check:tx_123:v1
(integer) 1

# Or delete pattern
127.0.0.1:6379> KEYS sentinel:fraud_check:*
127.0.0.1:6379> DEL sentinel:fraud_check:tx_123:v1 sentinel:fraud_check:tx_456:v1
```

**Solution 2: Flush all cache (USE WITH CAUTION)**

```bash
redis-cli

127.0.0.1:6379> FLUSHDB
OK
# All cache cleared - next requests will rebuild cache
```

**Solution 3: Reduce TTL**

```bash
# In .env
CACHE_TTL_FRAUD_CHECK=60  # Reduce from 300 to 60 seconds

# Restart server
```

---

### Issue 5: Redis Memory Full

**Error:**

```
OOM command not allowed when used memory > 'maxmemory'
```

**Solution 1: Check memory usage**

```bash
redis-cli

127.0.0.1:6379> INFO memory
used_memory_human:250.5M
maxmemory:256M  # Only 5.5M left!
```

**Solution 2: Increase max memory**

```bash
# Edit Redis config
sudo nano /etc/redis/redis.conf

# Find and update maxmemory
maxmemory 512mb

# Restart Redis
sudo systemctl restart redis-server
```

**Solution 3: Set eviction policy**

```bash
# In redis.conf
maxmemory-policy allkeys-lru  # Evict least recently used keys

# Options:
# allkeys-lru: Evict any key, LRU
# volatile-lru: Evict keys with TTL, LRU
# allkeys-random: Evict any key, randomly
# volatile-ttl: Evict keys with shortest TTL
```

**Solution 4: Clear old cache**

```bash
redis-cli

# Delete old fraud checks (>1 hour)
127.0.0.1:6379> KEYS sentinel:fraud_check:*
# Manually delete old ones
127.0.0.1:6379> DEL sentinel:fraud_check:old_tx_1:v1
```

---

### Issue 6: Import Errors

**Error:**

```
ImportError: cannot import name 'cache_manager' from 'app.cache'
```

**Solution:**

```bash
# Verify cache module structure
tree app/cache/

# Should show:
# app/cache/
# ├── __init__.py
# ├── keys.py
# └── redis.py

# Check __init__.py exports
cat app/cache/__init__.py

# Should contain:
# from app.cache.redis import cache_manager, CacheManager
# from app.cache.keys import CacheKeys
# __all__ = ["cache_manager", "CacheManager", "CacheKeys"]
```

---

### Issue 7: JSON Serialization Error

**Error:**

```
TypeError: Object of type datetime is not JSON serializable
```

**Solution:**

Update cache manager to handle datetime:

```python
# In app/cache/redis.py

import json
from datetime import datetime

class DateTimeEncoder(json.JSONEncoder):
    """JSON encoder that handles datetime objects."""
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)

# Update set method
def set(self, key, value, ttl=None):
    serialized_value = json.dumps(value, cls=DateTimeEncoder)
    # ...
```

---

## Key Takeaways

### What You Built Today

1. **Redis Cache Layer**
   - Connection pooling for performance
   - Automatic serialization/deserialization
   - Configurable TTL strategies
   - Graceful degradation when Redis is down

2. **Cache-Aside Pattern**
   - Check cache first
   - On miss, query database
   - Store result in cache
   - Return to client

3. **Cache Key Strategy**
   - Consistent naming conventions
   - Versioned keys for schema changes
   - Pattern-based invalidation
   - Hierarchical organization

4. **Performance Improvements**
   - 50x faster response times for cached data
   - 90%+ reduction in database load
   - Sub-millisecond latency for cache hits

5. **Cache Invalidation**
   - Time-based (TTL)
   - Write-through (manual)
   - Pattern-based (bulk)
   - Event-driven (future)

### Project Status

```
✅ Day 1: FastAPI foundation
✅ Day 2: PostgreSQL database
✅ Day 3: Pydantic schemas
✅ Day 4: Rules engine
✅ Day 5: Complete fraud detection API
✅ Day 6: Redis caching layer ← YOU ARE HERE
⬜ Day 7: Advanced caching patterns
⬜ Day 8: Background jobs
⬜ Day 9: Rate limiting
```

### Skills Gained

- ✅ Redis setup and configuration
- ✅ Connection pooling
- ✅ Cache-aside pattern implementation
- ✅ TTL strategy design
- ✅ Cache key naming conventions
- ✅ Cache invalidation patterns
- ✅ Performance measurement and optimization
- ✅ Graceful degradation
- ✅ Redis CLI for debugging

---

## Next Steps

### Day 7: Advanced Caching Patterns

Tomorrow you'll build on today's foundation with advanced caching techniques:

**What you'll build:**

1. **Cache Warming**
   - Pre-populate cache on startup
   - Scheduled cache refresh
   - Prevent cold cache issues

2. **Cache Stampede Prevention**
   - Lock mechanism for concurrent requests
   - Prevent multiple database queries for same data
   - Exponential backoff

3. **Multi-Level Caching**
   - L1: In-memory cache (fastest, smallest)
   - L2: Redis cache (fast, larger)
   - L3: Database (slowest, complete)

4. **Cache Statistics Dashboard**
   - Hit/miss rates
   - Memory usage
   - Popular keys
   - Performance metrics

5. **Smart Cache Invalidation**
   - Dependency tracking
   - Cascading invalidation
   - Partial updates

### Performance Goals for Day 7

```
Current (Day 6):
  - Cache hit: 2ms
  - Cache miss: 45ms
  - Hit rate: 90%

Day 7 Goals:
  - Cache hit: 0.5ms (4x faster - in-memory L1 cache)
  - Cache miss: 45ms (same)
  - Hit rate: 95% (cache warming)
  - Zero cache stampedes (locking)
```

### Recommended Reading

1. **Redis Documentation**
   - [Redis Commands](https://redis.io/commands/)
   - [Redis Data Types](https://redis.io/topics/data-types)
   - [Redis Persistence](https://redis.io/topics/persistence)

2. **Caching Best Practices**
   - [Caching Strategies](https://aws.amazon.com/caching/best-practices/)
   - [Cache Invalidation](https://martinfowler.com/bliki/TwoHardThings.html)

3. **Performance Optimization**
   - [High Performance Browser Networking](https://hpbn.co/)
   - [Redis Performance Tuning](https://redis.io/topics/latency)

### Practice Exercises

**Exercise 1: Implement Cache Statistics Endpoint**

```python
# Create endpoint: GET /api/v1/cache/stats
# Returns:
{
  "redis_version": "7.0.0",
  "used_memory_human": "2.5M",
  "total_keys": 150,
  "hit_rate": "93.5%",
  "cache_enabled": true
}
```

**Exercise 2: Add Cache Clear Endpoint**

```python
# Create endpoint: DELETE /api/v1/cache
# Clears all cache (admin only)
# Returns: {"message": "Cache cleared", "keys_deleted": 150}
```

**Exercise 3: Implement Rate Limiting**

```python
def check_rate_limit(user_id: str) -> bool:
    key = f"rate_limit:{user_id}:1m"
    count = cache_manager.increment(key)

    if count == 1:
        cache_manager.expire(key, 60)

    return count <= 100  # 100 requests per minute
```

**Exercise 4: Cache Warming on Startup**

```python
# Pre-load popular fraud rules on startup
@app.on_event("startup")
async def warm_cache():
    # Load fraud rules
    rules = load_fraud_rules()
    cache_manager.set(
        CacheKeys.fraud_rules("default"),
        rules,
        ttl=86400
    )
```

---

**Navigation:** [← Previous: Day 5](README-DAY-005.md) | [Main Guide](README.md) | [Next: Day 7 →](README-DAY-007.md)

---

**Congratulations!** 🎉

You've successfully implemented Redis caching and achieved **50x performance improvements**! Your fraud detection API now:

✅ Caches fraud check results (5-minute TTL)
✅ Caches user profiles (1-hour TTL)
✅ Uses connection pooling for efficiency
✅ Implements cache-aside pattern
✅ Handles cache invalidation
✅ Degrades gracefully when Redis is down
✅ Serves requests in <2ms (cached) vs 45ms (uncached)

Tomorrow we'll take caching to the next level with cache warming, stampede prevention, and multi-level caching!

Keep building! 🚀

---

**Questions or Issues?**

If you encounter problems:
1. Check the [Troubleshooting](#troubleshooting) section
2. Verify Redis is running: `redis-cli ping`
3. Check logs for cache-related errors
4. Test cache with `redis-cli MONITOR`
5. Ensure CACHE_ENABLED=true in .env

**Happy caching!** ⚡
