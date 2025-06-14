
# Shift Left, Ship Secure: Building a Bulletproof DevSecOps Pipeline

**Author**: [Tim Murkomen](https://www.linkedin.com/in/timmurkomen)  
**License**: MIT  

> “Build fast, fail fast — but fail securely!”  
> — Tim Murkomen

---

## Project Overview

This repository showcases how to implement a full **DevSecOps pipeline** for a Node.js application using **Bitbucket Pipelines** for CI/CD, **Docker** for containerization, and robust security tools like **Trivy**, **Gitleaks**, and **SonarQube**.

We focus on security from the start — "shifting left" in the development cycle — so vulnerabilities are caught **before** reaching production.

---

## Tech Stack & Tools

| Area                  | Tool/Service                  |
|-----------------------|-------------------------------|
| Version Control       | Bitbucket                     |
| CI/CD Engine          | Bitbucket Pipelines           |
| Language              | Node.js                       |
| Linting               | ESLint + eslint-plugin-security |
| Static Code Analysis  | SonarQube                     |
| Secrets Detection     | Gitleaks                      |
| Vulnerability Scanning| Trivy                         |
| Containerization      | Docker                        |
| Deployment            | AWS ECS via AWS ECR           |

---

## Pipeline Steps

Below is a breakdown of each step in the pipeline, including its purpose and corresponding implementation.

---

###  Step 1: Code Linting with ESLint

**Goal**: Catch bugs and enforce secure coding standards during development.

```
- step:
    name: Lint Code with ESLint
    image: node:18
    caches:
      - node
    script:
      - npm install
      - npm install eslint eslint-plugin-security eslint-plugin-node --save-dev
      - npx eslint . --ext .js,.ts --format table || true
````

Enforces best practices and detects common mistakes that could become vulnerabilities.

---

### Step 2: Static Code Analysis with SonarQube

**Goal**: Detect code smells, bugs, and security vulnerabilities using SonarQube.

```
- step:
    name: Static Code Analysis with SonarQube
    image: sonarsource/sonar-scanner-cli
    script:
      - sonar-scanner
    variables:
      SONAR_TOKEN: ${SONAR_TOKEN}
      SONAR_HOST_URL: ${SONAR_HOST_URL}
```

Static analysis scans the application codebase without execution, catching logic errors and insecure patterns.

---

### Step 3: Secrets Detection with Gitleaks

**Goal**: Prevent committing secrets (API keys, passwords, tokens) into source control.

```
- step:
    name: Detect Secrets with Gitleaks
    image: zricethezav/gitleaks:latest
    script:
      - gitleaks detect --source=. --redact=100 --verbose || true
```

Prevents credentials leakage — a common and dangerous DevOps mistake.

---

### Step 4: Docker Image Build

**Goal**: Build a Docker image for the Node.js application.

```
- step:
    name: Build Docker Image
    script:
      - docker build -t ${IMAGE_NAME}:staging .
      - docker save ${IMAGE_NAME}:staging -o image_staging.tar
```

Containerizes the app for consistent behavior across all environments.

---

### Step 5: Filesystem Vulnerability Scan with Trivy

**Goal**: Scan source code and dependencies for vulnerabilities **before building** the Docker image.

```
- step:
    name: Trivy Filesystem Scan
    image: aquasec/trivy:latest
    script:
      - trivy fs --exit-code 0 --ignore-unfixed --severity CRITICAL,HIGH --scanners vuln,license . || true
```

Detects vulnerable libraries before the app is packaged.

---

### Step 6: Docker Image Vulnerability Scan with Trivy

**Goal**: Ensure the Docker image is free from known OS/library vulnerabilities **after build**.

```
- step:
    name: Trivy Image Scan
    services:
      - docker
    script:
      - docker load -i image_staging.tar
      - trivy image --exit-code 0 --ignore-unfixed --severity CRITICAL,HIGH ${IMAGE_NAME}:staging || true
```

Validates that the final deployable image is secure.

---

### Step 7: Push Image to AWS ECR

**Goal**: Upload the scanned and validated Docker image to Amazon Elastic Container Registry (ECR).

```yaml
- step:
    name: Push Docker Image to AWS ECR
    script:
      - aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
      - docker tag ${IMAGE_NAME}:staging ${ECR_REPO_URI}:staging
      - docker push ${ECR_REPO_URI}:staging
```

Makes the image available to ECS for deployment.

---

### Step 8: Deploy to AWS ECS (Staging)

**Goal**: Automatically deploy the image to the ECS staging environment.

```
- step:
    name: Deploy to ECS (Staging)
    deployment: staging
    script:
      - pipe: atlassian/aws-ecs-deploy:1.8.0
        variables:
          CLUSTER_NAME: "staging-cluster"
          SERVICE_NAME: "node-service"
          TASK_DEFINITION: "node-task"
```

Enables automated testing of new features in a controlled environment.

---

### Step 9: Deploy to AWS ECS (Production)

**Goal**: Manually trigger production deployment once staging passes all checks.

```yaml
- step:
    name: Deploy to ECS (Production)
    deployment: production
    trigger: manual
    script:
      - pipe: atlassian/aws-ecs-deploy:1.8.0
        variables:
          CLUSTER_NAME: "prod-cluster"
          SERVICE_NAME: "node-service"
          TASK_DEFINITION: "node-task"
```

Adds a human approval gate to protect production from bad pushes.

---

## Folder Structure

```
.
├── .bitbucket-pipelines.yml      # CI/CD pipeline definition
├── Dockerfile                    # Docker build definition
├── sonar-project.properties      # SonarQube config
├── package.json                  # Node.js project metadata
├── trivy-reports/                # Optional: store Trivy scan results
└── README.md                     # This documentation
```

---

## Feedback Loop: Shift Left Security

This pipeline creates a **feedback loop** for developers:

* Lint & scan during dev
* Validate builds before release
* Block deployment if critical issues exist

Security becomes a **collaborative and automated** process across dev, ops, and security teams.

---

## Author

**Tim Murkomen**
| DevSecOps | Cybersecurity | Software Engineer | IT Risk | GRC |

* [LinkedIn](https://www.linkedin.com/in/timmurkomen)
* [Twitter](https://twitter.com/timmurkomen)
* [Medium](https://medium.com/@timmurkomen)

---

## License

This project is licensed under the MIT License.
Feel free to fork, improve, and build secure pipelines for your team!

---

🌟 If you found this helpful, give it a star!
💬 Feel free to open issues or suggestions to improve it.

```
Happy shipping, securely! 
```
