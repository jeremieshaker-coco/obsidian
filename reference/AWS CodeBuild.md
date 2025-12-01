---
tags:
  - aws
  - ci-cd
  - deployment
---
# AWS CodeBuild

AWS CodeBuild is used for the CD (Continuous Deployment) part of the pipeline, triggered by [[GitHub Actions]]. It is responsible for building Docker images, pushing them to Amazon ECR, and deploying services to the AWS EKS cluster.

The build process is defined in [[buildspec.yml]] files.

It's important to note that while some documentation may refer to AWS CodeDeploy, [[AWS CodeBuild]] is the primary tool used for automated service deployments in `coco-services` and `delivery-platform`. See [[Deployment Process]] for more context.

See also: [[Software Automation and CI-CD]]

## CodeBuild Usage by Codebase

-   [[coco-services Automation]]
-   [[delivery-platform Automation]]
