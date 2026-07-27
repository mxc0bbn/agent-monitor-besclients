# Third-Party Licenses

Agent Monitor for BES Clients bundles and depends on the open-source components
listed below. Each remains under its own license; the terms in the project
[LICENSE](LICENSE) apply only to Agent Monitor's own code and assets.

The web interface is built with plain HTML, CSS, and JavaScript and includes no
third-party front-end framework.

## Python runtime dependencies (dashboard)

| Component | Purpose | License |
|---|---|---|
| FastAPI | Web framework | MIT |
| Starlette | ASGI toolkit (FastAPI core) | BSD-3-Clause |
| Uvicorn | ASGI application server | BSD-3-Clause |
| python-multipart | Multipart form parsing | Apache-2.0 |
| SQLAlchemy | Database toolkit / ORM | MIT |
| psycopg2 | PostgreSQL driver | LGPL-3.0-or-later (with OpenSSL exception) |
| PyJWT | JSON Web Token handling | MIT |
| bcrypt | Password hashing | Apache-2.0 |
| passlib | Password hashing framework | BSD-2-Clause |
| cryptography | Cryptographic primitives (incl. FIPS 204 ML-DSA verification) | Apache-2.0 OR BSD-3-Clause |
| httpx | HTTP client | BSD-3-Clause |
| requests | HTTP client (BigFix REST) | Apache-2.0 |
| APScheduler | Background job scheduling | MIT |
| tzlocal | Local timezone detection | MIT |
| pydantic | Data validation | MIT |
| pydantic-settings | Settings management | MIT |
| python-dotenv | Environment file loading | BSD-3-Clause |
| psutil | System / process metrics | BSD-3-Clause |
| reportlab | PDF export | BSD-3-Clause |
| python-dateutil | Date/time utilities | Apache-2.0 AND BSD-3-Clause |
| pytz | Timezone database | MIT |

## Platform components (installed by the installer, not bundled)

| Component | Purpose | License |
|---|---|---|
| PostgreSQL | Database server | PostgreSQL License (permissive, BSD-style) |
| Nginx | Reverse proxy / TLS termination | BSD-2-Clause |

Full license texts for each component are available from the respective project
distributions and package repositories.
