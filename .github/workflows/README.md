# Tekton Pipelines

- [Overview](#overview)
  - [Layout](#layout)
  - [Common Workflow](#common-workflow)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Pipeline Run Templates](#pipeline-run-templates)
  - [**build-push**](#buildah-build-push)
  - [**maven-build**](#maven-build)
  - [**codeql-scan**](#codeql-scan)
  - [**sonar-scan-mvn**](#sonar-scan)
  - [**sonar-scan-mvn**](#sonar-scan)
  - [**trivy-scan**](#trivy-scan)
  - [**owasp-scan**](#owasp-scan)
- [How It Works](#how-it-works)

## Overview



The `./tekton.sh -i` argument sources the `.env` file at the root of the repository. Variables referenced by path are added as files to Kubernetes secrets.

### Layout

All templates are stored in the .github folder. 

```diff
./github
|-- workflows
    |-- README.md
    |-- build-push.yaml
    |-- codeql.yml
    |-- full-workflow.yaml
    |-- helm-build-deploy.yaml
    |-- oc-build-deploy.yaml
    |-- owasp-scan.yml
    |-- pre-commit-check.yaml
    |-- release.yaml
    |-- sonar-scanner-mvn.yml
    |-- sonar-scanner.yml
    |-- trivy.yaml
    `-- version.yml
```

## How to Use

1. Fork this repository to make use of all the templates.

2. Modify the `triggers` and `env` section of each of the templates and provide values that match your environment.

```yaml
# These variables tell the worflow what and where steps should run against. 
# For example, if I am building a Docker image, I can set the working directory to the directory where the Dockerfile exists.
env:
  NAME: nginx-web
  IMAGE: gregnrobinson/bcgov-nginx-demo
  WORKDIR: ./demo/nginx

# Set triggers to tell the workflow when it should run...
on:
  push:
    branches:
    - main
    paths-ignore:
    - 'README.md'
    - '.pre-commit-config.yaml'
    - './github/workflows/pre-commit-check.yaml'
  workflow_dispatch:

# A schedule can be defined using cron format. 
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron:  '30 5,17 * * *'

    # Layout of cron schedule.  'minute hour day(month) month day(week)'
    # Schedule option to review code at rest for possible net-new threats/CVE's
    # List of Cron Schedule Examples can be found at https://crontab.guru/examples.html
    # Top of Every Hour ie: 17:00:00. '0 * * * *'
    # Midnight Daily ie: 00:00:00. '0 0 * * *'
    # 12AM UTC --> 8PM EST. '0 0 * * *'
    # Midnight Friday. '0 0 * * FRI'
    # Once a week at midnight Sunday. '0 0 * * 0'
    # First day of the month at midnight. '0 0 1 * *'
    # Every Quarter. '0 0 1 */3 *'
    # Every 6 months. '0 0 1 */6 *'
    # Every Year. '0 0 1 1 *'
    #- cron: '0 0 * * *'
  ...
```

### Full Workflow Example

[Pipeline Run](https://github.com/bcgov/security-pipeline-templates/actions/runs/1492508528)

![image](https://user-images.githubusercontent.com/26353407/142968369-6c62a7c5-46f2-423b-bf90-d5ba544d672a.png)


## Reference

[Workflow Triggers](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows)