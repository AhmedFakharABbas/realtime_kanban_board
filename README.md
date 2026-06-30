# Realtime Kanban Board
 
A collaborative task/Kanban board API where task changes (create, update, delete) make it possible for connected clients to receive live updates over WebSockets — so every user sees changes from teammates instantly.
 
## ✨ Features
 
- 🔐 **JWT authentication** — register/login, with bcrypt-hashed passwords
- 🛡️ **Role-based access control (RBAC)** — e.g. only `Admin` users can delete tasks
- 📋 **Task management** — create, read, update, and delete tasks
- 📍 **Location-aware tasks** — tasks are stored with geographic coordinates (PostGIS) and can be queried by proximity (`/tasks/nearby`)
- 🔄 **Realtime updates** — a WebSocket hub broadcasts incoming messages to all connected clients
- 🌐 **CORS-enabled** for a separate frontend client (default: `http://localhost:4200`)
## 🏗️ Tech Stack
 
| Layer        | Technology |
|--------------|------------|
| Backend      | Go, [Gin](https://github.com/gin-gonic/gin) |
| Database     | PostgreSQL + PostGIS (geography/location queries), via [sqlx](https://github.com/jmoiron/sqlx) |
| Auth         | JWT ([golang-jwt](https://github.com/golang-jwt/jwt)), bcrypt password hashing |
| Realtime     | WebSockets ([gorilla/websocket](https://github.com/gorilla/websocket)) |
| Frontend     | Single-page app served from `http://localhost:4200` (e.g. Angular) |
 
## 📁 Project Structure
 
```
.
├── main.go                       # Entry point, routes, CORS, server bootstrap
├── internal/
│   ├── db/                       # Database connection (sqlx + Postgres)
│   ├── handlers/                 # HTTP & WebSocket handlers (auth, tasks, ws)
│   ├── middleware/                # JWT auth + RBAC middleware
│   └── models/                    # Data models (User, Task)
├── go.mod / go.sum
└── README.md
```
 
## 🚀 Getting Started
 
### Prerequisites
 
- [Go](https://go.dev/) 1.24+
- [PostgreSQL](https://www.postgresql.org/) with the [PostGIS](https://postgis.net/) extension enabled
- A frontend client running on `http://localhost:4200` (or update the CORS config in `main.go` to match yours)
### 1. Set up the database
 
Create a database (default expected name: `tasksdb`) and enable PostGIS:
 
```sql
CREATE DATABASE tasksdb;
\c tasksdb
CREATE EXTENSION IF NOT EXISTS postgis;
```
 
Create the required tables, e.g.:
 
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'User'
);
 
CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'todo',
    created_by INTEGER REFERENCES users(id),
    location GEOGRAPHY(Point, 4326),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```
 
### 2. Configure the connection string
 
The Postgres connection string currently lives in `main.go`:
 
```go
connStr := "postgres://postgres:hawk@localhost:5432/tasksdb?sslmode=disable"
```
 
> ⚠️ Move this (and the JWT secret in `middleware`/`handlers`) into environment variables before deploying anywhere beyond your local machine.
 
### 3. Run the server
 
```bash
go mod download
go run main.go
```
 
The server starts on port `9070` by default, or whatever is set in the `PORT` environment variable.
 
### 4. Connect a frontend
 
Point your frontend app (expected at `http://localhost:4200`) to:
- REST API: `http://localhost:9070`
- WebSocket: `ws://localhost:9070/ws`
## 📡 API Reference
 
### Public routes
 
| Method | Endpoint    | Description           |
|--------|-------------|------------------------|
| POST   | `/register` | Create a new user      |
| POST   | `/login`    | Authenticate and receive a JWT |
| GET    | `/ws`       | Upgrade to a WebSocket connection for realtime updates |
 
### Protected routes (require `Authorization: Bearer <token>`)
 
| Method | Endpoint        | Description                          | Required role |
|--------|-----------------|----------------------------------------|----------------|
| GET    | `/tasks`        | List all tasks                       | Any authenticated user |
| POST   | `/tasks`        | Create a task                        | Any authenticated user |
| PUT    | `/tasks/:id`    | Update a task                        | Any authenticated user |
| DELETE | `/tasks/:id`    | Delete a task                        | `Admin` |
| GET    | `/tasks/nearby` | Find tasks within a radius (`lat`, `lng`, `radius` query params) | Any authenticated user |
 
## 🔌 How Realtime Updates Work
 
Every client that connects to `/ws` is registered in an in-memory hub. Whenever any client sends a JSON message over the socket, the server rebroadcasts it to all other connected clients — so all connected users stay in sync as tasks/cards move and change.
 
> The current implementation broadcasts any incoming WebSocket message verbatim to all clients. For production use, consider tying broadcasts to actual task mutations (create/update/delete) and authenticating WebSocket connections.
 
## 🔐 Security Notes Before Deploying
 
- Replace the hardcoded JWT secret (`your_secret_key`) with a value from an environment variable or secrets manager.
- Move the database connection string into an environment variable rather than hardcoding credentials.
- Restrict `CheckOrigin` on the WebSocket upgrader (currently allows all origins).
- Add authentication to the `/ws` endpoint if task data should not be visible to unauthenticated clients.
## 🗺️ Roadmap Ideas
 
- [ ] Scope WebSocket broadcasts to specific boards/rooms instead of all clients
- [ ] Persist board/column structure, not just flat tasks
- [ ] Refresh tokens / token expiry handling on the frontend
- [ ] Dockerize backend + Postgres/PostGIS for easier local setup
## 🤝 Contributing
 
Issues and pull requests are welcome.
 
## 📄 License
 
Add your preferred license here (e.g. MIT).
 
