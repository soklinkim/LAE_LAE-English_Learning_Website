# English Online Learning Platform - Architecture

Five microservices behind one API Gateway, split by business area (not by user role). Backend first, frontend and production hardening afterwards.

## 1. Overview

```
                      Client (Postman now, web frontend later)
                                      |
                                      v
                          +-----------------------+
                          |      API Gateway      |   :3000  (only public port)
                          |  verify JWT, route    |
                          +-----------+-----------+
          +-------------------+-------+-----------+-------------------+
          |                   |                   |                   |
          v                   v                   v                   v
  +---------------+   +---------------+   +---------------+   +---------------+
  | Registration  |   |    Login      |   |   Content     |<->|   Learning    |
  |    :3001      |   |    :3002      |   |    :3003      |   |    :3004      |
  | create users  |   | verify + JWT  |   | courses       |   | enrollments   |
  |               |   |               |   | lessons       |   | progress      |
  +-------+-------+   +-------+-------+   +-------+-------+   +-------+-------+
          |                   |                   |                   |
          v                   v                   v                   v
   registration_db      registration_db        content_db         learning_db
   (users: writes)      (users: read only)                      (all on MongoDB Atlas)
```

The arrow between Content and Learning shows the two services calling each other (see section 5).

## 2. Design principles

1. **Split by business area and data ownership.** Things that change together and share data live in one service. Courses and lessons are one area (Content); enrollments and progress are another (Learning).
2. **Roles are permissions, not services.** Admin and Student are values in the JWT, enforced by role middleware. A new role (for example teacher) is a new value plus a few rules.
3. **One public door.** Only the gateway is reachable from outside. Every other service is private on the Docker network.
4. **Defence in depth.** The gateway verifies the JWT, and each service verifies it again, so a service is still safe if called directly.
5. **Stateless services.** No user data is kept in memory, so any service can run as several copies behind a load balancer.
6. **Each service owns its database.** No service writes into another's database.

## 3. Services

| Service | Port | Owns | Database |
|---|---|---|---|
| API Gateway | 3000 | Routing, JWT verification, CORS | none |
| Registration | 3001 | Creating accounts, bcrypt hashing | registration_db (`users`, writes) |
| Login | 3002 | Checking credentials, issuing JWT | registration_db (`users`, read only) |
| Content | 3003 | Courses and lessons | content_db (`courses`, `lessons`) |
| Learning | 3004 | Enrollments and progress | learning_db (`enrollments`, `progress`) |

### API Gateway
- Public routes: `POST /api/register`, `POST /api/login`, `GET /health`.
- Every other `/api` route needs a valid JWT; the gateway forwards the `Authorization` header unchanged.
- Rewrites public paths to service paths, for example `/api/courses/:id/lessons` becomes `/lessons/course/:id` on Content.
- Returns 502 if a service is down. Does not parse request bodies; it streams them through.

### Registration
- `POST /register` validates input, rejects duplicate emails (409), hashes the password with bcrypt, saves the user and returns it without the hash.

### Login
- `POST /login` compares the password to the stored hash and signs a JWT with `{ userId, email, role }`.
- Unknown email and wrong password return the same 401.

### Content
- Courses: list and read (Student, Admin); create, update, delete (Admin).
- Lessons: create, update, delete (Admin); read per course (Admin, or an enrolled Student).
- Deleting a course deletes its lessons in the same database.

### Learning
- Enrollments: enroll, list own, leave (Student); list all or by course (Admin).
- Progress: mark a lesson completed, view own progress and completion percentage (Student); view any student's progress (Admin).
- Leaving a course also deletes that student's progress for it.

## 4. Request flows

### Login
```
Client -> Gateway -> Login -> registration_db.users
                       |
                       +-- check bcrypt hash --> sign JWT --> Client
```

### Student completes a lesson (`POST /api/progress`)
```
1. Client sends JWT + { courseId, lessonId } to the Gateway
2. Gateway verifies the JWT and forwards to Learning
3. Learning checks role == student
4. Learning checks enrollment in its own database        (local query)
5. Learning asks Content: does the lesson exist and belong to the course?   (HTTP call)
6. Learning saves the progress record (unique per student + lesson)
7. 201 returned to the client
```

### Student reads lessons (`GET /api/courses/:id/lessons`)
```
1. Gateway verifies the JWT and routes to Content
2. Content checks the role; for a student it asks Learning: is this student enrolled?   (HTTP call)
3. Enrolled -> lessons sorted by order; not enrolled -> 403
```

## 5. Service-to-service calls

| Caller | Callee | Why |
|---|---|---|
| Learning | Content | Course exists (enroll); lesson exists and belongs to the course (progress); lesson count (completion %) |
| Content | Learning | Is this student enrolled? (before returning lessons) |

Calls forward the caller's JWT, use a 5 second timeout, and turn a failure into 503 instead of crashing.

**Known trade-off:** the two services depend on each other, so if one is down the other partly breaks. Alternatives, if this becomes a problem: let students read lessons of any course and enforce enrollment only for progress, or route student lesson access through Learning, which checks enrollment locally and then fetches the lesson from Content.

## 6. Authentication and authorization

- Login signs a JWT with `JWT_SECRET` (environment variable). Payload: `userId`, `email`, `role`, plus expiry.
- Clients send `Authorization: Bearer <token>`.
- Missing or invalid token: **401**. Valid token, wrong role: **403**.
- Passwords are stored only as bcrypt hashes and never returned.

| Role | Can do |
|---|---|
| Student | Read courses, enroll and leave, read lessons of enrolled courses, complete lessons, view own progress |
| Admin | Manage courses and lessons, read all lessons, view all enrollments and any student's progress |

## 7. Data model

| Database | Collection | Fields | Constraints |
|---|---|---|---|
| registration_db | users | name, email, password (hash), role | email unique |
| content_db | courses | title, description, level, createdBy | level in Beginner, Intermediate, Advanced |
| content_db | lessons | courseId, title, content, order | index on courseId |
| learning_db | enrollments | studentId, courseId, enrolledAt | unique (studentId, courseId) |
| learning_db | progress | studentId, courseId, lessonId, completed, completedAt | unique (studentId, lessonId) |

Ids that point at another service's data (courseId, studentId) are stored as plain strings; there are no cross-database foreign keys.

## 8. Repository layout

```
english-online-learning-platform/
|-- api-gateway/          src/, package.json, Dockerfile, .env.example, README.md
|-- registration-service/ src/, package.json, Dockerfile, .env.example, README.md
|-- login-service/        src/, package.json, Dockerfile, .env.example, README.md
|-- content-service/      src/, package.json, Dockerfile, .env.example, README.md
|-- learning-service/     src/, package.json, Dockerfile, .env.example, README.md
|-- postman/              collection with assertions
|-- docker-compose.yml
|-- .env.example
`-- README.md
```

## 9. Configuration

| Variable | Used by | Example |
|---|---|---|
| `PORT` | all | 3003 |
| `JWT_SECRET` | all | long random string, same in every service |
| `MONGO_URI` | all except gateway | `mongodb+srv://USER:PASS@cluster.mongodb.net/content_db` |
| `CONTENT_SERVICE_URL` | gateway, learning | `http://content-service:3003` |
| `LEARNING_SERVICE_URL` | gateway, content | `http://learning-service:3004` |
| `REGISTRATION_SERVICE_URL`, `LOGIN_SERVICE_URL` | gateway | `http://registration-service:3001` |
| `CORS_ORIGIN` | gateway | frontend origin |

Secrets live in `.env` files that are never committed to Git.

## 10. Deployment

```
Internet --> :3000 --> [ EC2 server: Docker network "elp" ]
                          gateway -> registration, login, content, learning
                                          |
                                          v
                                   MongoDB Atlas (cloud)
```

- One Docker Compose file runs the five services; only port 3000 is published.
- Atlas replaces a database container. The EC2 server's IP is added to Atlas network access.
- Security group: port 22 from the owner's IP only, port 3000 (later 443 through HTTPS) from anywhere.

## 11. Scaling notes

- Services are stateless, so more copies can run behind a load balancer when needed.
- The database is the first real limit; Atlas can be upgraded without code changes.
- Content and Learning are clean boundaries, so either can be split further (for example pulling progress out of Learning) if traffic demands it.

## 12. Phase 2: production hardening

Done after the base backend passes all tests: HTTPS and a domain, restricting admin signup, rate limiting, stronger passwords and lockout, refresh tokens, email verification and password reset, media in object storage, Atlas backups, logs and uptime alerts, Helmet and strict CORS, automated tests and CI. The full list with reasons is in the requirements specification, section 10.

## 13. Known limitations

- Content and Learning call each other (section 5).
- Login reads the users collection owned by Registration.
- Services call each other synchronously; there is no message queue.
- Admin self-registration is open until hardening is done.
