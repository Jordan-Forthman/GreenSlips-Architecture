# Predictive Engine: Dependencies

The GreenSlips predictive engine **runs in-process in .NET**, not as a separate Python service. There is no inference sidecar, no FastAPI/gRPC bridge, and no cross-language call at request time — the engine is resolved through a `ISportsModelEngine` seam inside the same ASP.NET Core host that serves the API, and its scheduled fits run as Hangfire jobs in the worker role.

This file replaces the `requirements.txt` that previously lived here. That file pinned the Python stack (PyTorch / XGBoost / Optuna / SHAP) used by the earlier NBA capstone pipeline, which has been retired along with the rest of the NBA vertical; it no longer describes anything in the current product.

## Runtime

| | |
|---|---|
| Target framework | `net10.0` |
| Host | ASP.NET Core 10 |

## Numerical & machine learning

| Package | Version | Role |
|---|---|---|
| `MathNet.Numerics` | 5.0.0 | Linear algebra, distributions, and the numerical primitives behind the simulation, copula, and calibration code |
| `Microsoft.ML` | 5.0.0 | ML.NET pipeline and model persistence for the trained classifier components |
| `Microsoft.ML.LightGbm` | 5.0.0 | Gradient-boosted tree learner |

Where a model genuinely requires a library with no .NET equivalent, the architectural decision of record is to **train offline and export to ONNX** for in-process inference, rather than introduce a second runtime into the request path.

## Data access & scheduling

| Package | Version | Role |
|---|---|---|
| `Npgsql.EntityFrameworkCore.PostgreSQL` | 10.0.0 | PostgreSQL provider for the event store and derived-feature tables |
| `Hangfire.AspNetCore` | 1.8.21 | Job scheduling for the nightly ingest → rebuild → fit → inference chain |
| `Hangfire.PostgreSql` | 1.20.12 | PostgreSQL-backed job storage |
| `Microsoft.Extensions.Caching.StackExchangeRedis` | 10.0.5 | Distributed cache in front of read-heavy projections |

## Ingestion & reporting

| Package | Version | Role |
|---|---|---|
| `CsvHelper` | 33.1.0 | Streaming parser for the off-peak sabermetric CSV feeds |
| `ClosedXML` | 0.105.0 | Workbook export of scored inference runs for offline research |
| `Microsoft.Extensions.Http.Resilience` | 9.6.0 | Retry, circuit-breaker, and timeout policies on every vendor client |

*Versions are those pinned in the private application repository at the time of writing. Package names and versions are public information; no proprietary configuration is included here.*
