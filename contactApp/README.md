# ContactApp — Spring Boot REST API

A Spring Boot REST application for managing contacts with pre-loaded emergency numbers, company name validation, and phone number uniqueness enforcement.

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Features](#features)
- [Data Model](#data-model)
- [Business Rules](#business-rules)
- [Emergency Contacts (Pre-loaded)](#emergency-contacts-pre-loaded)
- [API Endpoints](#api-endpoints)
- [Setup & Running](#setup--running)
- [Known Issues & Gaps](#known-issues--gaps)

---

## Tech Stack

| Layer        | Technology                        |
|--------------|-----------------------------------|
| Language     | Java 8                            |
| Framework    | Spring Boot 2.7.8                 |
| ORM          | Spring Data JPA / Hibernate       |
| Database     | MySQL                             |
| Build Tool   | Maven                             |
| Packaging    | WAR (deployable on Tomcat)        |
| Boilerplate  | Lombok                            |
| Dev Tools    | Spring Boot DevTools              |

---

## Project Structure

```
contactApp/
├── pom.xml
└── src/
    └── main/
        ├── java/com/valueshipr/contactapp/app/
        │   ├── SpringbootcrudapplicationApplication.java   ← Entry point
        │   ├── ServletInitializer.java                    ← WAR support
        │   ├── model/
        │   │   └── Contact.java                           ← JPA Entity
        │   ├── repository/
        │   │   └── ContactRepository.java                 ← Spring Data JPA
        │   ├── contactserviceI/
        │   │   └── ContactServiceI.java                   ← Service Interface
        │   ├── serviceImpl/
        │   │   └── ContactService.java                    ← Business Logic
        │   └── controller/
        │       └── ContactController.java                 ← REST Controller
        └── resources/
            └── application.properties                     ← DB & server config
```

---

## Features

- Save a new contact with name, address, profile picture (URL/path), company name, primary number, and unlimited alternative numbers
- View all saved contacts
- Update an existing contact by ID
- Delete a contact by ID
- Search a contact by phone number
- Pre-loaded emergency contacts (seeded on startup via `addEmergencyContacts()`)
- Company name format validation (`ValueshiprN`)
- Global phone number uniqueness (primary number checked across all contacts)

---

## Data Model

### `Contact` Entity

| Field                | Type           | Description                                      |
|----------------------|----------------|--------------------------------------------------|
| `id`                 | `Long`         | Auto-generated primary key                       |
| `name`               | `String`       | Contact's full name                              |
| `address`            | `String`       | Contact's address                                |
| `pic`                | `String`       | Profile picture (URL or file path)               |
| `companyName`        | `String`       | Must match pattern `Valueshipr[0-9]+`            |
| `number`             | `String`       | Primary phone number — must be **unique**        |
| `alternativeNumbers` | `List<String>` | Stored as a separate collection table (`@ElementCollection`) |

---

## Business Rules

### 1. Company Name Format
The `companyName` field must match the regex pattern `Valueshipr[0-9]+`.

Valid examples: `Valueshipr1`, `Valueshipr2`, `Valueshipr100`

Invalid examples: `valueshipr1` (wrong case), `Valueshipr` (no number), `Acme Corp`

> **Note:** Emergency contacts are exempt from this rule — they have `null` company names and are seeded directly via the repository.

### 2. Phone Number Uniqueness
The primary `number` must be **unique across all contacts**. Duplicate primary numbers will throw a `RuntimeException`.

> **Current Gap:** Alternative numbers are not checked for uniqueness against the full dataset. See [Known Issues](#known-issues--gaps).

### 3. Multiple Alternative Numbers
A contact can have any number of alternative phone numbers stored in `alternativeNumbers` (a `List<String>`). These are persisted in a separate join table via `@ElementCollection`.

---

## Emergency Contacts (Pre-loaded)

These contacts are seeded into the database on application startup via `addEmergencyContacts()`:

| Name                 | Primary Number | Alternative Number |
|----------------------|----------------|--------------------|
| Citizens Call Centre | 155300         | —                  |
| Child Helpline       | 1098           | —                  |
| Women Helpline       | 1091           | —                  |
| Crime Stopper        | 1090           | —                  |
| Rescue and Relief    | 1070           | —                  |
| Ambulance            | 102            | 108                |
| Police Helpline      | 100            | —                  |
| Railway Helpline     | 23004000       | —                  |

> **Note:** The `addEmergencyContacts()` method is defined in `ContactService` and declared in `ContactServiceI`, but is currently **not wired to any startup trigger** (the `@PostConstruct` block in the controller is commented out). See [Known Issues](#known-issues--gaps).

---

## API Endpoints

Base URL: `http://localhost:9191`

### Save a Contact
```
POST /save
Content-Type: application/json
```
**Request Body:**
```json
{
  "name": "John Doe",
  "address": "123 Main St, Pune",
  "pic": "https://example.com/photo.jpg",
  "companyName": "Valueshipr5",
  "number": "9876543210",
  "alternativeNumbers": ["9876543211", "9876543212"]
}
```
**Response:** Saved `Contact` object with generated `id`.

**Validation failures throw `RuntimeException` for:**
- Invalid `companyName` format
- Duplicate `number`

---

### Get All Contacts
```
GET /get
```
**Response:** Array of all `Contact` objects.

---

### Update a Contact
```
PUT /update/{id}
Content-Type: application/json
```
**Path Variable:** `id` — the contact's database ID

**Request Body:** Same structure as Save. All fields are overwritten.

**Response:** Updated `Contact` object.

---

### Delete a Contact
```
DELETE /delete/{id}
```
**Path Variable:** `id` — the contact's database ID

**Response:** `204 No Content`

---

### Search by Phone Number
```
GET /getbynumber/{number}
```
**Path Variable:** `number` — the primary phone number to search

**Response:** Matching `Contact` object, or `null` if not found.

---

## Setup & Running

### Prerequisites

- Java 8+
- Maven 3.6+
- MySQL 5.7+ running locally

### 1. Create the Database

```sql
CREATE DATABASE contactapp;
```

### 2. Configure Credentials

Edit `src/main/resources/application.properties`:

```properties
server.port=9191

spring.datasource.url=jdbc:mysql://localhost:3306/contactapp
spring.datasource.username=root
spring.datasource.password=root   # ← change this

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### 3. Build & Run

```bash
# Build
./mvnw clean package -DskipTests

# Run as Spring Boot app
./mvnw spring-boot:run

# OR deploy the WAR to an external Tomcat
cp target/springbootcrudapplication-0.0.1-SNAPSHOT.war <TOMCAT_HOME>/webapps/
```

### 4. Seed Emergency Contacts

Because the startup hook is currently commented out, call the seed method manually after startup. The quickest fix is to add a `@PostConstruct` in `ContactController` or `ContactService`:

```java
// In ContactService.java — add this method and annotate it:
@PostConstruct
public void init() {
    if (contactRepository.count() == 0) {
        addEmergencyContacts();
    }
}