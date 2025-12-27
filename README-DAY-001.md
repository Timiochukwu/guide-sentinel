# Day 1: Project Foundation & FastAPI Setup

**Navigation:** [← Previous: Main Guide](README.md) | [Main Guide](README.md) | [Next: Day 2 →](README-DAY-002.md)

## Overview

Welcome to Day 1 of building Sentinel, a production-grade fraud detection platform! Today, we'll establish the foundation of our project by setting up a FastAPI application with proper project structure, environment configuration, and our first API endpoints.

**What You'll Build Today:**
- A properly structured Python project with virtual environment
- FastAPI application with health check and info endpoints
- Environment-based configuration system
- Comprehensive development tooling setup

**Time Estimate:** 2-3 hours

**Prerequisites:**
- Python 3.9+ installed
- Basic understanding of Python
- Terminal/command line familiarity
- Text editor or IDE (VS Code, PyCharm, etc.)

---

## Table of Contents

1. [Understanding FastAPI](#understanding-fastapi)
2. [Project Structure Setup](#project-structure-setup)
3. [Complete Code Implementation](#complete-code-implementation)
4. [Step-by-Step Installation](#step-by-step-installation)
5. [Running the Application](#running-the-application)
6. [Testing Your Endpoints](#testing-your-endpoints)
7. [Understanding the Code](#understanding-the-code)
8. [Troubleshooting](#troubleshooting)
9. [Next Steps](#next-steps)

---

## Understanding FastAPI

### What is FastAPI?

FastAPI is a modern, high-performance web framework for building APIs with Python 3.9+ based on standard Python type hints. It's built on top of Starlette for web routing and Pydantic for data validation.

**Key Features:**
- **Fast Performance:** One of the fastest Python frameworks (comparable to NodeJS and Go)
- **Type Safety:** Uses Python type hints for automatic validation
- **Auto Documentation:** Generates interactive API docs (Swagger UI & ReDoc)
- **Async Support:** Native async/await support for high concurrency
- **Modern Python:** Leverages the latest Python features

**Why FastAPI for Fraud Detection?**

For a fraud detection platform like Sentinel, we need:
1. **High throughput** - Process thousands of transactions per second
2. **Low latency** - Real-time fraud scoring (< 100ms response time)
3. **Data validation** - Strict validation of transaction data
4. **Easy integration** - RESTful API for microservices architecture

FastAPI excels in all these areas.

### ASGI vs WSGI

Understanding the difference between ASGI and WSGI is crucial for modern Python web development.

**WSGI (Web Server Gateway Interface):**
```
Traditional synchronous protocol for Python web apps
Request → WSGI Server → Application (blocks) → Response
Examples: Django (pre-async), Flask, Gunicorn
```

**ASGI (Asynchronous Server Gateway Interface):**
```
Modern asynchronous protocol for Python web apps
Request → ASGI Server → Application (async) → Response
Examples: FastAPI, Starlette, Django (3.0+)
```

**Key Differences:**

| Feature | WSGI | ASGI |
|---------|------|------|
| Async Support | No | Yes |
| WebSockets | No | Yes |
| HTTP/2 | No | Yes |
| Concurrency | Thread/Process | Event Loop |
| Performance | Good | Excellent |

**Example: Handling 1000 concurrent requests**

WSGI (synchronous):
```python
# Each request blocks a worker thread
# Needs 1000 threads or processes
def handle_request():
    result = slow_database_query()  # Blocks thread
    return result
```

ASGI (asynchronous):
```python
# All requests share event loop
# Needs only a few workers
async def handle_request():
    result = await slow_database_query()  # Releases control
    return result
```

### How Async/Await Works

Asynchronous programming allows your application to handle multiple operations concurrently without blocking.

**Synchronous Flow:**
```python
# Blocking operations - each waits for completion
def check_transaction(tx_id):
    user = fetch_user_from_db(tx_id)      # Wait 50ms
    history = fetch_history_from_db(user)  # Wait 100ms
    risk = call_ml_model(history)          # Wait 200ms
    return risk
# Total time: 350ms per transaction
# 1000 transactions = 350,000ms = 5.8 minutes
```

**Asynchronous Flow:**
```python
# Non-blocking operations - concurrent execution
async def check_transaction(tx_id):
    user = await fetch_user_from_db(tx_id)      # Yield control during wait
    history = await fetch_history_from_db(user)  # Yield control during wait
    risk = await call_ml_model(history)          # Yield control during wait
    return risk
# Total time: 350ms per transaction
# 1000 transactions (concurrent) ≈ 400ms total
```

**The Event Loop:**
```
┌─────────────────────────────┐
│      Event Loop             │
├─────────────────────────────┤
│ Task 1: ████████░░░░ (60%)  │ ← Waiting for DB
│ Task 2: ██░░░░░░░░░░ (20%)  │ ← Waiting for API
│ Task 3: ████████████ (100%) │ ← Ready to return
│ Task 4: ██████░░░░░░ (50%)  │ ← Waiting for ML
└─────────────────────────────┘
```

### API Endpoints and HTTP Methods

RESTful APIs use HTTP methods to indicate the action to perform:

```
GET     /transactions          - Retrieve list of transactions
GET     /transactions/{id}     - Retrieve specific transaction
POST    /transactions          - Create new transaction
PUT     /transactions/{id}     - Update entire transaction
PATCH   /transactions/{id}     - Update partial transaction
DELETE  /transactions/{id}     - Delete transaction
```

**Today's Endpoints:**

1. **GET /health** - Health check endpoint
   - Purpose: Verify service is running
   - Use case: Load balancers, monitoring systems
   - Response: Simple status message

2. **GET /api/v1/info** - Service information
   - Purpose: Provide service metadata
   - Use case: Service discovery, debugging
   - Response: Service name, version, environment

---

## Project Structure Setup

We'll create a well-organized project structure that scales:

```
sentinel/
├── app/                    # Application package
│   ├── __init__.py        # Package initializer
│   └── main.py            # FastAPI application entry point
├── tests/                 # Test suite (coming soon)
│   └── __init__.py
├── .env                   # Environment variables (DO NOT commit)
├── .env.example           # Environment template (commit this)
├── .gitignore             # Git ignore rules
├── requirements.txt       # Python dependencies
└── README.md              # Project documentation
```

**Why This Structure?**

- **app/**: Keeps application code organized and importable
- **tests/**: Separates test code from application code
- **.env**: Local configuration (secrets, database URLs)
- **requirements.txt**: Reproducible dependency management

---

## Complete Code Implementation

### 1. app/__init__.py

This file makes the `app` directory a Python package and can include package-level initialization.

```python
"""
Sentinel Fraud Detection Platform

A production-grade fraud detection system built with FastAPI.
This package contains the core application logic for detecting
fraudulent transactions in real-time.

Author: Sentinel Team
Version: 0.1.0
"""

__version__ = "0.1.0"
__author__ = "Sentinel Team"
__description__ = "Real-time fraud detection platform"

# Package-level exports
# When someone imports from app, these will be available
__all__ = [
    "__version__",
    "__author__",
    "__description__",
]
```

### 2. app/main.py

This is the heart of our application - the FastAPI instance and our first endpoints.

```python
"""
FastAPI Application Entry Point

This module initializes the FastAPI application and defines
the core API endpoints for the Sentinel fraud detection platform.

The application uses async/await for high-performance concurrent
request handling and includes automatic API documentation via
OpenAPI/Swagger.
"""

import os
from datetime import datetime
from typing import Dict, Any

from fastapi import FastAPI, status
from fastapi.responses import JSONResponse
import uvicorn


# ============================================================================
# APPLICATION INITIALIZATION
# ============================================================================

# Create FastAPI application instance
app = FastAPI(
    title="Sentinel Fraud Detection API",
    description=(
        "A production-grade fraud detection platform that analyzes "
        "transactions in real-time to identify and prevent fraudulent activity."
    ),
    version="0.1.0",
    docs_url="/docs",  # Swagger UI at http://localhost:8000/docs
    redoc_url="/redoc",  # ReDoc at http://localhost:8000/redoc
    openapi_url="/openapi.json",  # OpenAPI schema
)


# ============================================================================
# CONFIGURATION
# ============================================================================

# Load environment variables with defaults
ENV = os.getenv("ENVIRONMENT", "development")
DEBUG = os.getenv("DEBUG", "True").lower() == "true"
HOST = os.getenv("HOST", "0.0.0.0")
PORT = int(os.getenv("PORT", "8000"))


# ============================================================================
# LIFECYCLE EVENTS
# ============================================================================

@app.on_event("startup")
async def startup_event():
    """
    Application startup event handler.

    This function runs once when the application starts.
    Use it for:
    - Database connection initialization
    - Cache warm-up
    - Loading ML models
    - Establishing external service connections
    """
    print("=" * 60)
    print("🚀 Sentinel Fraud Detection API Starting...")
    print("=" * 60)
    print(f"Environment: {ENV}")
    print(f"Debug Mode: {DEBUG}")
    print(f"Listening on: {HOST}:{PORT}")
    print(f"Documentation: http://localhost:{PORT}/docs")
    print("=" * 60)


@app.on_event("shutdown")
async def shutdown_event():
    """
    Application shutdown event handler.

    This function runs once when the application shuts down.
    Use it for:
    - Closing database connections
    - Flushing caches
    - Cleanup operations
    - Logging final metrics
    """
    print("=" * 60)
    print("🛑 Sentinel Fraud Detection API Shutting Down...")
    print("=" * 60)


# ============================================================================
# API ENDPOINTS
# ============================================================================

@app.get(
    "/health",
    status_code=status.HTTP_200_OK,
    tags=["Health"],
    summary="Health Check",
    response_description="Service health status",
)
async def health_check() -> Dict[str, Any]:
    """
    Health check endpoint.

    This endpoint is used by:
    - Load balancers to check if the service is alive
    - Monitoring systems (Prometheus, Datadog, etc.)
    - Container orchestration platforms (Kubernetes, Docker Swarm)
    - CI/CD pipelines to verify deployment

    Returns:
        dict: Health status information including:
            - status: Service status ("healthy")
            - timestamp: Current server time (ISO 8601 format)
            - environment: Current environment (dev/staging/production)

    Example Response:
        {
            "status": "healthy",
            "timestamp": "2024-01-15T10:30:00.123456",
            "environment": "development"
        }
    """
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "environment": ENV,
    }


@app.get(
    "/api/v1/info",
    status_code=status.HTTP_200_OK,
    tags=["Information"],
    summary="Service Information",
    response_description="Detailed service information",
)
async def service_info() -> Dict[str, Any]:
    """
    Service information endpoint.

    Provides detailed information about the API service including
    version, capabilities, and configuration. Useful for:
    - Service discovery in microservices architecture
    - Debugging and troubleshooting
    - Version verification during deployment
    - Client compatibility checking

    Returns:
        dict: Service information including:
            - service: Service name
            - version: Current API version
            - environment: Deployment environment
            - debug_mode: Whether debug mode is enabled
            - endpoints: Available endpoint information
            - timestamp: Current server time
            - uptime_check: Health status

    Example Response:
        {
            "service": "Sentinel Fraud Detection API",
            "version": "0.1.0",
            "environment": "development",
            "debug_mode": true,
            "endpoints": {
                "health": "/health",
                "info": "/api/v1/info",
                "docs": "/docs",
                "redoc": "/redoc"
            },
            "timestamp": "2024-01-15T10:30:00.123456",
            "uptime_check": "healthy"
        }
    """
    return {
        "service": "Sentinel Fraud Detection API",
        "version": "0.1.0",
        "environment": ENV,
        "debug_mode": DEBUG,
        "endpoints": {
            "health": "/health",
            "info": "/api/v1/info",
            "docs": "/docs",
            "redoc": "/redoc",
        },
        "timestamp": datetime.utcnow().isoformat(),
        "uptime_check": "healthy",
    }


# ============================================================================
# APPLICATION RUNNER
# ============================================================================

if __name__ == "__main__":
    """
    Direct execution entry point.

    This allows running the application directly with:
        python app/main.py

    In production, you would typically use:
        uvicorn app.main:app --host 0.0.0.0 --port 8000
    """
    uvicorn.run(
        "app.main:app",
        host=HOST,
        port=PORT,
        reload=DEBUG,  # Auto-reload on code changes (development only)
        log_level="info",
    )
```

### 3. .env.example

Template for environment variables. Users copy this to `.env` and customize.

```bash
# ============================================================================
# Sentinel Fraud Detection Platform - Environment Configuration
# ============================================================================
#
# INSTRUCTIONS:
# 1. Copy this file to .env:
#    cp .env.example .env
#
# 2. Update the values in .env with your actual configuration
#
# 3. NEVER commit .env to version control (it's in .gitignore)
#
# ============================================================================

# ----------------------------------------------------------------------------
# Application Settings
# ----------------------------------------------------------------------------

# Environment: development, staging, production
ENVIRONMENT=development

# Enable debug mode (verbose logging, auto-reload)
# WARNING: Never set to True in production!
DEBUG=True

# Server configuration
HOST=0.0.0.0
PORT=8000

# ----------------------------------------------------------------------------
# Database Configuration (for future use)
# ----------------------------------------------------------------------------

# PostgreSQL connection string
# Format: postgresql+asyncpg://user:password@host:port/database
# DATABASE_URL=postgresql+asyncpg://sentinel:password@localhost:5432/sentinel_db

# Database pool settings
# DB_POOL_SIZE=20
# DB_MAX_OVERFLOW=10

# ----------------------------------------------------------------------------
# Redis Configuration (for future use)
# ----------------------------------------------------------------------------

# Redis connection for caching and rate limiting
# REDIS_URL=redis://localhost:6379/0

# ----------------------------------------------------------------------------
# Security Settings (for future use)
# ----------------------------------------------------------------------------

# Secret key for JWT token signing (generate with: openssl rand -hex 32)
# SECRET_KEY=your-secret-key-here

# JWT token expiration in minutes
# ACCESS_TOKEN_EXPIRE_MINUTES=30

# API key for external services
# API_KEY=your-api-key-here

# ----------------------------------------------------------------------------
# Machine Learning Model Configuration (for future use)
# ----------------------------------------------------------------------------

# Path to trained model files
# MODEL_PATH=/app/models/fraud_detector_v1.pkl

# Model inference settings
# MODEL_THRESHOLD=0.75
# MODEL_BATCH_SIZE=32

# ----------------------------------------------------------------------------
# Logging Configuration
# ----------------------------------------------------------------------------

# Log level: DEBUG, INFO, WARNING, ERROR, CRITICAL
LOG_LEVEL=INFO

# Log format: json, text
LOG_FORMAT=text

# ----------------------------------------------------------------------------
# External Services (for future use)
# ----------------------------------------------------------------------------

# Email service for alerts
# SMTP_HOST=smtp.gmail.com
# SMTP_PORT=587
# SMTP_USERNAME=alerts@sentinel.com
# SMTP_PASSWORD=your-password

# Monitoring and observability
# SENTRY_DSN=https://your-sentry-dsn
# DATADOG_API_KEY=your-datadog-api-key

# ----------------------------------------------------------------------------
# Feature Flags (for future use)
# ----------------------------------------------------------------------------

# Enable specific features
# ENABLE_RATE_LIMITING=true
# ENABLE_CACHING=true
# ENABLE_METRICS=true
```

### 4. .gitignore

Comprehensive Git ignore rules for Python projects.

```gitignore
# ============================================================================
# Sentinel Fraud Detection Platform - Git Ignore Rules
# ============================================================================

# ----------------------------------------------------------------------------
# Python
# ----------------------------------------------------------------------------

# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]
*$py.class

# C extensions
*.so

# Distribution / packaging
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
share/python-wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# PyInstaller
*.manifest
*.spec

# Unit test / coverage reports
htmlcov/
.tox/
.nox/
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.log
.hypothesis/
.pytest_cache/
cover/

# Translations
*.mo
*.pot

# Django stuff:
*.log
local_settings.py
db.sqlite3
db.sqlite3-journal

# Flask stuff:
instance/
.webassets-cache

# Scrapy stuff:
.scrapy

# Sphinx documentation
docs/_build/

# PyBuilder
.pybuilder/
target/

# Jupyter Notebook
.ipynb_checkpoints
*.ipynb

# IPython
profile_default/
ipython_config.py

# pyenv
.python-version

# pipenv
Pipfile.lock

# poetry
poetry.lock

# pdm
.pdm.toml

# PEP 582
__pypackages__/

# Celery stuff
celerybeat-schedule
celerybeat.pid

# SageMath parsed files
*.sage.py

# Virtual environments
env/
venv/
ENV/
env.bak/
venv.bak/
.venv/

# Spyder project settings
.spyderproject
.spyproject

# Rope project settings
.ropeproject

# mkdocs documentation
/site

# mypy
.mypy_cache/
.dmypy.json
dmypy.json

# Pyre type checker
.pyre/

# pytype static type analyzer
.pytype/

# Cython debug symbols
cython_debug/

# ----------------------------------------------------------------------------
# Environment Variables & Secrets
# ----------------------------------------------------------------------------

.env
.env.local
.env.*.local
*.key
*.pem
secrets.yaml
secrets.json

# ----------------------------------------------------------------------------
# IDEs and Editors
# ----------------------------------------------------------------------------

# VSCode
.vscode/
*.code-workspace

# PyCharm
.idea/
*.iml
*.iws

# Sublime Text
*.sublime-project
*.sublime-workspace

# Vim
*.swp
*.swo
*~

# Emacs
*~
\#*\#
.\#*

# ----------------------------------------------------------------------------
# Operating System
# ----------------------------------------------------------------------------

# macOS
.DS_Store
.AppleDouble
.LSOverride
Icon
._*

# Windows
Thumbs.db
Thumbs.db:encryptable
ehthumbs.db
ehthumbs_vista.db
*.stackdump
[Dd]esktop.ini

# Linux
*~

# ----------------------------------------------------------------------------
# Application Specific
# ----------------------------------------------------------------------------

# Logs
logs/
*.log

# Database
*.db
*.sqlite
*.sqlite3

# ML Models (large files - use Git LFS if needed)
models/*.pkl
models/*.h5
models/*.pt
models/*.pth

# Data files
data/raw/
data/processed/
*.csv
*.parquet

# Temporary files
tmp/
temp/
*.tmp

# ----------------------------------------------------------------------------
# Docker
# ----------------------------------------------------------------------------

# Docker files (if not using them yet)
# Uncomment when you add Docker support
# docker-compose.override.yml
# .dockerignore

# ----------------------------------------------------------------------------
# CI/CD
# ----------------------------------------------------------------------------

.github/workflows/*.yaml.bak

# ----------------------------------------------------------------------------
# Misc
# ----------------------------------------------------------------------------

.DS_Store
*.bak
*.swp
*~
```

### 5. requirements.txt

Python dependencies with specific versions for reproducibility.

```txt
# Day 1 - FastAPI Basics Only
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-dotenv==1.0.0
```

---

## Step-by-Step Installation

### Step 1: Create Project Directory

Open your terminal and create the project directory:

```bash
# Create project directory
mkdir -p ~/sentinel
cd ~/sentinel

# Verify you're in the correct directory
pwd
# Expected output: /home/yourusername/sentinel (or C:\Users\YourName\sentinel on Windows)
```

### Step 2: Create Project Structure

Create all necessary directories and files:

```bash
# Create application package directory
mkdir -p app

# Create tests directory
mkdir -p tests

# Create empty __init__.py files to make directories Python packages
touch app/__init__.py
touch tests/__init__.py

# Verify directory structure
tree .
# Or use ls -R if tree is not installed
ls -R
```

Expected output:
```
.
├── app
│   └── __init__.py
└── tests
    └── __init__.py
```

### Step 3: Create Configuration Files

Create the requirements file:

```bash
# Create requirements.txt
cat > requirements.txt << 'EOF'
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-dotenv==1.0.0
EOF
```

Create the .env.example file:

```bash
# Create .env.example
cat > .env.example << 'EOF'
ENVIRONMENT=development
DEBUG=True
HOST=0.0.0.0
PORT=8000
LOG_LEVEL=INFO
LOG_FORMAT=text
EOF
```

Create your local .env file:

```bash
# Copy .env.example to .env
cp .env.example .env

# View the file to verify
cat .env
```

Create .gitignore:

```bash
# Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.py[cod]
*$py.class
.env
.env.local
venv/
.venv/
env/
*.log
.pytest_cache/
.coverage
htmlcov/
.idea/
.vscode/
*.swp
.DS_Store
EOF
```

### Step 4: Set Up Virtual Environment

Create and activate a Python virtual environment:

**On macOS/Linux:**
```bash
# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Verify activation (you should see (venv) in your prompt)
which python
# Expected: /home/yourusername/sentinel/venv/bin/python
```

**On Windows (Command Prompt):**
```cmd
# Create virtual environment
python -m venv venv

# Activate virtual environment
venv\Scripts\activate.bat

# Verify activation
where python
```

**On Windows (PowerShell):**
```powershell
# Create virtual environment
python -m venv venv

# Activate virtual environment
venv\Scripts\Activate.ps1

# If you get an execution policy error, run:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Then try activating again
venv\Scripts\Activate.ps1
```

**Why Virtual Environments?**

Virtual environments isolate your project dependencies:
- Prevents conflicts between different projects
- Makes your project reproducible
- Allows different Python versions per project
- Keeps your global Python installation clean

### Step 5: Install Dependencies

With the virtual environment activated:

```bash
# Upgrade pip to latest version
pip install --upgrade pip

# Install all dependencies
pip install -r requirements.txt

# This will take 1-2 minutes and install approximately 50+ packages

# Verify installation
pip list
# You should see fastapi, uvicorn, pydantic, etc.

# Check specific packages
python -c "import fastapi; print(f'FastAPI version: {fastapi.__version__}')"
# Expected output: FastAPI version: 0.104.1
```

### Step 6: Create Application Files

Now create your application code files using your text editor, or use these commands:

**Create app/__init__.py:**

```bash
cat > app/__init__.py << 'EOF'
"""
Sentinel Fraud Detection Platform

A production-grade fraud detection system built with FastAPI.
This package contains the core application logic for detecting
fraudulent transactions in real-time.

Author: Sentinel Team
Version: 0.1.0
"""

__version__ = "0.1.0"
__author__ = "Sentinel Team"
__description__ = "Real-time fraud detection platform"

__all__ = [
    "__version__",
    "__author__",
    "__description__",
]
EOF
```

**Create app/main.py** (copy the complete code from section 2 above, or use your text editor)

Due to length, it's better to create this file using a text editor. Open your editor and create `app/main.py` with the complete code from the "Complete Code Implementation" section above.

### Step 7: Verify Installation

Check that everything is set up correctly:

```bash
# Check directory structure
ls -la

# Expected files:
# app/
# tests/
# .env
# .env.example
# .gitignore
# requirements.txt
# venv/

# Verify Python can import your app
python -c "from app import main; print('✓ Import successful')"

# Verify environment variables
python -c "import os; from dotenv import load_dotenv; load_dotenv(); print(f'Environment: {os.getenv(\"ENVIRONMENT\")}')"
# Expected: Environment: development
```

---

## Running the Application

### Method 1: Using Uvicorn Directly (Recommended)

This is the standard way to run ASGI applications:

```bash
# Make sure your virtual environment is activated
# You should see (venv) in your prompt

# Run the application
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Explanation of flags:
# app.main:app  - Import path: app/main.py, object named 'app'
# --reload      - Auto-reload on code changes (development only)
# --host        - Listen on all network interfaces
# --port        - Port number to listen on
```

Expected output:
```
INFO:     Will watch for changes in these directories: ['/home/user/sentinel']
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [12345] using WatchFiles
INFO:     Started server process [12346]
INFO:     Waiting for application startup.
============================================================
🚀 Sentinel Fraud Detection API Starting...
============================================================
Environment: development
Debug Mode: True
Listening on: 0.0.0.0:8000
Documentation: http://localhost:8000/docs
============================================================
INFO:     Application startup complete.
```

### Method 2: Using Python Directly

Run the main.py file directly:

```bash
python app/main.py
```

This uses the `if __name__ == "__main__"` block in main.py and produces the same result.

### Method 3: Background Process (Linux/macOS)

Run the server in the background:

```bash
# Start in background
nohup uvicorn app.main:app --host 0.0.0.0 --port 8000 > server.log 2>&1 &

# Get process ID
echo $!

# View logs
tail -f server.log

# Stop the server (replace PID with actual process ID)
kill <PID>
```

### Stopping the Server

To stop the development server:

```bash
# Press CTRL+C in the terminal where the server is running
# You should see:
INFO:     Shutting down
============================================================
🛑 Sentinel Fraud Detection API Shutting Down...
============================================================
INFO:     Finished server process [12346]
```

---

## Testing Your Endpoints

### Automatic Interactive Documentation

FastAPI automatically generates interactive API documentation. With your server running, visit:

**Swagger UI (recommended for testing):**
```
http://localhost:8000/docs
```

Features:
- Try out endpoints directly in the browser
- See request/response schemas
- View example responses
- No additional tools needed

**ReDoc (alternative documentation):**
```
http://localhost:8000/redoc
```

Features:
- Clean, readable documentation
- Better for sharing with stakeholders
- Three-column layout

### Using cURL (Command Line)

cURL is a command-line tool for making HTTP requests. Perfect for scripting and automation.

**Test health endpoint:**

```bash
curl http://localhost:8000/health

# Expected response:
# {"status":"healthy","timestamp":"2024-01-15T10:30:00.123456","environment":"development"}
```

**Test with pretty printing (requires jq):**

```bash
# Install jq if not available:
# Ubuntu/Debian: sudo apt-get install jq
# macOS: brew install jq
# Windows: choco install jq

curl -s http://localhost:8000/health | jq

# Expected output (formatted):
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00.123456",
  "environment": "development"
}
```

**Test info endpoint:**

```bash
curl http://localhost:8000/api/v1/info | jq

# Expected response:
{
  "service": "Sentinel Fraud Detection API",
  "version": "0.1.0",
  "environment": "development",
  "debug_mode": true,
  "endpoints": {
    "health": "/health",
    "info": "/api/v1/info",
    "docs": "/docs",
    "redoc": "/redoc"
  },
  "timestamp": "2024-01-15T10:30:00.123456",
  "uptime_check": "healthy"
}
```

**Test with verbose output (see headers):**

```bash
curl -v http://localhost:8000/health

# Shows:
# - Request headers sent
# - Response headers received
# - Response body
```

**Save response to file:**

```bash
curl http://localhost:8000/api/v1/info -o response.json

# View file
cat response.json
```

### Using HTTPie (More User-Friendly)

HTTPie is a modern, user-friendly HTTP client.

**Install HTTPie:**

```bash
# Using pip
pip install httpie

# Or system package manager
# Ubuntu: sudo apt-get install httpie
# macOS: brew install httpie
```

**Make requests:**

```bash
# GET request (simple syntax)
http localhost:8000/health

# Output is automatically formatted and colorized
HTTP/1.1 200 OK
content-length: 85
content-type: application/json
date: Mon, 15 Jan 2024 10:30:00 GMT
server: uvicorn

{
    "environment": "development",
    "status": "healthy",
    "timestamp": "2024-01-15T10:30:00.123456"
}

# Test info endpoint
http localhost:8000/api/v1/info
```



---

## Understanding the Code

### FastAPI Application Instance

```python
app = FastAPI(
    title="Sentinel Fraud Detection API",
    description="...",
    version="0.1.0",
)
```

**What this does:**
- Creates an ASGI application instance
- Configures OpenAPI documentation
- Sets up automatic request/response validation
- Enables interactive docs at /docs and /redoc

### Lifecycle Events

```python
@app.on_event("startup")
async def startup_event():
    # Runs once when app starts
    print("Starting up...")

@app.on_event("shutdown")
async def shutdown_event():
    # Runs once when app stops
    print("Shutting down...")
```

**Use cases:**
- Startup: Connect to database, load ML models, warm cache
- Shutdown: Close connections, flush logs, cleanup resources

### Route Decorators

```python
@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

**Breakdown:**
- `@app.get`: Decorator that registers a route handler
- `"/health"`: URL path pattern
- `async def`: Async function (can use await)
- Return value: Automatically converted to JSON response

### Type Hints and Return Types

```python
async def health_check() -> Dict[str, Any]:
    return {"status": "healthy"}
```

**Benefits:**
- IDE autocomplete and type checking
- FastAPI uses hints for validation
- Better code documentation
- Runtime error prevention

### Status Codes

```python
@app.get("/health", status_code=status.HTTP_200_OK)
```

**Common status codes:**
- 200 OK: Successful request
- 201 Created: Resource created successfully
- 400 Bad Request: Invalid input
- 401 Unauthorized: Authentication required
- 404 Not Found: Resource doesn't exist
- 500 Internal Server Error: Server error

### Async vs Sync Functions

```python
# Async - can handle concurrent requests
async def async_endpoint():
    data = await fetch_from_db()  # Non-blocking
    return data

# Sync - blocks during execution
def sync_endpoint():
    data = fetch_from_db()  # Blocking
    return data
```

**When to use async:**
- Database queries
- External API calls
- File I/O operations
- Any I/O-bound work

**When sync is okay:**
- Pure computation (CPU-bound)
- Simple responses with no I/O
- Quick operations (<1ms)

---

## Troubleshooting

### Issue 1: Port Already in Use

**Error message:**
```
Error: [Errno 48] Address already in use
```

**Solution:**

Find and kill the process using the port:

```bash
# On Linux/macOS
lsof -i :8000
# Find the PID and kill it
kill -9 <PID>

# Or use a different port
uvicorn app.main:app --port 8001
```

```powershell
# On Windows (PowerShell)
netstat -ano | findstr :8000
# Find the PID and kill it
taskkill /PID <PID> /F
```

### Issue 2: Module Not Found

**Error message:**
```
ModuleNotFoundError: No module named 'fastapi'
```

**Solution:**

Ensure virtual environment is activated and dependencies are installed:

```bash
# Check if venv is activated (should see (venv) in prompt)
which python  # Should point to venv/bin/python

# If not activated, activate it
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows

# Reinstall dependencies
pip install -r requirements.txt

# Verify installation
pip list | grep fastapi
```

### Issue 3: Import Error from app

**Error message:**
```
ImportError: cannot import name 'main' from 'app'
```

**Solution:**

Check that you're running from the project root:

```bash
# Verify current directory
pwd
# Should be: /path/to/sentinel

# Verify app/main.py exists
ls app/main.py

# Run from project root
uvicorn app.main:app --reload
```

### Issue 4: Permission Denied (Port < 1024)

**Error message:**
```
PermissionError: [Errno 13] Permission denied
```

**Solution:**

Ports below 1024 require root/admin privileges. Use a higher port:

```bash
# In .env file, change:
PORT=8000  # Instead of PORT=80

# Or specify when running:
uvicorn app.main:app --port 8000
```

### Issue 5: Uvicorn Not Found

**Error message:**
```
uvicorn: command not found
```

**Solution:**

```bash
# Ensure virtual environment is activated
source venv/bin/activate

# Reinstall uvicorn
pip install uvicorn[standard]

# Verify installation
which uvicorn
# Should show: /path/to/sentinel/venv/bin/uvicorn
```

### Issue 6: Auto-Reload Not Working

**Problem:** Code changes don't reflect in running application

**Solution:**

```bash
# Ensure --reload flag is set
uvicorn app.main:app --reload

# If still not working, check you're editing the right file
# And restart the server manually (CTRL+C, then start again)
```

### Issue 7: Empty Response / 404 Errors

**Problem:** Endpoints return 404 Not Found

**Solution:**

```bash
# Check endpoint path exactly
curl http://localhost:8000/health     # Correct
curl http://localhost:8000/Health     # Wrong (case-sensitive)
curl http://localhost:8000/api/v1/info  # Correct

# View all routes in docs
open http://localhost:8000/docs
```

### Issue 8: .env Variables Not Loading

**Problem:** Environment variables show as None or default values

**Solution:**

```bash
# Verify .env file exists
ls -la .env

# Check file contents
cat .env

# Ensure python-dotenv is installed
pip install python-dotenv

# Load variables explicitly in Python:
from dotenv import load_dotenv
load_dotenv()
```

---

## Next Steps

Congratulations! You've successfully built a working FastAPI application. Here's what you've accomplished today:

**✓ Completed:**
- Set up a properly structured Python project
- Created a FastAPI application with 2 endpoints
- Configured environment-based settings
- Learned about async/await and ASGI
- Tested your API with multiple tools
- Understood the core concepts of FastAPI

**Tomorrow (Day 2), you'll add:**
- PostgreSQL database connection
- Database models with SQLAlchemy
- Alembic for database migrations
- Transaction model and CRUD operations
- Database connection pooling
- Async database queries

**Preparation for Day 2:**

1. **Install PostgreSQL** (if not already installed):
   ```bash
   # Ubuntu/Debian
   sudo apt-get install postgresql postgresql-contrib

   # macOS
   brew install postgresql

   # Or use Docker
   docker run -d \
     --name sentinel-postgres \
     -e POSTGRES_PASSWORD=sentinel123 \
     -e POSTGRES_USER=sentinel \
     -e POSTGRES_DB=sentinel_db \
     -p 5432:5432 \
     postgres:15-alpine
   ```

2. **Verify PostgreSQL is running:**
   ```bash
   # Check status
   sudo service postgresql status  # Linux
   brew services list              # macOS

   # Connect to verify
   psql -U postgres
   ```

3. **Optional: Install database GUI tool:**
   - pgAdmin: https://www.pgadmin.org/
   - DBeaver: https://dbeaver.io/
   - TablePlus: https://tableplus.com/

**Additional Resources:**

- FastAPI Documentation: https://fastapi.tiangolo.com/
- Pydantic Documentation: https://docs.pydantic.dev/
- Uvicorn Documentation: https://www.uvicorn.org/
- Python Asyncio: https://docs.python.org/3/library/asyncio.html
- RESTful API Design: https://restfulapi.net/

**Practice Exercises:**

1. Add a new endpoint `GET /api/v1/version` that returns only the version number
2. Create a `POST /api/v1/echo` endpoint that accepts JSON and returns it
3. Add query parameters to the info endpoint (e.g., `?format=simple`)
4. Implement a simple counter endpoint that increments on each call
5. Add request logging to see all incoming requests

**Code Quality Checklist:**

Before moving to Day 2, ensure:
- [ ] All endpoints return 200 status code
- [ ] No import errors or warnings
- [ ] Virtual environment is properly set up
- [ ] .env file is not committed to Git
- [ ] Test all endpoints with curl or browser (Swagger UI at /docs)

---

## Git Commit Checkpoint

Save your progress with Git:

```bash
# Initialize Git repository (if not done already)
git init

# Add all files except .env (protected by .gitignore)
git add .

# Verify .env is NOT staged (should not appear)
git status

# Create your first commit
git commit -m "Day 1: Initial FastAPI setup with health and info endpoints

- Set up project structure with app/ and tests/ directories
- Created FastAPI application with 2 endpoints (/health, /api/v1/info)
- Configured environment variables with .env support
- Added comprehensive .gitignore and requirements.txt
- Implemented async endpoints with proper documentation
- Added startup/shutdown event handlers"

# View commit history
git log --oneline
```

---

**Navigation:** [← Previous: Main Guide](README.md) | [Main Guide](README.md) | [Next: Day 2 →](README-DAY-002.md)

---

**Last Updated:** 2024-01-15
**Author:** Sentinel Team
**Version:** 1.0.0
**Estimated Completion Time:** 2-3 hours

---

## Quick Reference Card

### Essential Commands

```bash
# Activate virtual environment
source venv/bin/activate              # Linux/macOS
venv\Scripts\activate                 # Windows

# Start development server
uvicorn app.main:app --reload

# Install dependencies
pip install -r requirements.txt

# Test health endpoint
curl http://localhost:8000/health

# View API docs
open http://localhost:8000/docs

# Check Python imports
python -c "from app import main"

# Deactivate virtual environment
deactivate
```

### Project File Locations

```
sentinel/
├── app/main.py          # Main FastAPI application
├── app/__init__.py      # Package initialization
├── .env                 # Environment variables (local)
├── .env.example         # Environment template
├── requirements.txt     # Python dependencies
└── .gitignore          # Git ignore rules
```

### Important URLs (when server is running)

- Application: http://localhost:8000
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc
- OpenAPI Schema: http://localhost:8000/openapi.json
- Health Check: http://localhost:8000/health
- Service Info: http://localhost:8000/api/v1/info

---

*End of Day 1 Guide*
