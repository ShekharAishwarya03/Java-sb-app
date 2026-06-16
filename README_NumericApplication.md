# Numeric Application — Kubernetes DevSecOps Service

## Overview

`NumericApplication` is a Spring Boot REST API designed for a Kubernetes-based DevSecOps pipeline. It exposes endpoints to compare numeric values and increment them via a downstream Node.js microservice. Telemetry and observability are built-in using **Azure Application Insights**.

---

## Project Structure

```
com.devsecops/
├── NumericApplication.java     # Spring Boot entry point
└── NumericController.java      # REST controller with business logic
```

---

## Technology Stack

| Technology              | Purpose                                      |
|-------------------------|----------------------------------------------|
| Java + Spring Boot      | Application framework                        |
| Spring Web (RestTemplate) | HTTP client for inter-service communication |
| Azure Application Insights | Telemetry, metrics, and event tracking    |
| SLF4J                   | Logging                                      |
| Kubernetes              | Container orchestration and deployment       |

---

## API Endpoints

### `GET /`
Returns a welcome message confirming the service is running.

**Response:**
```
Kubernetes DevSecOps
```

---

### `GET /compare/{value}`
Compares a given integer against the threshold value of **50**.

**Path Parameter:**

| Parameter | Type | Description              |
|-----------|------|--------------------------|
| `value`   | int  | The integer to evaluate  |

**Response:**

| Condition        | Message                        |
|------------------|--------------------------------|
| `value > 50`     | `Greater than 50`              |
| `value <= 50`    | `Smaller than or equal to 50`  |

**Example:**
```
GET /compare/75  →  "Greater than 50"
GET /compare/30  →  "Smaller than or equal to 50"
```

**Telemetry tracked:**
- Event: `CompareToFifty`
- Metric: `ComparedValue` (the input value)

---

### `GET /increment/{value}`
Increments a given integer by calling the downstream **Node.js microservice** and returns the result.

**Path Parameter:**

| Parameter | Type | Description              |
|-----------|------|--------------------------|
| `value`   | int  | The integer to increment |

**Downstream Service:** `http://node-service:5000/plusone/{value}`

**Example:**
```
GET /increment/9  →  10
```

**Telemetry tracked:**
- Event: `IncrementValue`
- Metric: `ProcessingTime` (milliseconds taken)
- Metric: `InputValue` (original value)
- Metric: `OutputValue` (incremented result)

---

## Inter-Service Communication

The `/increment` endpoint depends on a running **Node.js service** accessible within the Kubernetes cluster at:

```
http://node-service:5000/plusone
```

Ensure this service is deployed and reachable before calling `/increment`. In a local environment, update the `baseURL` in `NumericController.java` or configure it via application properties.

---

## Observability & Telemetry

This service integrates with **Azure Application Insights** (`TelemetryClient`) for:

- **Custom Events** — track named business events (`CompareToFifty`, `IncrementValue`)
- **Custom Metrics** — track numeric values like processing time and input/output values
- **Request Telemetry** — trace individual request details

Ensure the Application Insights **Instrumentation Key** is configured in `application.properties` or as an environment variable:

```properties
azure.application-insights.instrumentation-key=<YOUR_KEY>
```

---

## Running the Application

### Prerequisites

- Java 11+
- Maven or Gradle
- Azure Application Insights dependency on the classpath
- Node.js microservice running at `http://node-service:5000` (for `/increment`)

### Run Locally

```bash
# Build
mvn clean package

# Run
java -jar target/numeric-application.jar
```

### Run in Kubernetes

```bash
# Apply your deployment manifest
kubectl apply -f deployment.yaml

# Verify pods are running
kubectl get pods
```

---

## Logging

SLF4J is used for structured logging. Key log entries include:

- Incoming comparison requests with input value and result
- Node service request/response values for the increment flow

Logs are output at the `INFO` level and can be forwarded to any compatible log aggregator (e.g., Azure Monitor, ELK Stack).

---

## Notes

- The inner class `compare` inside `NumericController` is annotated with `@RestController`, which means both the outer and inner classes register REST endpoints. Refactoring into a single flat controller is recommended for clarity.
- `RestTemplate` is instantiated directly in the controller. Consider injecting it as a `@Bean` via `RestTemplateBuilder` for better testability and configuration control.
- No authentication or authorization is applied to the endpoints. Add Spring Security if this service is exposed outside the cluster.
