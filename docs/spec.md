# Commute Optimizer — Collector Technical Specification

**Version:** 0.1.0  
**Status:** Draft  
**Component:** Collector  
**Stack:** Java · Spring Boot

---

## Table of Contents

1. [Overview](#1-overview)
2. [Goals](#2-goals)
3. [Architecture](#3-architecture)
4. [Milestones](#4-milestones)
5. [Module Responsibilities](#5-module-responsibilities)
6. [Directory Structure](#6-directory-structure)
7. [CSV Schema](#7-csv-schema)
8. [Telemetry Schema](#8-telemetry-schema)
9. [Scheduling](#9-scheduling)
10. [Retry Strategy](#10-retry-strategy)
11. [Backup Strategy](#11-backup-strategy)
12. [Configuration](#12-configuration)
13. [Error Handling](#13-error-handling)
14. [Future Improvements](#14-future-improvements)
15. [Assumptions](#15-assumptions)
16. [Known Limitations](#16-known-limitations)

---

## 1. Overview

This document is the technical specification for the **Collector** component of Commute Optimizer. The Collector is a standalone backend service responsible for periodically querying ride pricing and weather APIs, and persisting the results as structured CSV files. Those files will later serve as the training dataset for a ride-pricing prediction model.

This specification covers the Collector exclusively. The Android application, ML model, route optimization algorithm, and prediction engine are out of scope, except where a direct interface with the Collector must be described.

---

## 2. Goals

### Primary Goal

Collect high-quality ride price estimate data over multiple months with high reliability, while respecting external API rate limits.

### Design Philosophy

The following principles govern every architectural decision in this component:

- **Reliability over complexity.** A simple design that runs for months unattended is worth more than a sophisticated one that requires frequent intervention.
- **Data quality over quantity.** A smaller dataset with clean, consistent records is more valuable than a large one with gaps and anomalies.
- **Minimal manual intervention.** Once deployed, the Collector should run without human input.
- **Maintainability first.** Any contributor — including a future version of the original author — should be able to understand, modify, and extend the codebase quickly.
- **No premature optimization.** Solve the problem at hand. Optimize only when a bottleneck is proven.
- **CSV as canonical store.** CSV files are simple, portable, and sufficient at the current data scale. No database is introduced unless a clear need arises.
- **Provider-agnostic where practical.** Abstractions should allow additional ride providers to be integrated with minimal rework.

### Non-Functional Requirements

| Attribute | Target |
|---|---|
| Maintainability | High |
| Modularity | High |
| Reliability | High |
| Extensibility | High |
| Performance | Moderate |
| Cost | Near zero |

---

## 3. Architecture

The Collector is a Spring Boot application organized as a linear pipeline. On each scheduled execution, the pipeline runs the following stages in order:

```
Coordinate Generator
        │
        ▼
   Uber API Client
        │
        ▼
 Weather API Client
        │
        ▼
  Response Parser
        │
        ▼
   Dataset Writer  ──► data/YYYY-MM-DD.csv
        │
        ▼
 Telemetry Writer  ──► telemetry/YYYY-MM-DD.csv
        │
        ▼
  Backup Manager   ──► GitHub
```

Each stage is implemented as a separate module with a single, well-defined responsibility. Modules communicate through plain Java objects; there is no message broker or shared mutable state between pipeline stages.

---

## 4. Milestones

### Milestone 0 — API Exploration

**Objective:** Remove all uncertainty around the external APIs before writing production code.

**Success criteria:**

- Authentication succeeds for both the Uber Pricing API and the Weather API.
- Valid requests are issued and verified from Postman.
- Every required request parameter is identified and understood.
- Every important response field is identified and understood.
- Common error conditions (auth failure, rate limit, invalid coordinates) have been triggered and observed.
- Both APIs can be explained in full without consulting the official documentation.

---

### Milestone 1 — Collector Engine

**Objective:** Build a fully functional, locally executable collector with no automation.

**Pipeline steps:**

1. Generate a coordinate pair.
2. Call the Uber Pricing API.
3. Call the Weather API.
4. Parse both responses.
5. Append a row to the dataset CSV.
6. Append a row to the telemetry CSV.

**Success criteria:**

- Runs locally from the command line.
- No scheduler is implemented.
- No deployment is required.
- A single execution successfully appends one valid row to each CSV.

---

### Milestone 2 — Automation and Reliability

**Objective:** Convert the one-shot collector into a reliable, autonomous service.

**Features introduced:**

- Scheduled execution at a configurable interval.
- Configurable retry logic on API failure.
- Automatic daily CSV file rotation.
- Telemetry capture on every execution.
- Automated daily backup to GitHub.

---

### Milestone 3 — Deployment and Stability

**Objective:** Deploy the service and prove it runs reliably in production.

**Requirements:**

- Deployed to [Render](https://render.com).
- Runs continuously for at least one week without intervention.
- Only bug fixes are permitted during the stability window; no new features.
- Manual intervention during the stability window constitutes a failure that requires investigation.

---

### Milestone 4 — Dataset Review

**Objective:** Evaluate the collected dataset before any ML work begins.

**Review checklist:**

- Missing values — identify columns with nulls or blanks and their frequency.
- Distribution — verify that prices, times of day, and weather conditions are reasonably distributed.
- Coverage — confirm sufficient data across weekdays, weekends, and different hours.
- Outliers — flag anomalous rows for investigation.
- Data quality — spot-check a random sample of raw rows.
- Row count — confirm the dataset is large enough to be useful for model training.
- Readiness — produce a clear go / no-go assessment for the ML phase.

---

## 5. Module Responsibilities

### Coordinate Generator

Generates origin and destination coordinate pairs for each collection run. The initial implementation may draw from a fixed pool of known valid coordinates. Future versions may sample coordinates more dynamically.

### Uber Client

Encapsulates all communication with the Uber Pricing API: constructing requests, attaching authentication, sending the request, and returning the raw response. This module is aware of the Uber API contract and nothing else.

### Weather Client

Fetches current weather conditions for the pickup location at the time of collection. Returns raw weather data to the Parser.

### Parser

Receives raw responses from the Uber Client and Weather Client and converts them into normalized, typed Java objects. All field extraction and type coercion happens here. Downstream modules never interact with raw API responses.

### Dataset Writer

Receives a normalized data record from the Parser and appends it as a new row to the current day's dataset CSV. Creates a new CSV file with headers if none exists for the current date.

### Telemetry Writer

Records collector health metadata for every pipeline execution, regardless of whether the data collection succeeded or failed. Appends to the current day's telemetry CSV.

### Scheduler

Triggers the full pipeline at a configured interval using Spring's scheduling support. The interval is externalized in `application.yml` and requires no code change to modify.

### Backup Manager

At the end of each day (or at a configured frequency), commits the current dataset and telemetry CSVs to a designated GitHub repository. Provides versioned, off-host backup at no cost.

---

## 6. Directory Structure

```
commute-optimizer-collector/
├── src/                        # Spring Boot application source
│   └── main/
│       └── java/
│           └── com/example/collector/
│               ├── coordinator/    # Coordinate Generator
│               ├── uber/           # Uber API Client
│               ├── weather/        # Weather API Client
│               ├── parser/         # Response Parser
│               ├── writer/         # Dataset Writer + Telemetry Writer
│               ├── scheduler/      # Scheduler
│               └── backup/         # Backup Manager
├── config/
│   └── application.yml         # All externalized configuration
├── data/
│   └── YYYY-MM-DD.csv          # Daily dataset files
├── telemetry/
│   └── YYYY-MM-DD.csv          # Daily telemetry files
├── locations/
│   └── coordinates.json        # Coordinate pool
└── docs/
    └── collector-spec.md       # This document
```

---

## 7. CSV Schema

Each successful pipeline execution appends one row per ride product returned by the Uber API. Files are named `YYYY-MM-DD.csv` and stored in `data/`.

| Field | Type | Description |
|---|---|---|
| `query_id` | UUID | Unique identifier for this collection run |
| `timestamp` | ISO 8601 | UTC datetime of the collection |
| `pickup_latitude` | Decimal | Origin latitude |
| `pickup_longitude` | Decimal | Origin longitude |
| `drop_latitude` | Decimal | Destination latitude |
| `drop_longitude` | Decimal | Destination longitude |
| `weather` | String | Weather condition (e.g., `Clear`, `Rain`) |
| `temperature` | Decimal | Temperature in degrees Celsius |
| `hour` | Integer | Hour of day (0–23) |
| `weekday` | Integer | Day of week (0 = Monday, 6 = Sunday) |
| `weekend` | Boolean | `true` if Saturday or Sunday |
| `product_id` | String | Uber internal product identifier |
| `display_name` | String | Product name (e.g., `UberX`) |
| `localized_display_name` | String | Localized product name |
| `currency_code` | String | ISO 4217 currency code |
| `minimum` | Decimal | Minimum fare |
| `low_estimate` | Decimal | Low end of the price estimate |
| `high_estimate` | Decimal | High end of the price estimate |
| `estimate` | String | Human-readable price estimate string |
| `surge_multiplier` | Decimal | Surge multiplier (1.0 = no surge) |
| `duration_seconds` | Integer | Estimated trip duration in seconds |
| `distance_miles` | Decimal | Estimated trip distance in miles |

---

## 8. Telemetry Schema

Telemetry is written for every pipeline execution, including failed ones. Files are named `YYYY-MM-DD.csv` and stored in `telemetry/`.

| Field | Type | Description |
|---|---|---|
| `query_id` | UUID | Matches the corresponding dataset row(s) |
| `timestamp` | ISO 8601 | UTC datetime of the execution |
| `collector_version` | String | Application version (e.g., `0.1.0`) |
| `schema_version` | String | Dataset schema version |
| `success` | Boolean | Whether the pipeline completed without error |
| `http_status` | Integer | HTTP response status from the Uber API |
| `latency_ms` | Integer | Total round-trip time for the Uber API call |
| `retry_count` | Integer | Number of retries attempted (0 = first attempt succeeded) |
| `exception` | String | Exception class name, if any; null on success |
| `backup_success` | Boolean | Whether the GitHub backup succeeded |
| `rows_written` | Integer | Number of dataset rows written in this execution |

---

## 9. Scheduling

Scheduling is implemented using Spring's `@Scheduled` annotation. The execution interval is defined in `application.yml` and defaults to a fixed delay rather than a fixed rate, ensuring that a slow or retried execution does not cause overlap with the next.

The scheduler does not attempt to catch up on missed executions. If the service restarts, it resumes from the next scheduled slot.

The frequency is intentionally configurable to allow adjustment based on observed API rate limit behaviour without code changes.

---

## 10. Retry Strategy

The Collector applies a simple linear retry strategy on API failures:

- **Maximum retries:** Configurable via `application.yml` (default: 3).
- **Retry delay:** Fixed delay between attempts (configurable).
- **Retryable conditions:** HTTP 429 (rate limited), HTTP 5xx (server error), connection timeout.
- **Non-retryable conditions:** HTTP 400 (bad request), HTTP 401 (auth failure). These indicate a configuration problem and should surface immediately.
- **Exhausted retries:** If all retries fail, the pipeline logs the failure to telemetry and continues. The next scheduled execution starts fresh.

The retry count and exception class are recorded in the telemetry CSV for every execution, enabling post-hoc analysis of failure patterns.

---

## 11. Backup Strategy

Dataset and telemetry CSVs are committed and pushed to a dedicated GitHub repository once per day by the Backup Manager.

**Rationale:** GitHub provides free, versioned, off-host storage. Each commit is a point-in-time snapshot of the collected data. In the event of host failure or accidental deletion, the full dataset history is recoverable.

**Implementation notes:**

- The Backup Manager shells out to `git` commands (add, commit, push). A JGit integration may be considered if shell-out proves unreliable.
- Credentials are provided via an access token stored in `application.yml`.
- Backup success or failure is recorded in the telemetry CSV.
- A backup failure does not halt collection.

---

## 12. Configuration

All runtime configuration is externalized to `application.yml`. No configuration values are hardcoded in application source code.

**Configurable parameters:**

| Parameter | Description |
|---|---|
| `uber.api.key` | Uber API authentication key |
| `uber.api.base-url` | Uber API base URL |
| `weather.api.key` | Weather API authentication key |
| `weather.api.base-url` | Weather API base URL |
| `collector.schedule.interval-ms` | Polling interval in milliseconds |
| `collector.retry.max-attempts` | Maximum retry count per execution |
| `collector.retry.delay-ms` | Delay between retries in milliseconds |
| `collector.output.data-dir` | Directory for dataset CSV files |
| `collector.output.telemetry-dir` | Directory for telemetry CSV files |
| `backup.github.repo` | Target GitHub repository (owner/repo) |
| `backup.github.token` | GitHub personal access token |

Sensitive values (API keys, tokens) should be supplied via environment variables that `application.yml` references using Spring's `${ENV_VAR}` syntax. They must never be committed to source control.

---

## 13. Error Handling

The Collector is designed to continue operating in the presence of transient failures.

- **API errors** are caught, logged to telemetry, and retried according to the retry strategy. If retries are exhausted, the pipeline records the failure and waits for the next scheduled execution.
- **Parse errors** are caught and logged. A row is not written to the dataset CSV if parsing fails. The failure is recorded in telemetry with the exception class name.
- **Write errors** (e.g., disk full) are caught and logged. The service does not crash; the next execution will attempt to write again.
- **Backup errors** are caught and logged to telemetry. A backup failure has no effect on ongoing data collection.
- **Unhandled exceptions** at the scheduler level are caught by a global error handler, logged, and treated as execution failures. The scheduler continues to fire at the next interval.

All failures are visible through the telemetry CSV. The `success`, `exception`, and `backup_success` fields provide a clear audit trail for diagnosing recurring issues.

---

## 14. Future Improvements

The following improvements are explicitly deferred and may be introduced in later phases:

- **Multiple ride providers.** The Uber Client is one implementation of a provider interface. Additional providers (e.g., Lyft, Ola) can be added by implementing the same interface and registering them with the pipeline.
- **Improved coordinate sampling.** The initial coordinate pool is static. Future versions may generate coordinates dynamically or weight sampling toward high-traffic areas.
- **Additional weather features.** Wind speed, humidity, and precipitation intensity may improve model accuracy and can be added as new dataset columns.
- **Database migration.** If CSV files become unwieldy at scale, migration to a lightweight database (e.g., SQLite or PostgreSQL) can be considered. The Dataset Writer abstracts the persistence layer, making this a contained change.
- **ML feature engineering.** The dataset schema may evolve to include derived features (e.g., rush hour flag, time since last surge) that simplify model training.
- **Android integration.** The Collector may eventually expose a read endpoint for the Android application to consume. This is currently out of scope.

---

## 15. Assumptions

- The Uber Pricing API is available in the target city and returns estimates without requiring a logged-in user context.
- The Weather API provides current conditions by geographic coordinate.
- The deployment host (Render free tier) provides sufficient uptime for continuous collection.
- GitHub remains available and free for use as a backup store.
- The data volume generated at the configured collection interval fits comfortably within free-tier storage and bandwidth limits.
- A single JVM process is sufficient; no horizontal scaling is required.

---

## 16. Known Limitations

- **No deduplication.** If the service restarts mid-execution, a duplicate row may be written. This is considered acceptable at the current scale and will be addressed in the dataset review phase.
- **No data validation on write.** The Dataset Writer trusts the Parser output. Malformed rows are not detected at write time.
- **Single provider.** The initial implementation supports Uber only. The architecture accommodates additional providers, but none are integrated yet.
- **Shell-based Git backup.** The Backup Manager relies on `git` being available on the host. This is an operational assumption that must be verified during deployment.
- **No alerting.** The Collector logs failures to telemetry CSV but does not send alerts. Failure detection requires periodic manual inspection of the telemetry files or an external monitoring tool.
- **Free-tier deployment constraints.** Render's free tier may spin down the service during periods of inactivity, causing missed collection windows. This trade-off is accepted for the prototype phase.