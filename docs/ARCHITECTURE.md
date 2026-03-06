# CI/CD Pipeline Architecture

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                        GitHub                           │
│   ┌─────────────┐      ┌─────────────────────────────┐  │
│   │  main branch│      │     develop branch          │  │
│   └──────┬──────┘      └────────────┬────────────────┘  │
└──────────┼──────────────────────────┼───────────────────┘
           │ webhook trigger          │ webhook trigger
           ▼                          ▼
┌──────────────────────────────────────────────────────────┐
│                  Jenkins Master (EC2)                    │
│         Multibranch Pipeline Job                        │
│         Scans both branches                             │
└──────────────────┬───────────────────────────────────────┘
                   │ delegates build
                   ▼
┌──────────────────────────────────────────────────────────┐
│              Jenkins Agent (EC2)                        │
│         label: maven-agent                              │
│   Stage 1: Checkout                                     │
│   Stage 2: mvn clean test                               │
│   Stage 3: OWASP Security Scan                         │
│   Stage 4: mvn package + archive                       │
│   Stage 5: SCP deploy (main only) ──────────────────┐  │
└─────────────────────────────────────────────────────┼──┘
                                                      │ SSH/SCP
                                                      ▼
                                        ┌─────────────────────┐
                                        │  App Server (EC2)   │
                                        │  Runs app.jar       │
                                        │  Port 8080          │
                                        └─────────────────────┘
```

## Pipeline Stages

| Stage | Description |
|-------|-------------|
| 1. Checkout | Clone repository from GitHub |
| 2. Build & Test | `mvn clean test` with JUnit reporting |
| 3. Security Scan | OWASP Dependency Check (CVSS ≥ 7 fails build) |
| 4. Package | `mvn package` and archive JAR artifact |
| 5. Deploy | SCP to App Server and restart (main branch only) |
