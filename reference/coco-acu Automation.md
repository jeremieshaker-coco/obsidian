---
tags:
  - automation
  - ci-cd
  - github-actions
  - docker
  - ros
---
# coco-acu Automation

This document details the CI/CD and automation setup for the `coco-acu` codebase.

See also: [[Software Automation and CI-CD]]

## GitHub Actions

The `coco-acu` repository uses two main [[GitHub Actions]] workflows:

-   **`develop.yml`**: This is the CI pipeline. It runs on pushes to `develop` and on pull requests. It performs:
    -   Linting (`clang-format`)
    -   Unit tests
    -   Docker builds
    -   Pull request title validation
    -   Schema change checks

-   **`other.yml`**: This is the CD pipeline. It triggers on new tags or GitHub releases. It performs:
    -   Builds for `jetpack4` and `jetpack5` targets
    -   Staging tests
    -   Creation of deployment artifacts for AWS IoT Greengrass
    -   Sends a Slack notification upon completion

## Docker

The `Dockerfile` in this repository is a multi-stage build that creates a complete ROS (Robot Operating System) environment. It supports cross-compilation for `arm64` and `amd64`.

The `docker-compose.yml` is configured to run the robot's software stack with host networking and privileged access to allow the container to interface with hardware.
