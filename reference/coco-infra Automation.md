---
tags:
  - automation
  - ci-cd
  - github-actions
  - terraform
---
# coco-infra Automation

This document details the automation setup for the `coco-infra` codebase.

See also: [[Software Automation and CI-CD]]

## GitHub Actions

-   **`tag-modules.yml`**: This workflow automatically creates a new git tag and GitHub release when changes are made to the Terraform modules in the `modules/` directory.
