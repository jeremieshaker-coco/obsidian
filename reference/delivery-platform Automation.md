---
tags:
  - automation
  - ci-cd
  - github-actions
  - aws-codebuild
  - docker
  - typescript
  - frontend
---
# delivery-platform Automation

This document details the CI/CD and automation setup for the `delivery-platform` codebase, which contains [[Frontend Applications]] and [[TypeScript Backend Services]].

See also: [[Software Automation and CI-CD]]

## GitHub Actions

-   **`web-ci.yml` & `mobile-ci.yml`**: These are the CI pipelines for the web and mobile applications. They run builds, linting, and tests for the applications that have changed within the monorepo. They use Playwright for end-to-end tests.
-   **`deploy.yaml`**: A manually triggered workflow for deploying services. A developer selects the application and environment, and the workflow uses Helm to deploy the corresponding Docker container to AWS EKS. It also sends Datadog and Slack notifications.

## AWS CodeBuild and buildspec.yml

The `buildspec.yml` in this repository is used by [[AWS CodeBuild]] to build and deploy the Node.js backend services. It uses `docker buildx` and `turbo prune` to efficiently build a service from the monorepo, push it to ECR, and deploy it to EKS with Helm.

## Docker

-   **`service.Dockerfile`**: A multi-stage Dockerfile optimized for building Node.js services from the monorepo.
-   **`docker-compose.yml`**: Provides a local development environment with PostgreSQL, RabbitMQ, and Redis.
