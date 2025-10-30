# React CI/CD with GitHub Actions Demo

## Objective

This repository demonstrates how to set up a complete **CI/CD pipeline for React applications** using **GitHub Actions**. The pipeline automates the entire workflow from code push to deployment, including:

- **Dependency Caching**: Speed up builds by caching npm/yarn dependencies
- **Unit Testing**: Automatically run tests on every push and pull request
- **Application Build**: Compile and build the React app
- **Artifact Publishing**: Generate and store build artifacts
- **Docker Image Publishing**: Optional Docker image creation and pushing to registries

This setup enables continuous integration and continuous deployment practices, ensuring code quality and faster iteration cycles.

---

## Table of Contents

1. [Sample Workflow YAML](#sample-workflow-yaml)
2. [Setup Guide](#setup-guide)
3. [Common Workflow Steps and Triggers](#common-workflow-steps-and-triggers)
4. [Resource Links](#resource-links)

---

## Sample Workflow YAML

Below is a comprehensive GitHub Actions workflow file for a React application. Create this file at `.github/workflows/ci-cd.yml` in your repository.

```yaml
name: React CI/CD Pipeline

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
      - develop

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    
    steps:
      # Step 1: Check out the repository code
      - name: Checkout code
        uses: actions/checkout@v4
      
      # Step 2: Set up Node.js
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      
      # Step 3: Cache dependencies for faster builds
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      # Step 4: Install dependencies
      - name: Install dependencies
        run: npm ci
      
      # Step 5: Run linting (optional)
      - name: Run ESLint
        run: npm run lint --if-present
        continue-on-error: true
      
      # Step 6: Run unit tests
      - name: Run tests
        run: npm test -- --coverage --watchAll=false
      
      # Step 7: Build the React app
      - name: Build application
        run: npm run build
      
      # Step 8: Upload build artifacts
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        if: success()
        with:
          name: build-${{ matrix.node-version }}
          path: build/
          retention-days: 30
  
  # Optional: Publish Docker image
  publish-docker:
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: false  # Set to true with proper credentials for actual push
          tags: my-registry/react-app:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Workflow File Breakdown

| Section | Purpose |
|---------|----------|
| `name` | Identifies the workflow in GitHub Actions UI |
| `on` | Defines triggers (push, pull_request, schedule, etc.) |
| `jobs` | Contains multiple jobs that run in parallel or sequentially |
| `runs-on` | Specifies the runner environment (ubuntu-latest, windows-latest, etc.) |
| `steps` | Individual tasks within a job |
| `uses` | References external GitHub Actions |
| `run` | Executes shell commands |

---

## Setup Guide

### 1. Enable GitHub Actions

- Navigate to your repository settings
- Go to **Settings** → **Actions** → **General**
- Ensure **Actions is enabled** under "Policies"
- Configure runner settings as needed

### 2. Create Workflow File

**Location**: `.github/workflows/ci-cd.yml`

Steps:

1. In your repository, create the directory structure if it doesn't exist:
   ```bash
   mkdir -p .github/workflows
   ```

2. Create `ci-cd.yml` file with the workflow content above

3. Commit and push to your repository:
   ```bash
   git add .github/workflows/ci-cd.yml
   git commit -m "Add CI/CD workflow with GitHub Actions"
   git push origin main
   ```

### 3. Monitor Workflow Execution

- Go to your repository → **Actions** tab
- Click on a workflow run to see detailed logs
- Check **Artifacts** section to download build outputs

### 4. Required Files in Your React Project

Ensure your React project has:

- `package.json` with scripts:
  ```json
  {
    "scripts": {
      "test": "react-scripts test",
      "build": "react-scripts build",
      "lint": "eslint src"
    }
  }
  ```

- `Dockerfile` (optional, for Docker publishing):
  ```dockerfile
  FROM node:18-alpine AS builder
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci
  COPY . .
  RUN npm run build
  
  FROM node:18-alpine
  WORKDIR /app
  RUN npm install -g serve
  COPY --from=builder /app/build ./build
  EXPOSE 3000
  CMD ["serve", "-s", "build", "-l", "3000"]
  ```

---

## Common Workflow Steps and Triggers

### Workflow Triggers

| Trigger | When It Runs | Example Use Case |
|---------|--------------|------------------|
| `push` | When code is pushed to specified branches | Run tests on every commit |
| `pull_request` | When a PR is created/updated | Validate PR before merging |
| `schedule` | On a cron schedule | Nightly builds or dependency updates |
| `workflow_dispatch` | Manual trigger from GitHub UI | Deploy on demand |
| `release` | When a release is published | Build and publish on version release |

### Common Workflow Steps

#### 1. **Checkout Code**
```yaml
- name: Checkout code
  uses: actions/checkout@v4
```
Retrieve the repository code for the workflow.

#### 2. **Setup Node.js**
```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20.x'
```
Prepare the Node.js runtime environment.

#### 3. **Cache Dependencies**
```yaml
- name: Cache npm dependencies
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```
Speed up builds by caching dependencies.

#### 4. **Install Dependencies**
```yaml
- name: Install dependencies
  run: npm ci
```
Use `npm ci` (clean install) instead of `npm install` in CI environments.

#### 5. **Run Tests**
```yaml
- name: Run tests
  run: npm test -- --coverage --watchAll=false
```
Execute unit tests with coverage reporting.

#### 6. **Build Application**
```yaml
- name: Build
  run: npm run build
```
Compile the React application for production.

#### 7. **Upload Artifacts**
```yaml
- name: Upload artifacts
  uses: actions/upload-artifact@v3
  with:
    name: build-artifacts
    path: build/
    retention-days: 30
```
Store build outputs for deployment or download.

#### 8. **Publish to Docker Registry**
```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: registry/my-app:latest
```
Create and push Docker images.

### Environment Variables

Set environment variables in workflow:

```yaml
env:
  NODE_ENV: production
  CI: true

jobs:
  build:
    steps:
      - run: echo $NODE_ENV
```

### Conditional Execution

Run steps based on conditions:

```yaml
- name: Deploy to production
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: npm run deploy
```

---

## Resource Links

### GitHub Actions Documentation
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Quick Start](https://docs.github.com/en/actions/quickstart)
- [Workflow Syntax for GitHub Actions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)

### React & Node.js
- [React Documentation](https://react.dev)
- [Create React App Documentation](https://create-react-app.dev)
- [Node.js Official Site](https://nodejs.org)
- [npm Documentation](https://docs.npmjs.com)

### CI/CD Best Practices
- [CI/CD Best Practices](https://docs.github.com/en/actions/guides)
- [GitHub Actions Tutorial](https://docs.github.com/en/actions/learn-github-actions)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Semantic Versioning](https://semver.org)

### Related Tools & Services
- [Docker Hub](https://hub.docker.com)
- [GitHub Container Registry (GHCR)](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Node Actions](https://github.com/actions/setup-node)
- [Codecov for Coverage Reports](https://codecov.io)

### Useful GitHub Actions
- [actions/checkout](https://github.com/actions/checkout)
- [actions/setup-node](https://github.com/actions/setup-node)
- [actions/cache](https://github.com/actions/cache)
- [actions/upload-artifact](https://github.com/actions/upload-artifact)
- [docker/build-push-action](https://github.com/docker/build-push-action)
- [actions/deploy-pages](https://github.com/actions/deploy-pages)

---

## Getting Started

1. **Clone this repository** or use it as a template
2. **Customize the workflow** in `.github/workflows/ci-cd.yml` for your React project
3. **Push to GitHub** and watch the Actions tab for workflow execution
4. **Monitor and optimize** based on build times and test results

## License

MIT License - Feel free to use this as a template for your projects.

---

**Happy Building! 🚀**
