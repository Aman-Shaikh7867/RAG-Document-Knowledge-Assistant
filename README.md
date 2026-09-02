# RAG System — Full Stack Retrieval-Augmented Generation

A full-stack RAG application built with Python FastAPI, Spring Boot, React, and PostgreSQL with pgvector. Users upload PDF or Word documents and ask questions — the system retrieves relevant content and generates cited answers using OpenAI GPT-4.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Phase 1 — Environment Setup](#phase-1--environment-setup)
- [Phase 2 — Document Ingestion Pipeline](#phase-2--document-ingestion-pipeline)
- [Phase 3 — Embeddings and Vector Search](#phase-3--embeddings-and-vector-search)
- [Phase 4 — Spring Boot Backend](#phase-4--spring-boot-backend)
- [Phase 5 — React Frontend](#phase-5--react-frontend)
- [Phase 6 — Docker Compose](#phase-6--docker-compose)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
- [Running Locally (without Docker)](#running-locally-without-docker)
- [Running with Docker Compose](#running-with-docker-compose)
- [Known Issues and Fixes](#known-issues-and-fixes)

---

## Architecture Overview

```
Browser (port 3000)
       │
       ▼
React Frontend (Nginx)
       │
       ▼
Spring Boot Backend (port 8080)
    │           │
    ▼           ▼
PostgreSQL   Python FastAPI AI Service (port 8000)
(users,          │
 conversations)  ▼
             PostgreSQL + pgvector
             (documents, chunks, embeddings)
```

**Request flow:**
1. User uploads a PDF/DOCX → Spring Boot → Python AI service
2. Python extracts text, chunks it, embeds with OpenAI, stores vectors in pgvector
3. User asks a question → Spring Boot → Python AI service
4. Python embeds the question, runs cosine similarity search, passes top chunks to GPT-4
5. GPT-4 returns a cited answer → saved to conversation history → returned to frontend

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, Tailwind CSS, React Router |
| Backend | Spring Boot 3.3, Spring Security, JWT, JPA |
| AI Service | Python 3.11, FastAPI, LangChain text splitters |
| LLM | OpenAI GPT-4o |
| Embeddings | OpenAI text-embedding-3-small (1536 dimensions) |
| Database | PostgreSQL 15 + pgvector extension |
| PDF extraction | pdfplumber, PyPDF2 |
| Containerisation | Docker, Docker Compose |

---

## Project Structure

```
rag-system/
├── docker-compose.yml
├── init.sql
├── .env
├── .gitignore
│
├── rag-ai-service/          # Python FastAPI
│   ├── app/
│   │   ├── main.py
│   │   ├── db/
│   │   │   └── database.py
│   │   ├── models/
│   │   │   └── document.py
│   │   ├── routers/
│   │   │   ├── documents.py
│   │   │   ├── chat.py
│   │   │   └── admin.py
│   │   └── services/
│   │       ├── extraction_service.py
│   │       ├── chunking_service.py
│   │       ├── document_service.py
│   │       ├── embedding_service.py
│   │       ├── retrieval_service.py
│   │       └── chat_service.py
│   ├── requirements.txt
│   ├── .env
│   ├── .dockerignore
│   └── Dockerfile
│
├── rag-backend/             # Spring Boot
│   ├── src/main/java/com/rag/
│   │   ├── controller/
│   │   │   ├── AuthController.java
│   │   │   ├── DocumentController.java
│   │   │   └── ChatController.java
│   │   ├── service/
│   │   │   ├── UserService.java
│   │   │   ├── AiService.java
│   │   │   └── ChatService.java
│   │   ├── model/
│   │   │   ├── User.java
│   │   │   ├── Document.java
│   │   │   ├── Conversation.java
│   │   │   └── Message.java
│   │   ├── repository/
│   │   │   ├── UserRepository.java
│   │   │   ├── DocumentRepository.java
│   │   │   ├── ConversationRepository.java
│   │   │   └── MessageRepository.java
│   │   ├── security/
│   │   │   ├── JwtUtil.java
│   │   │   ├── JwtFilter.java
│   │   │   └── SecurityConfig.java
│   │   └── exception/
│   │       └── GlobalExceptionHandler.java
│   ├── src/main/resources/
│   │   └── application.yml
│   ├── pom.xml
│   ├── .dockerignore
│   └── Dockerfile
│
└── rag-frontend/            # React
    ├── src/
    │   ├── context/
    │   │   └── AuthContext.jsx
    │   ├── services/
    │   │   └── api.js
    │   ├── hooks/
    │   │   └── useChat.js
    │   ├── components/
    │   │   ├── Layout.jsx
    │   │   ├── MessageList.jsx
    │   │   ├── ChatInput.jsx
    │   │   ├── SourceBadge.jsx
    │   │   └── ConfirmDialog.jsx
    │   ├── pages/
    │   │   ├── LoginPage.jsx
    │   │   ├── RegisterPage.jsx
    │   │   ├── ChatPage.jsx
    │   │   └── UploadPage.jsx
    │   ├── App.jsx
    │   └── main.jsx
    ├── nginx.conf
    ├── .env
    ├── .env.production
    ├── .dockerignore
    └── Dockerfile
```

---

## Prerequisites

| Tool | Version | Download |
|---|---|---|
| Python | 3.11+ | python.org |
| Java | 21 | adoptium.net |
| Node.js | 20+ | nodejs.org |
| Docker Desktop | Latest | docker.com |
| PostgreSQL | 15+ (local dev only) | postgresql.org |
| OpenAI API key | — | platform.openai.com |

---

## Phase 1 — Environment Setup

### 1. Clone / create the project

```bash
mkdir rag-system
cd rag-system
git init
```

### 2. Set up Python AI service

```bash
mkdir rag-ai-service
cd rag-ai-service
python -m venv venv

# Activate (Windows)
venv\Scripts\activate

# Activate (Mac/Linux)
source venv/bin/activate

pip install -r requirements.txt
```

**`requirements.txt`:**

```
fastapi==0.111.0
uvicorn==0.30.0
python-dotenv==1.0.1
openai==1.35.0
psycopg2-binary==2.9.9
pgvector==0.2.5
pypdf2==3.0.1
pdfplumber==0.11.0
python-docx==1.1.2
langchain-text-splitters==0.2.2
pydantic==2.7.4
sqlalchemy==2.0.31
alembic==1.13.1
python-multipart==0.0.9
boto3==1.34.0
```

**`rag-ai-service/.env`:**

```env
OPENAI_API_KEY=sk-your-key-here
DATABASE_URL=postgresql://raguser:ragpass@localhost:5432/ragdb
```

### 3. Set up PostgreSQL with pgvector

```sql
CREATE DATABASE ragdb;
CREATE USER raguser WITH PASSWORD 'ragpass';
GRANT ALL PRIVILEGES ON DATABASE ragdb TO raguser;
\c ragdb
CREATE EXTENSION vector;
```

Run `init.sql` from project root:

```bash
psql -U raguser -d ragdb -f init.sql
```

### 4. Create the Spring Boot project

Go to [start.spring.io](https://start.spring.io) with these settings:

- Project: Maven
- Language: Java
- Spring Boot: 3.3.x
- Group: `com.rag`
- Artifact: `rag-backend`
- Java: 21
- Dependencies: Spring Web, Spring Security, Spring Data JPA, PostgreSQL Driver, Lombok, Validation, Spring Boot Actuator, WebFlux

### 5. Create the React project

```bash
cd rag-system
npm create vite@latest rag-frontend -- --template react
cd rag-frontend
npm install
npm install axios react-router-dom
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

**`rag-frontend/.env`:**

```env
VITE_API_BASE_URL=http://localhost:8080/api
```

---

## Phase 2 — Document Ingestion Pipeline

The Python AI service handles file upload, text extraction, and chunking.

**Key files:**

- `app/services/extraction_service.py` — extracts text from PDF (pdfplumber + PyPDF2 fallback) and DOCX
- `app/services/chunking_service.py` — merges short pages then splits with `RecursiveCharacterTextSplitter` (chunk size 1000, overlap 150)
- `app/services/document_service.py` — orchestrates extract → chunk → embed → save
- `app/routers/documents.py` — `POST /documents/upload`, `GET /documents/`, `DELETE /documents/{id}`

**Upload a document:**

```bash
curl -X POST http://localhost:8000/documents/upload \
  -F "file=@/path/to/document.pdf" \
  -F "user_id=your-user-uuid"
```

---

## Phase 3 — Embeddings and Vector Search

**Key files:**

- `app/services/embedding_service.py` — calls OpenAI `text-embedding-3-small`, supports batch embedding
- `app/services/retrieval_service.py` — cosine similarity search using pgvector `<=>` operator, returns top 8 chunks filtered by user_id
- `app/services/chat_service.py` — rewrites query with GPT-4, retrieves chunks, builds prompt, calls GPT-4o, returns answer + sources

**RAG quality settings in `chat_service.py`:**

```python
SIMILARITY_THRESHOLD = 0.10   # low threshold suits sparse slide/PDF content
TOP_K = 8                     # retrieve more candidates
```

**Query rewriting** is enabled by default — informal questions like "why important REST?" are rewritten to well-formed queries before embedding, significantly improving retrieval accuracy.

**Ask a question directly:**

```bash
curl -X POST http://localhost:8000/chat/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the leave policy?", "user_id": "your-uuid"}'
```

---

## Phase 4 — Spring Boot Backend

Spring Boot provides JWT authentication, user management, conversation history, and proxies requests to the Python AI service.

**Key endpoints:**

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | /api/auth/register | Public | Register new user |
| POST | /api/auth/login | Public | Login, returns JWT |
| POST | /api/documents/upload | JWT | Upload and process document |
| GET | /api/documents | JWT | List user's documents |
| DELETE | /api/documents/{id} | JWT | Delete document and chunks |
| POST | /api/chat/ask | JWT | Ask a question |
| GET | /api/chat/conversations | JWT | List user's conversations |
| GET | /api/chat/conversations/{id}/messages | JWT | Get conversation messages |
| DELETE | /api/chat/conversations/{id} | JWT | Delete conversation |

**`application.yml` key settings:**

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ragdb
    username: raguser
    password: ragpass
  jpa:
    hibernate:
      ddl-auto: validate
  servlet:
    multipart:
      max-file-size: 20MB
      max-request-size: 25MB

jwt:
  secret: your-256-bit-secret-change-in-production
  expiration-ms: 86400000

ai-service:
  base-url: http://localhost:8000

management:
  endpoints:
    web:
      exposure:
        include: health
```

**Multi-user document isolation:** Every upload saves `uploaded_by = user_id`. Every vector search filters `WHERE d.uploaded_by = :user_id`. Users only ever see answers from their own documents.

---

## Phase 5 — React Frontend

A clean chat interface with sidebar navigation, conversation history, document upload, and delete confirmation dialogs.

**Pages:**

- `/login` — JWT login form
- `/register` — account creation
- `/` — chat page with message bubbles and source citations
- `/upload` — drag-and-drop file upload with document list

**Key components:**

- `Layout.jsx` — sidebar with conversation history, delete on hover
- `MessageList.jsx` — user/assistant bubbles with typing indicator
- `SourceBadge.jsx` — inline document + page number citations
- `ConfirmDialog.jsx` — reusable confirmation modal for deletions
- `useChat.js` — all chat state, optimistic UI, conversation restoration

**`rag-frontend/.env.production`** (used inside Docker):

```env
VITE_API_BASE_URL=/api
```

In production the Nginx container proxies `/api/` to Spring Boot, so no CORS issues.

---

## Phase 6 — Docker Compose

Run the entire stack with one command.

### Start everything

```bash
cd rag-system
docker compose up --build
```

First build takes 10–15 minutes. Subsequent builds use cached layers and are much faster.

### Service startup order

```
postgres (healthy)
    → rag-ai-service (healthy)
        → rag-backend (healthy)
            → rag-frontend
```

### Access the application

| Service | URL |
|---|---|
| React frontend | http://localhost:3000 |
| Spring Boot API | http://localhost:8080 |
| Python AI service | http://localhost:8000 |
| Swagger UI (AI) | http://localhost:8000/docs |
| Health check | http://localhost:8080/actuator/health |

### Useful commands

```bash
# Start in background
docker compose up --build -d

# View logs for one service
docker compose logs -f rag-ai-service
docker compose logs -f rag-backend

# Restart one service without rebuilding
docker compose restart rag-backend

# Stop everything (keeps database volume)
docker compose down

# Stop and wipe database (fresh start)
docker compose down -v

# Rebuild one service only
docker compose build rag-frontend
docker compose up -d rag-frontend

# Check container status
docker compose ps
```

---

## Environment Variables

### `rag-system/.env` (root — Docker Compose reads this)

```env
OPENAI_API_KEY=sk-your-openai-key-here
JWT_SECRET=your-long-random-secret-at-least-32-characters
```

### `rag-ai-service/.env` (local dev only)

```env
OPENAI_API_KEY=sk-your-openai-key-here
DATABASE_URL=postgresql://raguser:ragpass@localhost:5432/ragdb
```

### `rag-frontend/.env` (local dev)

```env
VITE_API_BASE_URL=http://localhost:8080/api
```

### `rag-frontend/.env.production` (Docker build)

```env
VITE_API_BASE_URL=/api
```

---

## API Reference

### Auth

**Register**
```
POST /api/auth/register
Body: { "name": "Alice", "email": "alice@test.com", "password": "secret123" }
Response: { "token": "eyJ...", "name": "Alice", "role": "USER" }
```

**Login**
```
POST /api/auth/login
Body: { "email": "alice@test.com", "password": "secret123" }
Response: { "token": "eyJ...", "name": "Alice", "role": "USER" }
```

### Documents

**Upload**
```
POST /api/documents/upload
Headers: Authorization: Bearer <token>
Body: multipart/form-data — file field
Response: { "document_id": "uuid", "filename": "doc.pdf", "chunk_count": 42 }
```

**List**
```
GET /api/documents
Headers: Authorization: Bearer <token>
Response: [{ "id": "uuid", "name": "doc.pdf", "upload_date": "..." }]
```

**Delete**
```
DELETE /api/documents/{id}
Headers: Authorization: Bearer <token>
Response: 204 No Content
```

### Chat

**Ask**
```
POST /api/chat/ask
Headers: Authorization: Bearer <token>
Body: { "question": "What is the leave policy?", "conversationId": null }
Response: {
  "answer": "Employees are entitled to 14 days...",
  "sources": [{ "document": "HR_Policy.pdf", "page": 12, "score": 0.89 }],
  "conversation_id": "uuid",
  "chunks_used": 3
}
```

**List conversations**
```
GET /api/chat/conversations
Headers: Authorization: Bearer <token>
Response: [{ "id": "uuid", "title": "What is the leave policy?", "created_at": "..." }]
```

**Get messages**
```
GET /api/chat/conversations/{id}/messages
Headers: Authorization: Bearer <token>
Response: [{ "question": "...", "answer": "...", "created_at": "..." }]
```

**Delete conversation**
```
DELETE /api/chat/conversations/{id}
Headers: Authorization: Bearer <token>
Response: 204 No Content
```

---

## Running Locally (without Docker)

Run four terminals simultaneously:

**Terminal 1 — PostgreSQL**
```bash
# Make sure PostgreSQL is running locally with ragdb created
psql -U raguser -d ragdb -c "\dt"
```

**Terminal 2 — Python AI service**
```bash
cd rag-ai-service
venv\Scripts\activate        # Windows
source venv/bin/activate     # Mac/Linux
uvicorn app.main:app --reload --port 8000
```

**Terminal 3 — Spring Boot**
```bash
cd rag-backend
./mvnw spring-boot:run
```

**Terminal 4 — React**
```bash
cd rag-frontend
npm run dev
```

Open http://localhost:5173

---

## Running with Docker Compose

```bash
cd rag-system

# Make sure .env exists with your keys
cat .env

# Build and start
docker compose up --build

# Open browser
start http://localhost:3000
```

---

## Known Issues and Fixes

### PostgreSQL version mismatch

If you see `database files are incompatible with server`:

```bash
# Wipe the old volume and restart
docker compose down -v
docker compose up --build
```

### Document upload 500 error

Spring Boot has a 1MB multipart limit by default. Add to `application.yml`:

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 20MB
      max-request-size: 25MB
```

Also ensure `AiService.java` uses `ByteArrayResource` with `ExchangeStrategies` set to 20MB buffer.

### RAG answers saying "I don't have enough information"

Three possible causes and fixes:

1. **Similarity threshold too high** — lower `SIMILARITY_THRESHOLD` to `0.10` in `chat_service.py`
2. **Short/informal questions** — query rewriting is built in; ensure `rewrite_query()` is called before retrieval
3. **Slide deck PDFs with sparse text** — increase `CHUNK_SIZE` to `1000` and enable page merging in `chunking_service.py`; re-upload the document after changing settings

### Conversation history showing wrong dates

Use calendar date comparison in `formatDate` in `Layout.jsx`:

```js
const dDate   = new Date(d.getFullYear(),   d.getMonth(),   d.getDate());
const nowDate = new Date(now.getFullYear(), now.getMonth(), now.getDate());
const diff    = Math.round((nowDate - dDate) / 86400000);
```

### Docker EOF error on Windows

Pre-pull all base images before building:

```bash
docker pull python:3.11-slim
docker pull maven:3.9.6-eclipse-temurin-21
docker pull eclipse-temurin:21-jre-jammy
docker pull node:20-alpine
docker pull nginx:alpine
docker pull pgvector/pgvector:pg17
```

---

## Database Schema

```sql
users         — id, name, email, password, role, created_at
documents     — id, name, s3_url, upload_date, uploaded_by (→ users.id)
chunks        — id, document_id (→ documents.id), chunk_text, embedding (vector 1536), page_no
conversations — id, user_id (→ users.id), created_at
messages      — id, conversation_id (→ conversations.id), question, answer, created_at
```

Vector index on chunks:

```sql
CREATE INDEX chunks_embedding_idx
    ON chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

## Security Notes

- Passwords hashed with BCrypt
- JWT tokens expire after 24 hours (configurable)
- Each user can only access their own documents and conversations — enforced at both Spring Boot (ownership check) and Python (SQL filter) layers
- Never commit `.env` files — they are in `.gitignore`
- Change `JWT_SECRET` to a strong random value before any public deployment


Aman Shaikh — [LinkedIn](www.linkedin.com/in/amanshaikh07) · [Portfolio](https://github.com/Aman-Shaikh7867)