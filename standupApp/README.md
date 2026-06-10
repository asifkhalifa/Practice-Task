# 📋 StandUp App — Daily Stand-Up REST API

A Spring Boot REST API that allows team members to log their daily stand-up updates. The app supports credential-based user authentication, stand-up entry creation and retrieval, and enforces a blocker-escalation rule that prevents saving a fourth consecutive blocked stand-up.

---

## 📁 Project Structure

```
standupApp/
├── pom.xml
└── src/
    └── main/
        ├── java/com/valueshipr/standupapp/app/
        │   ├── SpringbootcrudapplicationApplication.java   # Entry point
        │   ├── ServletInitializer.java                     # WAR deployment support
        │   ├── controller/
        │   │   └── StandUpController.java                  # REST endpoints
        │   ├── model/
        │   │   ├── StandUp.java                            # JPA entity
        │   │   └── BlockerStatus.java                      # Enum for blocker severity
        │   ├── repository/
        │   │   └── StandUpRepository.java                  # Spring Data JPA repo
        │   ├── serviceI/
        │   │   ├── StandUpService.java                     # Stand-up service interface
        │   │   └── UserServiceI.java                       # User validation interface
        │   ├── serviceImpl/
        │   │   ├── StandUpServiceImpl.java                 # Business logic
        │   │   └── UserServiceImpl.java                    # In-memory credential store
        │   └── exception/
        │       ├── StandUpValidationException.java         # Custom exception
        │       └── StandUpValidationExceptionAdvice.java   # Global exception handler
        └── resources/
            └── application.properties                      # DB and server config
```

---

## ✅ Features

| Feature | Description |
|---|---|
| User Login | Credentials validated against a hardcoded in-memory JSON-like map |
| Save Stand-Up | Logs ToDo, Yesterday's Achievement, Blockers, and Blocker Status |
| Get Stand-Ups | Retrieves all stand-up entries for a given user |
| Blocker Validation | Blocks saving a 4th stand-up if blocker is unresolved for 3 days |
| Error Handling | Custom exception + global `@ControllerAdvice` for clean error responses |

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 8 |
| Framework | Spring Boot 2.7.8 |
| ORM | Spring Data JPA / Hibernate |
| Database | MySQL |
| Build Tool | Maven |
| Packaging | WAR (deployable to external Tomcat) |
| Utility | Lombok 1.18.24 |

---

## ⚙️ Prerequisites

- Java 8+
- Maven 3.6+
- MySQL 5.x or 8.x running locally
- (Optional) Apache Tomcat for WAR deployment

---

## 🗄️ Database Setup

1. Start your MySQL server.
2. Create the database:
   ```sql
   CREATE DATABASE standupapp;
   ```
3. The application uses `spring.jpa.hibernate.ddl-auto=update`, so the `stand_ups` table is **created automatically** on first run.

---

## 🔧 Configuration

Edit `src/main/resources/application.properties` to match your environment:

```properties
server.port=9193

# MySQL DataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/standupapp
spring.datasource.username=root
spring.datasource.password=root

# JPA / Hibernate
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
```

---

## 🚀 Running the Application

**Using Maven:**
```bash
cd standupApp
mvn spring-boot:run
```

**Or build and run the JAR:**
```bash
mvn clean package
java -jar target/springbootcrudapplication-0.0.1-SNAPSHOT.war
```

The application starts on **http://localhost:9193**.

---

## 👤 Predefined Users

Authentication is handled via an in-memory credential map (`UserServiceImpl`). The following users are available out of the box:

| Username | Password |
|---|---|
| `asif` | `password123` |
| `pratik` | `password456` |

> To add more users, modify the `UserServiceImpl` constructor.

---

## 📡 API Endpoints

### 1. Save a Stand-Up

**POST** `/saved`

Saves a new daily stand-up entry for the authenticated user.

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `username` | String | ✅ | Registered username |
| `password` | String | ✅ | User password |

**Request Body (JSON):**

```json
{
  "username": "asif",
  "toDo": "Complete the payment module",
  "yesterdayAchievement": "Finished API design for order service",
  "blockers": "Waiting for DB schema approval",
  "blockerStatus": "BLOCKED_1_DAY"
}
```

**`blockerStatus` Enum Values:**

| Value | Meaning |
|---|---|
| `NOT_BLOCKED` | No active blocker |
| `BLOCKED_1_DAY` | Blocker ongoing for 1 day |
| `BLOCKED_2_DAYS` | Blocker ongoing for 2 days |
| `BLOCKED_3_DAYS` | ❌ Saving is **blocked** — cannot submit |

**Responses:**

| HTTP Status | Condition |
|---|---|
| `200 OK` | Stand-up saved successfully |
| `400 Bad Request` | `blockerStatus` is `BLOCKED_3_DAYS` — error message returned in body |
| `401 Unauthorized` | Invalid username or password |

**Error Response Body (400):**
```
You cannot save your stand-up for today as you have not solved your blocker for 3 days.
```

---

### 2. Get All Stand-Ups

**GET** `/get`

Retrieves all stand-up entries submitted by the authenticated user.

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `username` | String | ✅ | Registered username |
| `password` | String | ✅ | User password |

**Example Request:**
```
GET http://localhost:9193/get?username=asif&password=password123
```

**Response (200 OK):**
```json
[
  {
    "id": 1,
    "username": "asif",
    "toDo": "Complete the payment module",
    "yesterdayAchievement": "Finished API design for order service",
    "blockers": "Waiting for DB schema approval",
    "blockerStatus": "BLOCKED_1_DAY"
  }
]
```

**Responses:**

| HTTP Status | Condition |
|---|---|
| `200 OK` | Returns list of stand-ups (empty array if none) |
| `401 Unauthorized` | Invalid credentials |

---

## 🔒 Blocker Escalation Rule

The `blockerStatus` field is **manually set by the client** (frontend or API consumer) and must be escalated progressively:

```
Day 1 with blocker  →  BLOCKED_1_DAY
Day 2 with blocker  →  BLOCKED_2_DAYS
Day 3 with blocker  →  BLOCKED_3_DAYS  ← saving is blocked from day 4 onward
```

> When `blockerStatus` is `BLOCKED_3_DAYS`, the server throws a `StandUpValidationException` and returns a **400 Bad Request** with a descriptive error message. The stand-up is **not** persisted.

---

## 🧩 Key Classes

### `StandUp.java` — JPA Entity

Maps to the `stand_ups` table in MySQL.

| Field | Type | Nullable | Description |
|---|---|---|---|
| `id` | Long | ❌ | Auto-generated primary key |
| `username` | String | ❌ | Owner of the stand-up |
| `toDo` | String | ❌ | Today's planned tasks |
| `yesterdayAchievement` | String | ❌ | What was completed yesterday |
| `blockers` | String | ❌ | Description of any blockers |
| `blockerStatus` | BlockerStatus (Enum) | ❌ | Current blocker severity level |

### `BlockerStatus.java` — Enum

```java
public enum BlockerStatus {
    NOT_BLOCKED,
    BLOCKED_1_DAY,
    BLOCKED_2_DAYS,
    BLOCKED_3_DAYS
}
```

### `StandUpServiceImpl.java` — Business Logic

- On `saveStandUp()`: checks if `blockerStatus == BLOCKED_3_DAYS` and throws `StandUpValidationException` if true.
- On `getStandUps()`: delegates to repository to fetch all records by `username`.

### `UserServiceImpl.java` — Authentication

- Stores credentials in a `HashMap<String, String>`.
- `validateCredentials()` looks up the username and compares the stored password.

### `StandUpValidationExceptionAdvice.java` — Global Error Handler

- Annotated with `@ControllerAdvice`.
- Catches `StandUpValidationException` and returns a `400 Bad Request` with the exception message as the response body.

---

## 🧪 Sample cURL Commands

**Save a stand-up (valid):**
```bash
curl -X POST "http://localhost:9193/saved?username=asif&password=password123" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "asif",
    "toDo": "Fix login bug",
    "yesterdayAchievement": "Code review done",
    "blockers": "None",
    "blockerStatus": "NOT_BLOCKED"
  }'
```

**Save a stand-up (blocked — 3-day unresolved blocker):**
```bash
curl -X POST "http://localhost:9193/saved?username=asif&password=password123" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "asif",
    "toDo": "Fix login bug",
    "yesterdayAchievement": "Trying to resolve blocker",
    "blockers": "DB access not granted",
    "blockerStatus": "BLOCKED_3_DAYS"
  }'
# Response: 400 Bad Request
# "You cannot save your stand-up for today as you have not solved your blocker for 3 days."
```

**Get all stand-ups:**
```bash
curl "http://localhost:9193/get?username=asif&password=password123"
```

**Invalid credentials:**
```bash
curl "http://localhost:9193/get?username=asif&password=wrongpass"
# Response: 401 Unauthorized
```