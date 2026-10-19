# Agnos Hospital Middleware API

Middleware API for searching patient information across hospitals.
Hospital staff log in and can only see patients that belong to their own hospital.

**Tech stack:** Go, Gin, PostgreSQL, Docker Compose, Nginx

## Quick start

Requirements: Docker Desktop (with Docker Compose).

1. Create the environment file and set your own values:

   ```bash
   cp .env.example .env
   ```

   Use letters and digits only in `POSTGRES_PASSWORD` (it is placed inside a connection URL).
   Make `JWT_SECRET` a long random string.

2. Build and start everything (nginx + api + postgres):

   ```bash
   docker compose up -d --build
   ```

3. Check that it works:

   ```bash
   curl http://localhost/health
   ```

   Expected: `{"message":"ok"}`

Only nginx is exposed to the host (port 80). The API and the database are reachable only inside the Docker network.

The database schema and sample data are loaded automatically from `migrations/` the first time the
database volume is created. To reset the database: `docker compose down -v` and start again.

## Sample data

| Hospital   | Patient (EN)     | National ID     |
|------------|------------------|-----------------|
| Hospital A | Somchai Jaidee   | 1100100000011   |
| Hospital A | Somying Rakdee   | 1100100000022   |
| Hospital B | Wichai Mankong   | 1100100000033   |

All data is fictional. Two hospitals exist: `Hospital A` and `Hospital B`.

## API

All request and response bodies are JSON. Errors have the form `{"error": "message"}`.

### POST /staff/create

Create a staff member. Passwords are stored as bcrypt hashes.

Request:

```json
{ "username": "nurse1", "password": "pass1234", "hospital": "Hospital A" }
```

| Status | Meaning |
|--------|---------|
| 201 | Created: `{"message":"staff created"}` |
| 400 | Invalid body, password shorter than 6 characters, or unknown hospital |
| 409 | Username already exists |

### POST /staff/login

Request:

```json
{ "username": "nurse1", "password": "pass1234" }
```

| Status | Meaning |
|--------|---------|
| 200 | `{"token":"<jwt>"}` (valid for 24 hours) |
| 400 | Invalid body |
| 401 | Wrong username or password (same message for both) |

### POST /patient/search (login required)

Send the token as `Authorization: Bearer <token>`. Every field is optional; an empty body returns
all patients of the staff member's hospital.

```json
{
  "national_id": "",
  "passport_id": "",
  "first_name": "",
  "middle_name": "",
  "last_name": "",
  "date_of_birth": "1990-05-12",
  "phone_number": "",
  "email": ""
}
```

- `national_id`, `passport_id`, `phone_number`: exact match.
- `first_name`, `middle_name`, `last_name`: partial, case-insensitive match against both the Thai and the English name.
- `email`: exact match, case-insensitive.
- `date_of_birth`: `YYYY-MM-DD`.
- Filters are combined with AND.

Response `200`: an array of patients (empty array when nothing matches). Each patient has
`first_name_th`, `middle_name_th`, `last_name_th`, `first_name_en`, `middle_name_en`, `last_name_en`,
`date_of_birth`, `patient_hn`, `national_id`, `passport_id`, `phone_number`, `email`, `gender` (`M` / `F`).

| Status | Meaning |
|--------|---------|
| 200 | Matching patients of the same hospital |
| 400 | Invalid body or date format |
| 401 | Missing, invalid or expired token |

### GET /patient/search/{id} (login required)

`id` is a national ID or a passport ID. Returns one patient of the staff member's hospital.

| Status | Meaning |
|--------|---------|
| 200 | The patient |
| 401 | Missing, invalid or expired token |
| 404 | Not found (also returned for patients of other hospitals) |

### Example

```bash
curl -X POST http://localhost/staff/create -H "Content-Type: application/json" \
  -d '{"username":"nurse1","password":"pass1234","hospital":"Hospital A"}'

TOKEN=$(curl -s -X POST http://localhost/staff/login -H "Content-Type: application/json" \
  -d '{"username":"nurse1","password":"pass1234"}' | sed 's/.*"token":"\([^"]*\)".*/\1/')

curl -X POST http://localhost/patient/search -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"national_id":"1100100000011"}'

curl http://localhost/patient/search/1100100000011 -H "Authorization: Bearer $TOKEN"
```

## Tests

```bash
go test ./... -cover
```

Unit tests cover the handlers, the JWT middleware and the services, including negative cases
(bad input, duplicate username, wrong password, expired or forged token, other hospital's patient).
Dependencies are replaced by fakes, so no database is needed.

## Project structure

See [docs/PLANNING.md](docs/PLANNING.md) for the project structure, API spec and ER diagram.

## Design notes and assumptions

- **Hospital A API:** the assignment names an external route (`hospital-a.api.co.th`). It is implemented
  here as a local endpoint backed by the `patients` table.
- **Hospital isolation:** the hospital is read from the signed JWT, never from the request body, and is
  always part of the SQL `WHERE` clause.
- `GET /patient/search/{id}` also requires login, because it returns patient data.
- `/patient/search` is a `POST` because it takes many optional fields.

## Known limitations

- `/staff/create` is open, as written in the assignment. In production it should be restricted to administrators.
- No rate limiting on login and no token refresh or revocation.
- The repository (SQL) layer has no automated tests yet; it would need integration tests against a real PostgreSQL.
