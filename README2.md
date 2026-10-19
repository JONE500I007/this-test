# Development Planning

This document contains the three planning deliverables: Project Structure, API Spec and ER Diagram.

## 1. Project Structure

```
Agnos-Hospital-API/
├── cmd/server/main.go            # entry point: config, DB pool, routes
├── internal/
│   ├── config/config.go          # reads environment variables
│   ├── model/                    # Patient, PatientSearch, Staff, shared errors
│   ├── repository/               # SQL, talks to PostgreSQL (staff.go, patient.go)
│   ├── service/                  # business logic: bcrypt, JWT, (staff.go, patient.go)
│   ├── handler/                  # Gin HTTP handlers (staff.go, patient.go)
│   └── middleware/auth.go        # validates the JWT, puts hospital_id in the request context
├── migrations/001_init.sql       # schema + sample data (loaded by postgres on first start)
├── nginx/nginx.conf              # reverse proxy to the api service
├── Dockerfile                    # multi-stage build of the Go service
├── docker-compose.yml            # nginx + api + postgres
└── README.md
```

Layers and their dependencies:

```
handler  ->  service  ->  repository  ->  PostgreSQL
   ^            ^
 middleware   (each layer depends on an interface, so tests use fakes)
```

Request flow:

```
client -> nginx :80 -> api :8080 -> postgres :5432
```

## 2. API Spec

Base URL: `http://localhost` (through nginx). JSON in, JSON out. Errors: `{"error": "..."}`.

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | /staff/create | no | create a staff member |
| POST | /staff/login | no | get a JWT |
| POST | /patient/search | Bearer token | search patients of the staff member's hospital |
| GET | /patient/search/{id} | Bearer token | find one patient by national ID or passport ID |
| GET | /health | no | health check (includes a database ping) |

### POST /staff/create

Request: `{"username": string, "password": string (min 6), "hospital": string}`

| Status | Body |
|--------|------|
| 201 | `{"message":"staff created"}` |
| 400 | invalid body / password too short / hospital not found |
| 409 | username already exists |

### POST /staff/login

Request: `{"username": string, "password": string}`

| Status | Body |
|--------|------|
| 200 | `{"token":"<jwt>"}` (HS256, 24 h, claims: `staff_id`, `hospital_id`, `exp`) |
| 400 | invalid body |
| 401 | invalid username or password |

### POST /patient/search

Header: `Authorization: Bearer <token>`

Request (all optional): `national_id`, `passport_id`, `first_name`, `middle_name`, `last_name`,
`date_of_birth` (YYYY-MM-DD), `phone_number`, `email`

| Status | Body |
|--------|------|
| 200 | array of patients (possibly empty), only from the staff member's hospital |
| 400 | invalid body / date format |
| 401 | missing, invalid or expired token |

Patient object: `first_name_th`, `middle_name_th`, `last_name_th`, `first_name_en`, `middle_name_en`,
`last_name_en`, `date_of_birth`, `patient_hn`, `national_id`, `passport_id`, `phone_number`, `email`,
`gender` (M or F).

### GET /patient/search/{id}

Header: `Authorization: Bearer <token>`

| Status | Body |
|--------|------|
| 200 | patient object |
| 401 | missing, invalid or expired token |
| 404 | not found (also for patients of other hospitals) |

## 3. ER Diagram

```
┌──────────────┐       ┌────────────────────┐
│  hospitals   │ 1   N │       staff        │
├──────────────┤◄──────├────────────────────┤
│ id (PK)      │       │ id (PK)            │
│ name (UQ)    │       │ username (UQ)      │
└──────┬───────┘       │ password_hash      │
       │ 1             │ hospital_id (FK)   │
       │               └────────────────────┘
       │ N
┌──────┴───────────────────┐
│         patients         │
├──────────────────────────┤
│ id (PK)                  │
│ hospital_id (FK)         │
│ first/middle/last_name_th│
│ first/middle/last_name_en│
│ date_of_birth            │
│ patient_hn               │
│ national_id   (indexed)  │
│ passport_id   (indexed)  │
│ phone_number             │
│ email                    │
│ gender  (M | F)          │
└──────────────────────────┘
```

A hospital has many staff and many patients. A staff member can only search patients with the same `hospital_id`.

DBML (paste into https://dbdiagram.io to draw and export the diagram as an image):

```
Table hospitals {
  id serial [pk]
  name text [not null, unique]
}

Table staff {
  id serial [pk]
  username text [not null, unique]
  password_hash text [not null]
  hospital_id int [not null, ref: > hospitals.id]
}

Table patients {
  id serial [pk]
  hospital_id int [not null, ref: > hospitals.id]
  first_name_th text
  middle_name_th text
  last_name_th text
  first_name_en text
  middle_name_en text
  last_name_en text
  date_of_birth date
  patient_hn text
  national_id text [note: 'indexed']
  passport_id text [note: 'indexed']
  phone_number text
  email text
  gender char(1) [note: "M or F"]
}
```

## 4. Key design decisions

- **Hospital isolation:** the `hospital_id` comes from the signed JWT, is stored in the request context by the
  middleware, and is always part of the SQL `WHERE` clause. A client cannot choose another hospital.
- **Passwords:** stored only as bcrypt hashes. Login failures return the same message for an unknown user and
  a wrong password.
- **SQL injection:** all values go through query parameters (`$1`, `$2`, ...); only fixed SQL fragments are concatenated.
- **Testability:** handlers and services depend on small interfaces, so unit tests use fakes without a database.
- **Docker:** only nginx is published to the host; PostgreSQL has a healthcheck and the API waits for it.
