---
tags:
  - automation
  - ci-cd
  - github-actions
  - aws-codebuild
  - docker
  - go
---
# coco-services Automation

This document details the CI/CD and automation setup for the `coco-services` codebase, which contains various [[Go Backend Services]].

See also: [[Software Automation and CI-CD]]

## GitHub Actions

-   **`tests.yml` & `golangci-lint.yml`**: These workflows run on pull requests and pushes to `main`. They lint the Go code and run unit tests for changed modules.
-   **`mds-e2e-test.yaml`**: Runs end-to-end tests for the `mds` and `core` services using Docker Compose to spin up a test environment.
-   **`tag-core.yml`**: Automatically creates a new git tag and GitHub release when changes are merged to the `core` module.

## AWS CodeBuild and buildspec.yml

Most of the microservices in this repository have their own `buildspec.yml` file, which is used by [[AWS CodeBuild]] for deployment. The general process is:
1.  Build the Go application into a Docker image.
2.  Push the image to Amazon ECR.
3.  Deploy the service to AWS EKS using Helm.

The builds are configured and templatized using a custom `helm-ssm` plugin to inject secrets and parameters.

## Docker

A `docker-compose.yml` file is provided for local development, which includes services like PostgreSQL, RabbitMQ, Redis, and Kafka. This is also used in the `mds-e2e-test.yaml` GitHub Actions workflow.
