# CapConduit Call Platform

## 1. Overview

CapConduit is a specialized, multi-tenant backend platform designed to manage and route phone calls from "Call Sellers" (lead generators) to appropriate "Call Centers" (Clients). It allows fine-grained control over call distribution based on campaigns, DIDs (phone numbers), concurrency limits, and total call caps per client link.

This is a backend built with Flask and SQLAlchemy, leveraging PostgreSQL for data storage. It is specifically designed for integration with Asterisk Realtime Architecture (ARA) for dynamic PJSIP endpoint configuration, although the Asterisk configuration itself is the *next* phase of development.

**Key Features (Backend API):**

*   **User Roles:** Admin, Staff (Admin subset), User (Call Seller).
*   **Client Management (Admin/Staff):** Manage Call Center clients and their corresponding PJSIP configurations intended for Asterisk ARA.
*   **Seller Management:**
    *   **DID Management:** Add, update, list, delete owned DIDs with format validation.
    *   **Campaign Management:** Create campaigns with routing strategies (Priority, Round-Robin, Weighted) and timeouts.
    *   **Linking:** Associate DIDs with Campaigns, and Campaigns with Clients.
    *   **Granular Control:** Define specific settings (concurrency cap, total call cap, priority, weight) for each Campaign-Client link.
*   **Call Logging:** Internal API endpoint (`/api/internal/log_call`) for Asterisk (AGI) to log detailed call records (CDRs) and automatically increment total call counters on relevant Campaign-Client settings.
*   **Seller Reporting:** API for Sellers to view their call logs with filtering options.
*   **Authentication:** Secure login/logout using Flask-Login and session management.
*   **Authorization:** Role-based access control enforced via decorators.
*   **Standardized Backend:** Features consistent transaction management (commits in routes), custom exception handling, and a clear service layer architecture.

## 2. Technical Architecture Summary

*   **Backend Framework:** Python 3.11+ / Flask / SQLAlchemy
*   **Database:** PostgreSQL (Stores all configuration, user data, logs; Provides ARA tables)
*   **API:** RESTful API using Flask Blueprints and Marshmallow for validation/serialization.
*   **Migrations:** Flask-Migrate (Alembic)
*   **Testing:** Pytest with Flask-Testing utilities.
*   **WSGI Server (Production):** Gunicorn (or similar)
*   **Reverse Proxy (Production):** Nginx (recommended)
*   **Target Telephony Engine:** Asterisk 20+ (To be configured for ARA)

## 3. Project Structure

```
capconduit/
├── app/                  # Main Flask application package
│   ├── __init__.py       # App factory (create_app)
│   ├── api/              # API Blueprints, routes, schemas
│   │   ├── routes/       # API route definitions (auth, admin, seller, internal)
│   │   └── schemas/      # Marshmallow schemas
│   ├── database/         # SQLAlchemy models
│   │   └── models/
│   ├── services/         # Business logic layer
│   ├── utils/            # Utility functions, decorators, exceptions
│   ├── config.py         # Configuration classes
│   └── extensions.py     # Flask extension instances
├── migrations/           # Alembic database migrations
├── tests/                # Pytest integration tests
│   ├── conftest.py       # Pytest fixtures
│   └── integration/api/
├── .env                  # Environment variables (DB URI, secrets) - DO NOT COMMIT SECRETS
├── .flaskenv             # Flask CLI environment variables
├── .gitignore            # Git ignore rules
├── .python-version       # Target Python version (for pyenv)
├── README.md             # This file
├── dbcleanup.sh          # Utility script to clean PostgreSQL DB (USE WITH CAUTION)
├── requirements.txt      # Python package dependencies
├── sample_data.sql       # SQL script to populate DB with sample data
├── run.py                # Basic script to run Flask development server
├── wsgi.py               # WSGI entry point for production servers
├── handoff.md            # Project context document
├── masterplan.md         # Project goals and design document
└── progress.md           # Project progress tracking document
```

## 4. Setup and Installation

Follow these steps to set up the development environment on a Debian/Ubuntu-based system (like the target GCP VM).

**4.1. Prerequisites:**

*   Git
*   Python 3.11+ (use of `pyenv` is highly recommended: [https://github.com/pyenv/pyenv](https://github.com/pyenv/pyenv))
*   PostgreSQL Server (v12+) installed and running.
*   `psql` command-line tool installed and in PATH.
*   Basic build tools (usually covered by `build-essential` on Debian/Ubuntu).

**4.2. Clone Repository:**

```bash
git clone https://github.com/sgrwlc/capconduit.git
cd capconduit
```

**4.3. Set up Python Environment:**

```bash
# Optional: Install and set Python version using pyenv
# pyenv install 3.11+
# pyenv local 3.11+ # Creates .python-version if not present

# Create virtual environment
python -m venv venv

# Activate virtual environment
source venv/bin/activate
# Windows: venv\Scripts\activate
```

**4.4. Install Dependencies:**

```bash
pip install -r requirements.txt
```

**4.5. Setup PostgreSQL Database & User:**

*   Connect to PostgreSQL as a superuser (e.g., `postgres`).
*   Create the database user and the two required databases (main and test).
    ```sql
    -- Replace 'YourSecurePassword' - use a strong password!
    CREATE USER capconduit_user WITH PASSWORD 'YourSecurePassword';

    -- Create main database
    CREATE DATABASE capconduit_db OWNER capconduit_user;
    GRANT ALL PRIVILEGES ON DATABASE capconduit_db TO capconduit_user;

    -- Create test database
    CREATE DATABASE capconduit_test_db OWNER capconduit_user;
    GRANT ALL PRIVILEGES ON DATABASE capconduit_test_db TO capconduit_user;
    ```

**4.6. Configure Environment Variables (`.env`):**

*   Create a `.env` file in the project root (`capconduit/`) or you can use the structure the included in the repository.
*   **Critically, edit `.env` and set:**
    *   `SECRET_KEY`: Generate a strong key (`python -c 'import secrets; print(secrets.token_hex(32))'`).
    *   `DATABASE_URI`: Connection string for the main DB (e.g., `postgresql://capconduit_user:YourEncodedPassword@localhost:5432/capconduit_db`).
    *   `TEST_DATABASE_URI`: Connection string for the test DB (e.g., `postgresql://capconduit_user:YourEncodedPassword@localhost:5432/capconduit_test_db`).
    *   `INTERNAL_API_TOKEN`: Generate a secure random token.
    *   `FLASK_ENV`: Set to `development` for local setup.
*   **Important:** URL-encode your password if it contains special characters (e.g., `@` becomes `%40`, `/` becomes `%2F`). Add `.env` to your global or project `.gitignore`.

**4.7. Apply Database Migrations:**

*   Make sure your virtual environment is active.
*   Run the `flask db upgrade` command:
    ```bash
    mkdir -p migrations/versions
    ```
    ```bash
    flask db migrate -m "Initial migration"
    ```
    ```bash
    flask db upgrade
    ```
    This creates the tables in the **main** database (`DATABASE_URI`).

**4.8. Load Sample Data (Optional):**

*   This populates the **main** database with sample data.
* Password for all users is 'password'.
    ```bash
    # Ensure psql can connect using the DATABASE_URI
    psql "$(grep '^DATABASE_URI=' .env | cut -d '=' -f2-)" -f sample_data.sql
    ```

## 5. Running the Application

**5.1. Development Server:**

*   Activate the virtual environment (`source venv/bin/activate`).
*   Ensure `FLASK_ENV=development` is set (e.g., in `.flaskenv` or `.env`).
*   Run:
    ```bash
    flask run --host=0.0.0.0 --port=5000
    ```
*   The API will be available at `http://<your-external-ip>:5000` or `http://127.0.0.1:5000`.

**5.2. Production (Gunicorn Example):**

*   Ensure `FLASK_ENV=production` is set.
*   Run:
    ```bash
    gunicorn --bind 0.0.0.0:5000 wsgi:application
    ```
    *(Typically run behind Nginx)*

## 6. Running Tests

1.  Ensure the **test database** exists and the user has privileges (created in step 4.5).
2.  Ensure `TEST_DATABASE_URI` is correctly set in `.env`.
3.  Activate the virtual environment.
4.  Run from the project root (`capconduit/`):
    ```bash
    pytest
    ```
    Or for more detail:
    ```bash
    pytest -v -s
    ```
    The test suite will automatically create/drop tables and load sample data into the **test** database for each session.

## 7. Key API Endpoints Summary

*(See individual route files in `app/api/routes/` for request/response details)*

*   `/api/auth/`: Login, Logout, Status
*   `/api/admin/users/`: User Management (Admin only)
*   `/api/admin/clients/`: Client & PJSIP Management (Admin only)
*   `/api/seller/dids/`: Seller DID Management
*   `/api/seller/campaigns/`: Seller Campaign & Link Management
*   `/api/seller/logs/`: Seller Call Log Viewing
*   `/api/internal/log_call`: Internal Call Logging (Requires Token)
*   `/api/internal/route_info`: Internal Routing Info (Requires Token)