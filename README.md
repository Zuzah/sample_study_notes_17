# sample_study_notes_17
KYC/AML Studies

### Logging fix

logging.py:

```python
import sys
from pathlib import Path
from loguru import logger
from app.core.config import settings


def configure_logging():
    """
    Initializes a production-grade multi-target logging structure.
    Sinks informational tracks to Console (stdout) and system anomalies to an error log file.
    """
    # 1. Clear any default standard library logger configurations
    logger.remove()

    # 2. Add standard Console stdout handler for operational info tracking
    logger.add(
        sys.stdout,
        level="INFO",
        format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level:7}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>",
        colorize=True,
    )

    # 3. AUDIT_ROOT set (real K8s PVC mount): logs go to AUDIT_ROOT/Logs, matching
    # the legacy .NET convention and the same branching reporting_orchestrator.py's
    # _resolve_output_directory() uses for Archive/Failed. Unset (local dev):
    # unchanged, BASE_DIR/logs.
    log_dir = (
        Path(settings.AUDIT_ROOT) / "Logs"
        if settings.AUDIT_ROOT
        else settings.BASE_DIR / "logs"
    )
    log_dir.mkdir(parents=True, exist_ok=True)
    error_log_path = log_dir / "error.log"

    # 4. Bind file-system sink tracking exceptions and errors exclusively
    logger.add(
        str(error_log_path),
        level="WARNING",
        format="{time:YYYY-MM-DD HH:mm:ss} | {level:7} | {name}:{function}:{line} - {message}",
        rotation="10 MB",  # Prevents files from growing infinitely
        retention="30 days",  # Automatically purges stale server records
        compression="zip",  # Compresses historic tracking sets to optimize disk space
    )

    return logger


# Globally instantiate configured multi-sink execution logger
log = configure_logging()

```

new tests/core/test_logging.py
```python
from app.core.config import settings
from app.core.logging import configure_logging


def test_configure_logging_writes_under_audit_root_logs_when_set(monkeypatch, tmp_path):
    """AUDIT_ROOT set (real K8s PVC mount) - logs go to AUDIT_ROOT/Logs, same
    Archive/Failed-style branching as reporting_orchestrator.py's
    _resolve_output_directory(). Matches the legacy .NET convention
    (ExtractSettings__Logs=/app/Audit/Logs)."""
    audit_root = tmp_path / "Audit"
    monkeypatch.setattr(settings, "AUDIT_ROOT", str(audit_root))

    try:
        log = configure_logging()
        log.error("test message")

        assert (audit_root / "Logs" / "error.log").exists()
    finally:
        configure_logging()


def test_configure_logging_falls_back_to_base_dir_logs_when_audit_root_unset(monkeypatch):
    """Unset (local dev) - unchanged existing behavior, BASE_DIR/logs."""
    monkeypatch.setattr(settings, "AUDIT_ROOT", None)

    try:
        log = configure_logging()
        log.error("test message")

        assert (settings.BASE_DIR / "logs" / "error.log").exists()
    finally:
        configure_logging()
```

## Documentation

```
16.1.1.3 Middleware Description

Service ID	Service Name	Functionality
MW-01	Apache Airflow	Triggers/schedules this platform's run-report command; owns scheduling, monitoring, and retries for pipeline runs (per confirmed Airflow/platform boundary — Airflow handles orchestration only, all Fenergo-specific logic lives in the platform itself)
MW-02	SFTP Delivery (paramiko)	Transports the finished report file and its SHA256 checksum marker from this platform to downstream landing zones over SSH/SFTP
MW-03	Fenergo OAuth2/REST API Client	Authenticates (OAuth2 client_credentials) and exchanges data with Fenergo's Advanced Reporting API — submits the SQL/saved query, polls status, retrieves the presigned download URL
```

```
16.1.1.4 Backend/Core Banking Description

Program Name	Type	Functionality
Fenergo (FenX Advanced Reporting API)	External SaaS vendor platform — KYC/client onboarding system of record	Source system. Executes the submitted SQL/saved query against its reporting datastore and returns the result set as a downloadable CSV
RegCentral	TBD — confirm with the endpoint owner	Downstream recipient of the ChinaGTTReport output file and checksum marker
ClientCentralData	TBD — confirm with the endpoint owner	Downstream recipient of the CANDERReport/ProductReport/UKProductReport/SingaporeReport output files and checksum markers

```
