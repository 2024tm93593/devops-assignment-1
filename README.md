# ACEest Fitness & Gym Management Service

![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)
![Flask](https://img.shields.io/badge/Flask-Web%20Framework-lightgrey.svg)
![Docker](https://img.shields.io/badge/Docker-Containerized-blue.svg)
![CI/CD](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-brightgreen.svg)

A comprehensive Flask-based gym management web service. This repository demonstrates a complete DevOps lifecycle—encompassing version control, containerization, automated testing, CI/CD pipelines via GitHub Actions and Jenkins, Kubernetes deployments, and advanced release strategies.

---

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
  - [Local Setup](#local-setup)
  - [Docker Setup](#docker-setup)
- [Testing](#testing)
- [CI/CD Pipeline](#cicd-pipeline)
  - [GitHub Actions](#github-actions)
  - [Jenkins](#jenkins)
- [Kubernetes Deployment](#kubernetes-deployment)
  - [Rolling Update](#rolling-update)
  - [Blue-Green](#blue-green)
  - [Canary](#canary)
  - [Shadow](#shadow)
  - [A/B Testing](#ab-testing)
- [API Reference](#api-reference)
- [Legacy Versions](#legacy-versions)

---

## Overview

ACEest Fitness (Version 3.2.4) is a functional fitness gym management system. Building on previous versions, this release introduces comprehensive workout logging, body metrics tracking, user authentication, a BMI calculator, automated AI program generation, and PDF report creation.

**Available Programs:**

- **Fat Loss (FL) – 3 day:** 3-day full-body fat loss with a calorie factor of 22 kcal/kg.
- **Fat Loss (FL) – 5 day:** 5-day split, higher volume fat loss with a calorie factor of 24 kcal/kg.
- **Muscle Gain (MG) – PPL:** Push/Pull/Legs hypertrophy with a calorie factor of 35 kcal/kg.
- **Beginner (BG):** 3-day simple beginner full-body with a calorie factor of 26 kcal/kg.

**Features in v3.2.4:**

- Support for system users and secure API authentication (`/login`).
- Membership expiration tracking for all clients.
- Automated generation of AI workout schedules (`/ai_program`).
- Export client data to PDF reports (`/export_pdf`).
- Web interface dynamically displaying membership details and system improvements.
- A/B variant UI support via the `AB_VARIANT` environment variable (Variant A: stable gold theme; Variant B: blue test theme).

---

## Repository Structure

```text
.
├── app.py                      # Core Flask web application
├── requirements.txt            # Python dependencies
├── Dockerfile                  # Multi-stage Docker image configuration
├── Jenkinsfile                 # Jenkins declarative pipeline (test → SonarQube → build → deploy)
├── .github/
│   └── workflows/
│       └── main.yml            # GitHub Actions CI/CD pipeline (PR to main)
├── tests/
│   └── test_app.py             # Pytest test suite (~40 tests across all endpoints)
├── k8s/
│   ├── deployment.yaml         # Standard rolling-update Kubernetes deployment
│   ├── blue-green/             # Blue-green deployment manifests and shell scripts
│   ├── canary/                 # Canary deployment manifests and shell scripts
│   ├── shadow/                 # Shadow deployment manifests and nginx mirror proxy
│   └── ab-testing/             # A/B testing manifests and nginx routing proxy
├── versions/                   # Reference files prior to Flask migration
└── README.md                   # Project documentation
```

---

## Getting Started

### Local Setup

**Prerequisites:**

- Python 3.11+
- `pip` package manager

1. **Clone the repository:**

   ```bash
   git clone https://github.com/2024tm93593/devops-assignment-1.git
   cd devops-assignment-1
   ```

2. **Create and activate a virtual environment:**

   ```bash
   python -m venv venv
   # On macOS/Linux:
   source venv/bin/activate
   # On Windows:
   venv\Scripts\activate
   ```

3. **Install dependencies:**

   ```bash
   pip install -r requirements.txt
   ```

4. **Run the application:**
   ```bash
   python app.py
   ```
   The application will be available at `http://localhost:5000`.

### Docker Setup

The Dockerfile uses a multi-stage build: a `builder` stage installs dependencies, and the final stage copies only the installed packages and application code, keeping the image lean.

1. **Build the Docker image:**

   ```bash
   docker build -t aceest-fitness-app:3.2.4 .
   ```

2. **Run the container (with persistence):**
   Client data is stored at `/app/data` inside the container. Mount a local directory to preserve data across restarts:

   ```bash
   docker run -d -p 5000:5000 --name aceest \
     -v <your-host-path>:/app/data \
     aceest-fitness-app:3.2.4
   ```

   To run Variant B of the UI:

   ```bash
   docker run -d -p 5000:5000 --name aceest \
     -e AB_VARIANT=B \
     -v <your-host-path>:/app/data \
     aceest-fitness-app:3.2.4
   ```

3. **Stop the container:**
   ```bash
   docker stop aceest
   docker rm aceest
   ```

---

## Testing

The repository uses `pytest` for unit and integration testing. The test suite covers calorie calculations, program lookups, input validation, SQLite persistence, BMI, workout tracking, progress monitoring, metrics charting, AI program generation, PDF export, and all API endpoint behaviors.

**Run tests locally:**

```bash
PYTHONPATH=. pytest tests/ -v
```

**Run with coverage report:**

```bash
PYTHONPATH=. pytest tests/ -v --cov=. --cov-report=xml
```

**Run tests inside the Docker container:**

```bash
docker run --rm aceest-fitness-app:3.2.4 pytest tests/ -v
```

---

## CI/CD Pipeline

The project implements an automated Continuous Integration and Continuous Deployment (CI/CD) pipeline across two systems: GitHub Actions for automated PR validation and DockerHub publishing, and Jenkins for SonarQube analysis and Kubernetes deployment.

### GitHub Actions

The pipeline triggers automatically on every `pull_request` targeting the `main` branch.

**Pipeline Stages (`.github/workflows/main.yml`):**

| Stage | Name | What it does |
|-------|------|--------------|
| 1 | Build & Lint | Installs dependencies, compiles `app.py`, runs `flake8` for syntax and undefined-name checks |
| 2 | Docker Image Assembly | Builds the Docker image tagged with the commit SHA and `latest` |
| 3 | Automated Testing | Executes the full `pytest` suite with coverage reporting |
| 4 | SonarCloud Analysis | Uploads coverage and source to SonarCloud for code quality and security scanning |
| 5 | Push to DockerHub | Logs in with stored secrets and pushes `chetan56881/aceest-fitness-app` tagged with the GHA run number and `latest-gha` |

**Required repository secrets:**

| Secret | Purpose |
|--------|---------|
| `SONAR_TOKEN` | SonarCloud authentication token |
| `DOCKERHUB_USERNAME` | DockerHub login |
| `DOCKERHUB_TOKEN` | DockerHub access token |

### Jenkins

Jenkins serves as an independent build and deployment server. The declarative `Jenkinsfile` pipeline supports parameterised builds and deploys to a GCP VM running Kubernetes via SSH.

**Build parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `IMAGE_VERSION` | String | Docker image version tag; defaults to the Jenkins build number if left blank |
| `DEPLOY_STRATEGY` | Choice | One of `blue-green`, `rolling`, `canary`, `shadow`, `ab-testing` |

**Pipeline Stages:**

| Stage | What it does |
|-------|--------------|
| Env Setup & Test | Creates a Python venv, installs requirements, runs `pytest` with coverage |
| SonarQube Analysis | Runs `sonar-scanner` against SonarCloud with coverage data |
| Build & Push to DockerHub | Builds with `docker buildx` and pushes `chetan56881/aceest-fitness-app:<tag>` and `:latest` |
| Deploy | SSHes into a GCP VM and executes the chosen deployment strategy on the cluster |

**Automatic rollback:** If the Deploy stage fails, the `post { failure { ... } }` block SSHes back into the VM and executes the appropriate `rollback.sh` script for the chosen strategy.

**Required Jenkins credentials:**

| Credential ID | Type | Purpose |
|--------------|------|---------|
| `dockerhub-credentials` | Username/Password | DockerHub login |
| `gcp-vm-ssh-key` | SSH private key | SSH access to the GCP deployment VM |
| `GCP_VM_IP` | Secret string | IP address of the GCP VM |
| `GCP_VM_USER` | Secret string | SSH username on the GCP VM |

---

## Kubernetes Deployment

All manifests live in `k8s/`. The application image is `chetan56881/aceest-fitness-app` on DockerHub.

### Rolling Update

`k8s/deployment.yaml` — a standard Kubernetes `Deployment` with a `RollingUpdate` strategy (`maxSurge: 1`, `maxUnavailable: 0`) and a `LoadBalancer` service. Select this via `DEPLOY_STRATEGY=rolling` in Jenkins.

### Blue-Green

`k8s/blue-green/` — two independent namespaces (`blue`, `green`) each running the application. A single `router-service` in `default` namespace points to whichever colour is currently active. A `state-configmap` tracks the active colour.

- **`switch-traffic.sh`** — updates the router EndpointSlice to point to the target namespace.
- **`rollback.sh`** — switches the router back to the previous active colour.

### Canary

`k8s/canary/` — a `stable` namespace and a `canary` namespace. The new image is deployed to canary first while stable continues serving production traffic.

- **`canary-enable.sh`** — updates the router EndpointSlice to start sending traffic to canary.
- **`promote.sh`** — promotes the canary image to stable.
- **`rollback.sh`** — removes canary from the router and restores stable-only traffic.

### Shadow

`k8s/shadow/` — a `prod` namespace serving real users and a `shadow` namespace receiving a mirrored copy of all requests (responses from shadow are discarded). Traffic mirroring is handled by an nginx proxy deployed into the `default` namespace that duplicates incoming requests.

- **`rollback.sh`** — tears down the shadow namespace and nginx proxy.

### A/B Testing

`k8s/ab-testing/` — an `a-site` namespace (Variant A, stable gold theme) and a `b-site` namespace (Variant B, blue theme, new version under test). An nginx proxy in the `default` namespace routes requests based on the cookie `ab_variant=b` or the header `X-AB-Variant: b`; all other traffic goes to Variant A.

- The `AB_VARIANT` environment variable in `app.py` controls the UI rendered by each variant's pods.
- **`rollback.sh`** — removes the b-site deployment and nginx proxy.

---

## API Reference

The service provides a REST API to interact with the gym's database.

| Method | Endpoint                 | Description                                                    | Example Payload / Query                                                               |
| ------ | ------------------------ | -------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `GET`  | `/`                      | Web interface listing gym programs and client list             | N/A                                                                                   |
| `POST` | `/login`                 | Authenticate a user against the system users table             | `{"username": "admin", "password": "admin"}`                                          |
| `GET`  | `/programs`              | Returns all available fitness programs as JSON                 | N/A                                                                                   |
| `POST` | `/client`                | Register or update a client with a program                     | `{"name": "Ravi", "program": "Fat Loss (FL) – 3 day", "height": 175, "weight": 75}`  |
| `GET`  | `/client/<name>`         | Load a client profile by name                                  | `/client/Ravi`                                                                        |
| `GET`  | `/membership/<name>`     | Query a client's membership status and end date                | `/membership/Ravi`                                                                    |
| `GET`  | `/clients`               | Returns the full client list as JSON                           | N/A                                                                                   |
| `POST` | `/workouts`              | Log a workout session and optional exercise                    | `{"client_name": "Ravi", "workout_type": "Strength", "duration_min": 60}`             |
| `GET`  | `/workouts/<name>`       | Retrieve workout history for a client                          | `/workouts/Ravi`                                                                      |
| `POST` | `/metrics`               | Save body metrics (weight, waist, bodyfat); also updates client weight | `{"client_name": "Ravi", "weight": 74, "bodyfat": 14.5}`                     |
| `GET`  | `/metrics/<name>`        | Retrieve recorded body metrics history                         | `/metrics/Ravi`                                                                       |
| `GET`  | `/metrics/chart/<name>`  | Download a generated PNG weight trend chart                    | `/metrics/chart/Ravi`                                                                 |
| `GET`  | `/bmi`                   | Evaluate BMI and return category and health risk               | `?height=180&weight=80`                                                               |
| `POST` | `/ai_program`            | Generate an AI fitness schedule based on client program and experience | `{"client_name": "Ravi", "experience_level": "beginner"}`                   |
| `GET`  | `/export_pdf/<name>`     | Export and download a PDF report containing client details     | `/export_pdf/Ravi`                                                                    |
| `POST` | `/progress`              | Save weekly adherence for a client                             | `{"client_name": "Ravi", "adherence": 85}`                                            |
| `GET`  | `/progress/<name>`       | Returns all progress entries for a client                      | `/progress/Ravi`                                                                      |
| `GET`  | `/progress/chart/<name>` | Download a generated PNG adherence progress chart              | `/progress/chart/Ravi`                                                                |
| `GET`  | `/calories`              | Calculate estimated daily calories based on weight and program | `?weight=80&program=Muscle Gain (MG) – PPL`                                           |

---

## Legacy Versions

The `versions/` directory contains legacy Tkinter scripts (e.g., `Aceestver-X.X.py`). These files represent earlier desktop iterations of the application. They are preserved for historical context and educational purposes but are no longer actively maintained or integrated into the current web service.

---

_Developed for Introduction to DevOps, BITS Pilani (S2-25)_
