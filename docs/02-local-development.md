# 02 — Local Development Guide

> **Navigation:** [← Project Overview](01-project-overview.md) | [Documentation Hub](index.md) | **Next →** [Docker Guide](03-docker-guide.md)

---

## Overview

This guide walks you through running the Go web application locally on your machine. No cloud services, Docker, or Kubernetes are needed for this step.

---

## Prerequisites

You only need one tool installed:

- **Go 1.21 or higher** — [Download from go.dev](https://go.dev/dl/)

Verify your installation:

```bash
go version
# Expected output: go version go1.22.x <your-os/arch>
```

---

## Step 1 — Clone the Repository

```bash
git clone https://github.com/Sanket006/golang-eks-gitops-pipeline.git
cd golang-eks-gitops-pipeline
```

---

## Step 2 — Run the Application

Start the HTTP server directly from the project root:

```bash
go run main.go
```

You should see the server start silently (no output is printed on success). The server is now listening on **port 8080**.

---

## Step 3 — Access the Application

Open your browser and navigate to any of the four available routes:

| URL | Page |
|---|---|
| `http://localhost:8080/home` | Home / Landing page |
| `http://localhost:8080/courses` | Courses listing |
| `http://localhost:8080/about` | About page |
| `http://localhost:8080/contact` | Contact page |

> **Tip:** If port 8080 is already in use on your machine, another process is occupying it. Stop that process or modify `main.go` to use a different port number.

---

## Step 4 — Run the Unit Tests

The project includes a unit test in `main_test.go` that verifies the `/home` route returns an HTTP `200 OK` response with the correct `Content-Type` header.

**Run all tests:**

```bash
go test ./...
```

**Run with verbose output** (shows individual test names and results):

```bash
go test -v ./...
```

**Expected output:**

```
=== RUN   TestMain
--- PASS: TestMain (0.00s)
PASS
ok      github.com/Sanket006/golang-eks-gitops-pipeline   0.012s
```

> A passing test suite is a required gate in the [GitHub Actions CI pipeline](08-github-actions-cicd.md). If tests fail locally, the pipeline will also fail.

---

## Project Code Walkthrough

### `main.go`

The application entry point registers four HTTP handlers and starts the server:

```go
http.HandleFunc("/home",    homePage)
http.HandleFunc("/courses", coursePage)
http.HandleFunc("/about",   aboutPage)
http.HandleFunc("/contact", contactPage)

http.ListenAndServe("0.0.0.0:8080", nil)
```

Each handler uses `http.ServeFile` to read and return the corresponding HTML file from the `static/` directory. The server binds to `0.0.0.0` (all network interfaces) so it works both locally and inside a Docker container.

### `main_test.go`

The test creates an in-memory HTTP request to `/home` using Go's `net/http/httptest` package and checks:
1. The response status code is `200 OK`
2. The `Content-Type` header is `text/html; charset=utf-8`

This validates that the handler correctly serves the HTML file without any file system issues.

### `static/`

Contains the four HTML pages (`home.html`, `courses.html`, `about.html`, `contact.html`) and an `images/` subdirectory for static assets.

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| `bind: address already in use` | Port 8080 is occupied | Stop the other process or change the port |
| `open static/home.html: no such file` | Running from wrong directory | Make sure you're in the project root (`cd golang-eks-gitops-pipeline`) |
| `go: command not found` | Go not installed | [Install Go](https://go.dev/dl/) |

---

## What's Next?

Now that the application runs locally, the next step is to **containerize** it with Docker so it can be deployed anywhere consistently.

- **Next:** [03 — Docker Guide](03-docker-guide.md)
- **Skip ahead:** [04 — AWS Prerequisites](04-aws-prerequisites.md)

---

*[← Project Overview](01-project-overview.md) | [Documentation Hub](index.md) | [Next: Docker Guide →](03-docker-guide.md)*
