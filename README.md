# AIWAF Project

AIWAF is a multi-package web application firewall ecosystem for Python web apps, with optional Rust acceleration.

It provides a shared protection model across:

- `aiwaf` for Django
- `aiwaf_flask` for Flask
- `aiwaf-rust` (`aiwaf_rust` import) for performance-focused heuristics used by Python packages

## Table of Contents

1. Overview
2. Core Capabilities
3. Package Map
4. Architecture
5. Request Processing Flow
6. Configuration Model
7. Data and Storage
8. AI Training and Detection
9. Middleware and Runtime Controls
10. CLI and Management Commands
11. Local Development
12. Testing
13. Packaging and Release
14. Operational Guidance
15. Troubleshooting
16. Contributing
17. License

## 1. Overview

AIWAF combines rule-based controls and AI-assisted anomaly detection to block abusive traffic while preserving operability through exemptions, graceful fallbacks, and optional acceleration.

Design principles:

- Baseline protection should work without optional dependencies.
- Optional AI and Rust features should improve detection/performance without making runtime brittle.
- Exemptions are first-class to reduce false positives.
- Middleware ordering and observability are treated as operationally important.

## 2. Core Capabilities

- IP blacklist and allow/exemption handling
- Static and dynamic suspicious keyword checks
- Rate limiting and flood detection
- Header validation (including suspicious user-agent checks)
- Honeypot timing and request-pattern checks
- UUID tamper/probing protection
- Optional GeoIP country block/allow controls
- AI-assisted anomaly scoring (IsolationForest)
- Optional Rust-assisted header/feature/behavior analysis
- Runtime logging and offline retraining workflows

## 3. Package Map

### Django package: `aiwaf`

- Runtime: Django middleware stack
- Persistence: Django ORM models
- Operations: Django management commands
- Settings model: flat `AIWAF_*` plus compatibility mapping from legacy nested settings

### Flask package: `aiwaf_flask`

- Runtime: app registration with middleware modules
- Persistence: CSV, SQLAlchemy database, or memory fallback
- Operations: CLI commands (`aiwaf`, `aiwaf-console`, `aiwaf-whois`)
- Settings model: Flask `app.config` with `AIWAF_*` keys

### Rust package: `aiwaf-rust`

- Built with PyO3 + maturin
- Python import module: `aiwaf_rust`
- Exposes fast heuristics used by Python packages when enabled

## 4. Architecture

### Django runtime architecture

Primary runtime checks live in `aiwaf/middleware.py`, including:

- `JsonExceptionMiddleware`
- `GeoBlockMiddleware`
- `IPAndKeywordBlockMiddleware`
- `HeaderValidationMiddleware`
- `RateLimitMiddleware`
- `AIAnomalyMiddleware`
- `HoneypotTimingMiddleware`
- `UUIDTamperMiddleware`

Suspicious requests are blocked with `PermissionDenied("blocked")`, with JSON conversion support for API-style requests.

### Flask runtime architecture

Flask supports:

- `AIWAF(app, ...)` initialization
- `register_aiwaf_middlewares(app, ...)` registration
- Compatibility alias: `register_aiwaf_protection`

Middleware modules include:

- `ip_keyword_block`
- `rate_limit`
- `honeypot`
- `header_validation`
- `geo_block`
- `ai_anomaly`
- `uuid_tamper`
- `logging`

Route-level controls are available via decorators in `exemption_decorators.py`.

### Rust architecture

Core exported Python-callable functions in `src/lib.rs`:

- `validate_headers(...)`
- `validate_headers_with_config(...)`
- `extract_features(...)`
- `analyze_recent_behavior(...)`

This module accelerates hot heuristics but is optional; Python fallbacks remain available.

## 5. Request Processing Flow

1. Request enters framework middleware chain.
2. Exemption checks evaluate bypass controls (IP/path/route/view and keyword exemptions as configured).
3. Rule-based checks run (headers, blacklist/keywords, rate, honeypot, UUID, geo as enabled).
4. AI anomaly checks run if model and dependencies are available/enabled.
5. Suspicious traffic is denied with a blocked response.
6. Logging captures request/response metadata for analysis.
7. Offline/periodic training updates model and dynamic keyword state.

## 6. Configuration Model

AIWAF uses `AIWAF_*` settings consistently across Django and Flask, with framework-specific loading behavior.

High-impact configuration areas:

### Abuse controls

- `AIWAF_RATE_WINDOW`
- `AIWAF_RATE_MAX`
- `AIWAF_RATE_FLOOD`
- `AIWAF_MIN_FORM_TIME`

### Exemptions and filtering

- `AIWAF_EXEMPT_PATHS`
- `AIWAF_EXEMPT_IPS`
- `AIWAF_EXEMPT_KEYWORDS`
- `AIWAF_ALLOWED_PATH_KEYWORDS`

### Logging and training

- `AIWAF_ACCESS_LOG`
- `AIWAF_MIDDLEWARE_LOGGING` / logging enablement keys
- `AIWAF_MIN_AI_LOGS`
- `AIWAF_MIN_TRAIN_LOGS`
- `AIWAF_FORCE_AI_TRAINING`

### Model and backend

- `AIWAF_MODEL_PATH`
- `AIWAF_MODEL_STORAGE` (Django)
- `AIWAF_MODEL_STORAGE_FALLBACK` (Django)
- `AIWAF_USE_RUST`

### Geo controls

- `AIWAF_GEO_BLOCK_ENABLED`
- `AIWAF_GEO_BLOCK_COUNTRIES`
- `AIWAF_GEO_ALLOW_COUNTRIES`
- `AIWAF_GEOIP_DB_PATH`

Notes:

- Django supports legacy nested config via compatibility mapping in `aiwaf/settings_compat.py`.
- Flask storage selection is controlled by config (`AIWAF_USE_CSV`, DB readiness, and related data/log directory settings).

## 7. Data and Storage

### Django persistence (ORM)

Primary models in `aiwaf/models.py` include:

- `BlacklistEntry`
- `IPExemption`
- `ExemptPath`
- `DynamicKeyword`
- `FeatureSample`
- `RequestLog`
- `AIModelArtifact`
- `GeoBlockedCountry`

Model artifact storage backends in Django:

- `file`
- `db`
- `cache`

### Flask persistence (storage abstraction)

`aiwaf_flask/storage.py` supports:

- `csv` mode (default when CSV is enabled)
- `database` mode (SQLAlchemy configured, CSV disabled)
- `memory` fallback mode

Persisted items include block/allow lists, dynamic keywords, geo blocks, and path exemptions.

## 8. AI Training and Detection

Training and inference logic is implemented per framework (`aiwaf/trainer.py`, `aiwaf_flask/trainer.py`) and generally follows:

- Parse access logs (including rotated and `.gz` logs where supported) or fallback logging sources
- Extract request behavior features
- Train/update an IsolationForest model when sample thresholds are met
- Learn/refine dynamic suspicious keywords
- Apply safeguards to reduce false positives (exempt/allowed keywords, route-aware behavior in Flask)

If AI dependencies are unavailable, rule-based protection continues to work and AI behavior degrades gracefully.

## 9. Middleware and Runtime Controls

Operational controls available across packages:

- Middleware enable/disable selection
- Exemptions at path/IP/route/view level
- Geo block/allow control
- Rate and timing thresholds
- Header scoring/validation strictness
- Optional extended blocked-request context capture (Flask)

Important: middleware order materially changes behavior and false-positive profile.

## 10. CLI and Management Commands

### Django management commands

Representative command groups under `aiwaf/management/commands/`:

- Exemptions and path controls: `add_exemption`, `add_ipexemption`, `add_pathexemption`, `aiwaf_pathshell`
- Diagnostics and inspection: `aiwaf_list`, `aiwaf_logging`, `aiwaf_diagnose`, `aiwaf_whois`, `diagnose_blocking`, `debug_csv`, `geo_traffic_summary`
- Maintenance: `aiwaf_reset`, `clear_blacklist`, `clear_cache`, `setup_models`
- Training/model: `detect_and_train`, `regenerate_model`
- Geo control: `geo_block_country`

### Flask CLI

Defined via entrypoints and implemented in `aiwaf_flask/cli.py`:

- `aiwaf`
- `aiwaf-console`
- `aiwaf-whois`

Used for list management, state inspection/export, and operational maintenance.

## 11. Local Development

### Prerequisites

- Python 3.8+
- pip/venv tooling
- Rust toolchain (`cargo`, `rustc`) when working on or using `aiwaf-rust`

### Django workflow

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -U pip
pip install -e .
```

### Flask workflow

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -U pip
pip install -e .[dev]
```

### Rust workflow

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -U pip maturin
maturin develop
```

## 12. Testing

### Django tests

```bash
python run_tests.py
# or
python manage.py test
# optional curated subset
python tests/run_working_tests.py
```

### Flask tests

```bash
pytest
# optional targeted run
pytest tests/test_middleware.py
```

### Rust tests

```bash
cargo test
```

## 13. Packaging and Release

### Rust packaging (`aiwaf-rust`)

```bash
maturin build --release --out dist
maturin sdist --out dist
pip install dist/*.whl
```

Release flow:

1. Bump version metadata (`Cargo.toml`, `pyproject.toml` for Rust package).
2. Validate locally (`cargo test`, `maturin build`).
3. Create a tagged GitHub release.
4. Verify CI publishes wheels/sdist to PyPI.

## 14. Operational Guidance

- Keep exemption controls explicit and documented for each deployment.
- Validate middleware order whenever adding/removing protections.
- Ensure logs used for training are representative and accessible.
- Treat AI anomaly decisions as one signal in a broader defense strategy.
- Monitor false-positive trends and adjust thresholds/exemptions safely.
- Keep optional dependency failures non-fatal in production.

## 15. Troubleshooting

- AI model not loading: verify model path/storage configuration; runtime should continue with rule-based protection.
- Rust module import errors: run `maturin develop` in the active environment and verify interpreter alignment (`which python`, `python --version`).
- Missing Rust toolchain: install/validate `rustc` and `cargo`.
- Excessive false positives: review exemption settings, middleware order, keyword lists, and training data quality.
- Geo blocking not effective: verify MMDB path, geo settings, and middleware registration.

## 16. Contributing

General expectations:

- Keep behavior backward compatible unless a breaking change is intentional and documented.
- Add tests for allow/block/edge/error behavior when changing runtime rules.
- Preserve graceful fallback behavior for optional dependencies (AI libs, Rust backend, GeoIP availability).
- Reuse storage/model abstractions instead of ad-hoc persistence logic.
- Update user-facing docs for config, command, and behavior changes.

Common change workflows:

### Add middleware rule

1. Implement logic in middleware module.
2. Respect exemption behavior.
3. Add focused tests.
4. Document ordering and settings impact.

### Extend training feature

1. Update feature extraction/training pipeline.
2. Maintain fallback behavior.
3. Validate serialization/model loading paths.
4. Add regression tests for low-data and threshold boundaries.

### Add operational command

1. Implement command in framework command/CLI module.
2. Keep idempotency where possible.
3. Add tests for success and failure paths.

## 17. License

MIT
