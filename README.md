# Budget Analyzer

A production-style desktop personal finance application built with **Spring Boot**, **Spring Data JPA**, **Hibernate**, and **Swing**. Demonstrates non-web Spring Boot architecture, layered service design, statistical analytics algorithms, and BCrypt-secured authentication — all without a single HTTP endpoint.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Features](#features)
- [Analytics Engine](#analytics-engine)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Testing](#testing)
- [Design Decisions](#design-decisions)

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                        Swing UI Layer                        │
│   LoginFrame  →  MainApp  →  DashboardPanel / TrendPanel /  │
│                              ExpenseTablePanel               │
└───────────────────────────┬──────────────────────────────────┘
                            │ Spring-managed beans
┌───────────────────────────▼──────────────────────────────────┐
│                      Service Layer                           │
│          UserService  │  ExpenseService  │  BudgetService   │
└───────────────────────┬──────────────────────────────────────┘
                        │ Spring Data JPA repositories
┌───────────────────────▼──────────────────────────────────────┐
│                   Persistence Layer                          │
│        UserRepository │ ExpenseRepository │ BudgetRepository │
│                     H2 File Database                         │
└──────────────────────────────────────────────────────────────┘
```

The application runs `WebApplicationType.NONE` — Spring Boot manages the IoC container, JPA lifecycle, and transaction boundaries without starting an embedded HTTP server. The Swing event dispatch thread (EDT) is kept separate from the Spring context thread per the standard AWT threading contract.

---

## Features

| Feature | Details |
|---|---|
| **Secure authentication** | BCrypt (cost factor 12 = 2^12 rounds) password hashing via Spring Security Crypto |
| **Expense management** | Add, delete, search, and sort expenses by category, amount, date, and notes |
| **Budget tracking** | Set monthly/weekly budget goals; real-time progress bar with overspend alerts |
| **Spending analytics** | Category breakdown, monthly trend charts, anomaly detection, spend forecasting |
| **CSV import/export** | RFC 4180-compliant CSV with full round-trip fidelity |
| **Theme switching** | Light/dark mode toggle via FlatLaf with live `SwingUtilities.updateComponentTreeUI` |
| **Persistent storage** | H2 file-backed database with Hibernate schema auto-update |
| **Event-driven extension** | Optional Spring Kafka integration for expense event publishing |

---

## Analytics Engine

All analytics logic lives in `SpendingAnalytics` and `SpendingAnomaly` as pure stateless functions — no instance state, no side effects, trivially unit-testable.

### Algorithms implemented

**Category aggregation**
Groups expenses by category using Java Streams + `Collectors.groupingBy` and `Collectors.summingDouble`.

**Monthly trend aggregation**
Uses a `TreeMap` collector to guarantee chronological iteration order without a post-sort step.

**Top-K categories — min-heap**
Maintains a `PriorityQueue<Entry>` of size K. For each category total, push and evict the minimum when the heap exceeds K. Time: O(c log K) where c = number of categories. Space: O(K).

**Sliding-window moving average**
Builds a `TreeMap<LocalDate, Double>` of daily totals, then applies a two-pointer window across the sorted date list. O(n) time.

**Next-month spend forecast — OLS linear regression**
Computes slope and intercept via the closed-form least-squares formula over historical monthly totals. Predicts the next month's spend as `intercept + slope * nextX`. Returns 0 when insufficient history exists.

**Anomaly detection — per-category z-score**
For each expense, computes its z-score relative to the mean and standard deviation of all expenses in the same category. Flags entries where `|z| > 2.0`. Standard deviation uses a two-pass algorithm (mean pass, then variance pass) to avoid the numerical instability of the single-pass form when values cluster near a large mean. Requires a minimum of 3 samples per category.

**Budget burn-rate**
Estimates days remaining before the budget is exhausted based on average daily spend from the earliest recorded expense to today.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.2.5 |
| Persistence | Spring Data JPA, Hibernate 6, H2 |
| Security | Spring Security Crypto (BCrypt) |
| Mapping | ModelMapper 3.2 |
| Messaging | Spring Kafka (optional) |
| UI framework | Swing (Java standard library) |
| UI look-and-feel | FlatLaf 3.4 |
| Charts | JFreeChart 1.5.4 |
| Build | Maven |
| Testing | JUnit 5, Mockito, AssertJ, `@DataJpaTest`, Embedded Kafka |

---

## Project Structure

```
src/
├── main/java/com/finance/manager/
│   ├── FinanceManagerApplication.java   # Spring Boot entry point (WebApplicationType.NONE)
│   ├── MainApp.java                     # Root Swing frame, wired as a Spring bean
│   ├── Expense.java                     # JPA entity
│   ├── User.java                        # JPA entity with Role enum
│   ├── Budget.java / BudgetConfig.java  # Budget domain + value object
│   ├── CSVHandler.java                  # RFC 4180 CSV import/export
│   ├── analytics/
│   │   ├── SpendingAnalytics.java       # Stateless analytics (regression, top-K, moving avg)
│   │   └── SpendingAnomaly.java         # Z-score anomaly detection
│   ├── config/
│   │   └── SecurityConfig.java          # BCrypt PasswordEncoder bean
│   ├── repository/                      # Spring Data JPA interfaces
│   ├── service/                         # Transactional service layer
│   └── ui/                              # Swing panels (Dashboard, Expenses, Trends, Login)
└── test/java/com/finance/manager/
    ├── analytics/                        # Unit tests for analytics algorithms
    ├── BudgetHandlerTest.java
    ├── CSVHandlerTest.java
    ├── ExpenseRepositoryTest.java        # @DataJpaTest slice tests
    └── UserServiceTest.java
```

---

## Getting Started

### Prerequisites

- Java 21+
- Maven 3.8+

### Build and run

```bash
# Build fat JAR
mvn clean package

# Run
java -jar target/personal-finance-manager-1.0.0.jar
```

Or use the Spring Boot Maven plugin:

```bash
mvn spring-boot:run
```

### Import sample data

A `sample_expenses.csv` file is included in the repo root. Use the Import button in the Expenses tab to load it.

---

## Configuration

All runtime settings are in `src/main/resources/application.yml`.

```yaml
spring:
  datasource:
    url: jdbc:h2:file:./finance-db   # file-backed H2; swap for any JDBC URL
  jpa:
    hibernate:
      ddl-auto: update               # schema auto-migrated on startup

app:
  kafka:
    enabled: false                   # set true when a broker is available
```

To enable Kafka, set `app.kafka.enabled: true` and update `spring.kafka.bootstrap-servers` to point to a running broker.

---

## Testing

```bash
mvn test
```

The test suite covers three distinct levels:

- **Unit tests** — `SpendingAnalytics`, `SpendingAnomaly`, `CSVHandler`, and `BudgetHandler` are tested with plain JUnit 5 + AssertJ. No Spring context is loaded; tests run in milliseconds.
- **Repository slice tests** — `@DataJpaTest` spins up an in-memory H2 instance to validate repository queries in isolation without loading the full application context.
- **Service tests** — `UserService` is tested with Mockito-mocked repositories to verify business logic independently of persistence.

The test `application.yml` overrides the datasource to in-memory H2 so tests never touch the production data file.

---

## Design Decisions

**Why Spring Boot for a desktop app?**
`WebApplicationType.NONE` provides dependency injection, JPA lifecycle management, and `@Transactional` semantics without an HTTP server. This eliminates manual wiring of `EntityManagerFactory`, `DataSource`, and transaction manager — non-trivial boilerplate for a standalone app. Swapping to a web frontend in the future requires only adding the web starter and a controller layer.

**Why H2 file-backed instead of SQLite?**
H2 is a first-class Spring Boot autoconfiguration target. Migrating to PostgreSQL or MySQL requires only a JDBC URL change and a driver dependency — zero code changes.

**Why two-pass standard deviation?**
The single-pass formula computes `Σx²` and `(Σx)²` separately and subtracts — this causes catastrophic cancellation when values cluster near a large mean. The two-pass approach (mean first, variance second) is numerically stable at the cost of one extra O(n) pass, which is acceptable at desktop data volumes.

**BCrypt cost factor 12**
OWASP recommends a BCrypt work factor that keeps hashing time above ~100ms on target hardware. Factor 12 (4096 rounds) satisfies that threshold on modern CPUs while remaining imperceptible at login time.

**Stateless analytics**
`SpendingAnalytics` and `SpendingAnomaly` hold no instance state. This makes every method deterministic given the same input, safe to call from any thread, and testable without mocking infrastructure.

---

## Academic Context

Developed as an Honors-level project for CSE 205 at Arizona State University.
