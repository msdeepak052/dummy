# Docker

<img width="745" height="1115" alt="image" src="https://github.com/user-attachments/assets/fec16585-d9eb-425a-ab60-4ecc7d9443e0" />

<img width="745" height="1115" alt="image" src="https://github.com/user-attachments/assets/608e433d-7343-4ba9-b5e1-c31c1c609d94" />


<img width="730" height="1105" alt="image" src="https://github.com/user-attachments/assets/37ff1e75-f26f-40c4-937a-8e820cc84bbc" />

<img width="930" height="1105" alt="image" src="https://github.com/user-attachments/assets/d3af5474-b4ce-4767-bb98-a9f1f759d483" />



Here are 10 good ones to practice.

---

# 1. Python Flask

```dockerfile
# Build
FROM python:3.12-slim AS builder

WORKDIR /app

COPY requirements.txt .

RUN python -m venv /venv && \
    /venv/bin/pip install --no-cache-dir -r requirements.txt


# Runtime
FROM python:3.12-slim

WORKDIR /app

COPY --from=builder /venv /venv
COPY app.py .

ENV PATH="/venv/bin:$PATH"

RUN useradd -r appuser
USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

**Practices:** multi-stage, dependency caching, non-root, slim image.

---

# 2. Python FastAPI

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app

COPY requirements.txt .

RUN python -m venv /venv && \
    /venv/bin/pip install --no-cache-dir -r requirements.txt


FROM python:3.12-slim

WORKDIR /app

COPY --from=builder /venv /venv
COPY . .

ENV PATH="/venv/bin:$PATH"

RUN useradd -r appuser
USER appuser

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# 3. Java Maven Spring Boot

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

RUN useradd -r appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Key interview point

```text
Maven + JDK
    ↓
    Build
    ↓
JAR
    ↓
JRE only
```

---

# 4. Java Maven — Smaller Runtime with Alpine

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests


FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

RUN adduser -D appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Simple and very interview-friendly.

---

# 5. Go — Scratch

Go is perfect for demonstrating multi-stage builds.

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build -o server .


FROM scratch

COPY --from=builder /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

### Flow

```text
Go source
   ↓
Go compiler
   ↓
Static binary
   ↓
scratch
```

Very small final image.

---

# 6. Go — Alpine Runtime

If you don't want `scratch`:

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build -o server .


FROM alpine:3.22

WORKDIR /app

COPY --from=builder /app/server .

RUN adduser -D appuser
USER appuser

EXPOSE 8080

CMD ["./server"]
```

This is often easier to debug than `scratch`.

---

# 7. Node.js Backend

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json .
RUN npm ci

COPY . .
RUN npm run build


FROM node:22-alpine

WORKDIR /app

ENV NODE_ENV=production

COPY package*.json .
RUN npm ci --omit=dev

COPY --from=builder /app/dist ./dist

RUN adduser -D appuser
USER appuser

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### Important

```dockerfile
npm ci --omit=dev
```

means production doesn't need development dependencies.

---

# 8. React → Nginx

Very common real-world Dockerfile.

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json .
RUN npm ci

COPY . .
RUN npm run build


FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Flow

```text
React
  ↓
Node
  ↓
npm run build
  ↓
dist/
  ↓
Nginx
```

Node isn't required in the final image.

---

# 9. Rust Application

Good "etc." example.

```dockerfile
FROM rust:1.89-alpine AS builder

WORKDIR /app

RUN apk add --no-cache musl-dev

COPY Cargo.toml Cargo.lock ./
COPY src ./src

RUN cargo build --release


FROM alpine:3.22

WORKDIR /app

COPY --from=builder /app/target/release/myapp .

RUN adduser -D appuser
USER appuser

CMD ["./myapp"]
```

---

# 10. .NET Web API

Another very common DevOps interview scenario.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS builder

WORKDIR /src

COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app


FROM mcr.microsoft.com/dotnet/aspnet:9.0

WORKDIR /app

COPY --from=builder /app .

USER app

EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

# What You Should Notice Across All 10

The pattern is almost always:

```text
             BUILD STAGE
                 │
      ┌──────────┴──────────┐
      │ compiler/build tools│
      │ dependencies        │
      │ source code         │
      └──────────┬──────────┘
                 │
              artifact
                 │
                 ▼
             RUNTIME
                 │
        ┌────────┴────────┐
        │ only what's     │
        │ required to run │
        └─────────────────┘
```

### The 7 things you should explain in an interview

1. **Why multi-stage?** → Smaller production image.
2. **Why copy dependency files first?** → Docker layer caching.
3. **Why non-root?** → Security.
4. **Why slim/alpine/scratch?** → Reduce attack surface and image size.
5. **Why `npm ci` / `mvn dependency:go-offline` / `go mod download`?** → Reproducible and cache-friendly dependency installation.
6. **Why don't runtime images contain compilers?** → Build tools aren't required to run the application.
7. **Why `.dockerignore`?** → Keep unnecessary files out of the build context.
