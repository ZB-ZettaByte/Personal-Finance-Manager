# Personal Finance Manager

A desktop personal finance app built with Spring Boot and Swing. Log expenses, set budgets, visualize spending trends, and detect anomalies — all stored locally with no internet required.

---

## Features

- **Secure login** — BCrypt-hashed passwords (Spring Security Crypto)
- **Expense tracking** — add, delete, search, and sort by category, amount, or date
- **Budget goals** — set monthly or weekly limits with real-time progress alerts
- **Analytics** — category breakdown, monthly trends, anomaly detection, and spend forecast
- **CSV import/export** — bring in existing data or back up your expenses
- **Light/dark theme** — toggle anytime via FlatLaf

---

## Tech Stack

- Java 21, Spring Boot 3.2.5
- Spring Data JPA + Hibernate + H2 (file-backed database)
- Spring Security Crypto (BCrypt)
- JFreeChart, FlatLaf, ModelMapper
- Maven

---

## Run Locally

**Requirements:** Java 21+, Maven 3.8+

```bash
mvn clean package
java -jar target/personal-finance-manager-1.0.0.jar
```

Or:

```bash
mvn spring-boot:run
```

A `sample_expenses.csv` is included — import it from the Expenses tab to get started.

---

## Testing

```bash
mvn test
```

Covers unit tests (analytics, CSV), `@DataJpaTest` repository slice tests, and Mockito-based service tests.

---

## Academic Context

Developed as an Honors-level project for CSE 205 at Arizona State University.
