# 🎮 Fast MCQ — Multiplayer Quiz Game (Spring Boot Backend)

## Table of Contents
- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Architecture Overview](#architecture-overview)
- [Data Models](#data-models)
- [API Endpoints](#api-endpoints)
- [Database Configuration](#database-configuration)
- [How to Run](#how-to-run)
- [Game Flow](#game-flow)
- [Validations](#validations)
- [Current Gaps vs Problem Statement](#current-gaps-vs-problem-statement)
- [What Needs to Be Built (Roadmap)](#what-needs-to-be-built-roadmap)
- [Known Issues & Code Observations](#known-issues--code-observations)

---

## Project Overview

**Fast MCQ** is a multiplayer, real-time quiz game built with Spring Boot. Players join a lobby, wait for the Admin to start the game, then race to answer randomly fetched MCQ questions as fast as possible. The player who answers correctly in the **least time** wins and is crowned the **Champion**.

Questions are sourced from:
> https://www.avatto.com/general-knowledge/questions/mcqs/kbc/answers/285/1.html

---

## Problem Statement

| Requirement | Description |
|---|---|
| Multiplayer | Multiple users can join and play simultaneously |
| Admin Role | First user to join gets username **AB** and becomes Admin |
| Unique Usernames | All other players must choose distinct usernames |
| Lobby / Waiting Room | Players wait in a lobby until the Admin starts the game |
| Random Questions | Questions are fetched from the Avatto KBC MCQ page |
| Timestamped Answers | Each player's answer is recorded with the exact submission time |
| Champion Selection | Player with the correct answer and the fastest response time wins |
| Validations | Username uniqueness, answer validity, game state checks |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 8 |
| Framework | Spring Boot 2.7.8 |
| ORM | Hibernate 5.6.3 + Spring Data JPA 2.6.3 |
| Database | MySQL 5.x |
| Build Tool | Maven |
| Dev Tools | Spring Boot DevTools |
| Testing | Spring Boot Test (JUnit) |

---

## Project Structure

```
gameApp/
├── src/
│   └── main/
│       ├── java/com/valueshipr/gameapp/app/
│       │   ├── GameApplication.java              ← Entry point
│       │   ├── controller/
│       │   │   └── GameController.java           ← REST API layer
│       │   ├── model/
│       │   │   ├── Game.java                     ← Game entity (id, name, questions)
│       │   │   ├── Question.java                 ← Question entity (prompt, choices, answer)
│       │   │   └── AnswerType.java               ← Enum: MCQ | WORD | INTEGER
│       │   ├── repository/
│       │   │   └── GameRepository.java           ← JPA repository for Game
│       │   ├── service/
│       │   │   └── GameService.java              ← Service interface
│       │   └── serviceimpl/
│       │       └── GameServiceImpl.java          ← Service implementation
│       └── resources/
│           └── application.properties            ← DB config, server port
├── pom.xml
└── README.md
```

---

## Architecture Overview

```
Client (Postman / Frontend)
        │
        ▼
  GameController  (REST Layer — @RestController)
        │
        ▼
  GameService     (Interface)
        │
        ▼
  GameServiceImpl (Business Logic — @Service)
        │
        ▼
  GameRepository  (Data Access — JpaRepository<Game, Long>)
        │
        ▼
     MySQL DB     (gameapp schema)
```

---

## Data Models

### `Game` Entity (`games` table)

| Field | Type | Description |
|---|---|---|
| `id` | Long (PK, auto-increment) | Unique game identifier |
| `name` | String | Name of the game session |
| `questions` | List\<Question\> | One-to-many: list of questions in this game |

### `Question` Entity (`questions` table)

| Field | Type | Description |
|---|---|---|
| `id` | Long (PK, auto-increment) | Unique question identifier |
| `prompt` | String | The question text |
| `answerType` | AnswerType (Enum) | MCQ / WORD / INTEGER |
| `choices` | List\<String\> | Answer options (stored as @ElementCollection) |
| `correctAnswer` | String | The correct answer value |
| `optimizableAnswer` | String | Best/optimal answer (reserved) |
| `wrongAnswer` | String | An incorrect answer (reserved) |

### `AnswerType` Enum

```java
MCQ      // Multiple choice question
WORD     // Free text word answer
INTEGER  // Numeric answer
```

---

## API Endpoints

Base URL: `http://localhost:9192`

| Method | Endpoint | Description | Request Body |
|---|---|---|---|
| `POST` | `/saveGame` | Create a new game with questions | `Game` JSON |
| `GET` | `/getAll` | Fetch all saved games | — |
| `GET` | `/getById/{id}` | Fetch a specific game by ID | — |
| `PUT` | `/updateGame/{id}` | Update a game's name and questions | `Game` JSON |
| `DELETE` | `/deleteGame/{id}` | Delete a game by ID | — |

### Sample Request — Create Game

```json
POST /saveGame
{
  "name": "KBC Quiz Round 1",
  "questions": [
    {
      "prompt": "Who is the founder of Microsoft?",
      "answerType": "MCQ",
      "choices": ["Steve Jobs", "Bill Gates", "Elon Musk", "Mark Zuckerberg"],
      "correctAnswer": "Bill Gates"
    }
  ]
}
```

### Sample Response

```json
HTTP 201 Created
Location: /games/1
{
  "id": 1,
  "name": "KBC Quiz Round 1",
  "questions": [...]
}
```

---

## Database Configuration

File: `src/main/resources/application.properties`

```properties
server.port=9192

spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/gameapp
spring.datasource.username=root
spring.datasource.password=root

spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
```

> ⚠️ Before running, create the MySQL database:
> ```sql
> CREATE DATABASE gameapp;
> ```

---

## How to Run

### Prerequisites
- Java 8+
- Maven 3.6+
- MySQL running on `localhost:3306`

### Steps

```bash
# 1. Clone / extract the project
cd gameApp

# 2. Create the database in MySQL
mysql -u root -p -e "CREATE DATABASE gameapp;"

# 3. Build the project
./mvnw clean install

# 4. Run the application
./mvnw spring-boot:run
```

Application starts at: `http://localhost:9192`

---

## Game Flow

```
1. Player 1 joins → assigned username "AB" → becomes Admin
2. Other players join → choose unique usernames
3. All players land on the Lobby (waiting screen)
4. Admin clicks "Start Game"
5. A random MCQ question is fetched from Avatto KBC page
6. Question + options are broadcast to all players simultaneously
7. Each player selects an answer → timestamp is recorded server-side
8. After all answers submitted (or timer expires):
   → Correct answers identified
   → Among correct answers, the one with lowest response time wins
9. Champion is announced
10. Next question starts (or game ends)
```

---

## Validations

| Validation | Rule |
|---|---|
| Admin Username | First joiner is always assigned `AB`; not changeable |
| Username Uniqueness | No two players can have the same username |
| Username Blank | Username cannot be empty or null |
| Game Start | Only Admin (`AB`) can start the game |
| Answer Submission | Players cannot submit answers before the game starts |
| Duplicate Answer | A player can only submit one answer per question |
| Answer Options | Submitted answer must match one of the given choices |
| Timestamp Recording | Server-side timestamp is used (client time is not trusted) |

---

## Current Gaps vs Problem Statement

The existing codebase provides a **basic CRUD scaffold** for Game and Question entities. The following features from the problem statement are **not yet implemented**:

| Feature | Status | Notes |
|---|---|---|
| Multiplayer / WebSocket support | ❌ Missing | Needs Spring WebSocket or STOMP |
| Admin (`AB`) auto-assignment | ❌ Missing | No player/session management exists |
| Unique username enforcement | ❌ Missing | No `Player` entity or lobby logic |
| Lobby / waiting room | ❌ Missing | No game state machine (WAITING → STARTED) |
| Question fetching from Avatto URL | ❌ Missing | Needs a web scraper (Jsoup recommended) |
| Server-side answer timestamping | ❌ Missing | No `PlayerAnswer` entity with timestamp |
| Champion determination logic | ❌ Missing | No comparison of answer times |
| Game state management | ❌ Missing | No `GameStatus` enum (LOBBY/IN_PROGRESS/ENDED) |