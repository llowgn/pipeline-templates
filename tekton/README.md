# Tekton Kustomize

- [Overview](#overview)
  - [Layout](#layout)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Pipeline Run Templates](#pipeline-run-templates)
  - [**buildah-build-push**](#buildah-build-push)
  - [**maven-build**](#maven-build)
  - [**codeql-scan**](#codeql-scan)
  - [**sonar-scan**](#sonar-scan)
- [How It Works](#how-it-works)

## Overview

This project aims to improve the management experience with tekton pipelines. The [pipeline](https://github.com/tektoncd/pipeline) and [triggers](https://github.com/tektoncd/triggers) projects are included as a single deployment. All pipelines and tasks are included in the deployment and can be incrementally updated by running `./tekton.sh --apply`. All operations specific to manifest deployment are handled by [Kustomize](https://kustomize.io/). A `kustomization.yaml` file exists recursively in all directories under `./base.`

The project creates secrets for your docker and ssh credentials using the Kustomize [secretGenerator](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/). This allows for the git-clone and buildah Tekton tasks to interact with private repositories. I would consider setting up git and container registry credentials a foundational prerequisite for operating cicd tooling. Once Kustomize creates the secrets, they are referenced directly by name in the [Pipeline Run Templates](#pipeline-run-templates) section.

The repository is intended to configure every aspect of Tekton, from the installation manifests to the custom Tekton CRDs that manage the creation and execution of pipelines. Whenever changes are made to `./base` or `./overlays,` run `./tekton.sh --apply` to apply the changes against the current Kubernetes context. Behind the scenes, the following functions are executed.

1. **setup**: Installs [yq](https://mikefarah.gitbook.io/yq/) for parsing YAML files.
2. **sync**: Pulls the following Tekton release manifests to `./base/install`
    - pipeline
    - triggers
    - interceptors
    - dashboard
3. **credentials**: copies ssh, docker, and webhook credentials to Kustomize and creates secrets.
4. **apply**: Runs `kubectl apply -k overlays/${ENV}` to install/update Tekton and deploy Tekton CRDs.
5. **cleanup**: Cleans up Completed pipeline runs and deletes all creds.

The `./tekton.sh --apply` argument sources the `.env` file at the root of the repository. Variables referenced by path are added as files to Kubernetes secrets.

### Layout

```diff
  ./base
  ├── install
  │   ├── dashboards.yaml
  │   ├── interceptors.yaml
+ │   ├── kustomization.yaml
  │   ├── pipelines.yaml
  │   └── triggers.yaml
+ ├── kustomization.yaml
  ├── pipelines
  │   ├── buildah.yaml
+ │   ├── kustomization.yaml
  │   └── maven.yaml
  ├── tasks
  │   ├── buildah.yaml
  │   ├── git-clone.yaml
+ │   ├── kustomization.yaml
  │   └── maven-build.yaml
  └── triggers
      ├── ingress.yaml
+     ├── kustomization.yaml
      ├── rbac.yaml
      └── trigger-template.yaml
```

## Prerequisites

Note: This project has been tested on *linux/arm64*, *linux/amd64*, *linux/aarch64*, and *darwin/arm64*.

1. [kubectl](https://kubernetes.io/docs/tasks/tools/) >= 1.21.0
2. [wget](https://www.tecmint.com/install-wget-in-linux/)

## Installation

1. Clone the repository. (If you want to make changes, fork the repository)

   ```bash
   git clone https://github.com/gregnrobinson/tekton-kustomize.git
   cd tekton-kustomize
   ```

2. Create a file named `.env` and adjust the following variables to match your environment.

   ```yaml
   cat <<EOF >>.env
   # - Kustomize Overlay Environment - #
   # The overlay configuration to target. Dev is the only configured overlay by default.
   ENV=dev
   
   # - Kubernetes Context - #
   # Sets the Kubernetes context that all manifests will deploy to.
   # Defaults to current context if not set.
   CONTEXT=""
   
   # - Github SSH Key - #
   # The SSH private key path used for authenticating to target Github repositories.
   SSH_KEY_PATH=~/.ssh/id_rsa

   # - Docker Config - #
   # Run 'docker login' to generate a config file.
   DOCKER_CONFIG_PATH=~/.docker/config.json

   # - Github Webhook Secret Token - #
   # Used by Tekton triggers to call a webhook configured within a repo using the configured secret.
   # Refer to https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks for creating a webook and secret.
   GITHUB_SECRET="<YOUR_GITHUB_SECRET>"

   # - Sonar Token - #
   # - Ussed for running sonar scans. it is not required to set this value for any tasks except the sonar-scan pipeline run.
   # - The mvn-build pipeline run has an option to run sonar scanning but can be disabled using the value false for the runSonarScan parameter
   SONAR_TOKEN="<YOUR_SONAR_TOKEN>"
   EOF
   ```

3. Apply the manifests.

   ```bash
   ./tekton.sh --apply
   ```

## Usage

Run `./tekton.sh -h` to display the help menu.

```bash
Usage: tekton.sh [option...]

   -a, --apply         Install and deploy Tekton resources. 
   -s, --sync          Pull the latest pipeline and trigger releases. 
   -p, --prune         Delete all Completed, Errored or DeadLineExceeded pod runs. 
   -c, --creds         Create secret declerations from the provided values in .env. 
   -h, --help          Display argument options. 
```

## Pipeline Run Templates

All pipeline run templates listed below are tested and working. The `PipelineRun` templates refernece pipelines and tasks that were deployed using `./tekton.sh`. All the dependancies to operate this repository are within the repository. Developers can focus on consuming the pipelines for their needs with minimal changes. Additional to adhoc use, developers can create a yaml file with the following templates and store them in a Git repositoy where they can incorporate the pipeline runs into their own automation workflows. Aslong as the runner has access a Kibernetes cluster, a pipeline run will execute with just `kubectl apply -f <YOUR_PIPELINE_RUN>.yaml`.

### **buildah-build-push**

*Build and push a docker image using [buildah](https://buildah.io/).*

```yaml
cat <<EOF | kubectl create -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: docker-build-push-run-
spec:
  pipelineRef:
    name: p-buildah
  params:
  - name: appName
    value: flask-web
  - name: repoUrl
    value: git@github.com:bcgov/security-pipeline-templates.git
  - name: imageUrl
    value: gregnrobinson/tkn-flask-web:latest
  - name: branchName
    value: main
  - name: dockerfile
    value: ./Dockerfile
  - name: pathToContext
    value: ./tekton/demo/flask-web
  - name: buildahImage
    value: quay.io/buildah/stable
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 500Mi
  - name: ssh-creds
    secret:
      secretName: tkn-ssh-credentials
  - name: docker-config
    secret:
      secretName: tkn-docker-credentials
EOF
```

### **maven-build**

*Builds and a java application with [maven](https://maven.apache.org/).*

```yaml
cat <<EOF | kubectl create -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: mvn-build-run-
spec:
  pipelineRef:
    name: p-mvn-build
  params:
  - name: appName
    value: maven-test
  - name: mavenImage
    value: index.docker.io/library/maven
  - name: repoUrl
    value: git@github.com:bcgov/security-pipeline-templates.git
  - name: branchName
    value: main
  - name: pathToContext
    value: ./tekton/demo/maven-test
  - name: runSonarScan
    value: 'true'
  - name: sonarProject
    value: tekton
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  - name: ssh-creds
    secret:
      secretName: tkn-ssh-credentials
  - name: docker-config
    secret:
      secretName: tkn-docker-credentials
  - name: maven-settings
    emptyDir: {}
EOF
```

### **codeql-scan**

*Scans a given repository for explicit languages. [CodeQL](https://codeql.github.com/)*

```yaml
cat <<EOF | kubectl create -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: codeql-scan-run-
spec:
  pipelineRef:
    name: p-codeql
  params:
  - name: buildImageUrl
    value: docker.io/gregnrobinson/codeql-cli:latest
  - name: repoUrl
    value: git@github.com:bcgov/security-pipeline-templates.git
  - name: repo
    value: bcgov/security-pipeline-templates
  - name: branchName
    value: main
  - name: pathToContext
    value: .
  - name: releaseName
    value: codeql.zip
  - name: githubToken
    value: tkn-github-token
  - name: version
    value: v2.7.0
  - name: language
    value: python
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  - name: ssh-creds
    secret:
      secretName: tkn-ssh-credentials
  - name: docker-config
    secret:
      secretName: tkn-docker-credentials
EOF
```

### **sonar-scan**

*Scans a given repository against a provided SonarCloud project. [SonarCloud](https://sonarcloud.io/)*

```yaml
cat <<EOF | kubectl create -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: sonar-scanner-run-
spec:
  pipelineRef:
    name: p-sonar
  params:
  - name: sonarHostUrl
    value: https://sonarcloud.io
  - name: sonarProject
    value: app-factory
  - name: repoUrl
    value: git@github.com:gregnrobinson/app-factory.git
  - name: branchName
    value: main
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  - name: ssh-creds
    secret:
      secretName: tkn-ssh-credentials
  - name: docker-config
    secret:
      secretName: tkn-docker-credentials
  - name: sonar-settings
    emptyDir: {}
EOF
```

### **trivy-scan**

*Scans for vulnerbilities and file systems. [SonarCloud](https://github.com/aquasecurity/trivy)*

```yaml
cat <<EOF | kubectl create -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: trivy-scanner-run-
spec:
  pipelineRef:
    name: p-trivy
  params:
  - name: targetImage
    value: python:3.4-alpine
  - name: repoUrl
    value: git@github.com:bcgov/security-pipeline-templates.git
  - name: branchName
    value: main
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  - name: ssh-creds
    secret:
      secretName: tkn-ssh-credentials
  - name: docker-config
    secret:
      secretName: tkn-docker-credentials
EOF
```

## How It Works

Much of the heavy lifting is performed by a tool called [Kustomize](https://kustomize.io/). Kustomize comes pre-bundled with **kubectl version >= 1.14** which means the only required prerequisite for developers to deploy this project is a target cluster and the latest version of kubectl.

All manifests that pertain to the installation of Tekton are located at the root of `./base/`.

All Tekton `Pipeline` and `Task` resource types are located respectively at `./base/pipelines` and `./base/tasks`.

```bash
tekton-kustomize
├── README.md
├── base
│   ├── install
│   │   ├── dashboards.yaml
│   │   ├── interceptors.yaml
│   │   ├── kustomization.yaml
│   │   ├── pipelines.yaml
│   │   └── triggers.yaml
│   ├── kustomization.yaml
│   ├── pipelines
│   │   ├── buildah.yaml
│   │   ├── kustomization.yaml
│   │   └── maven.yaml
│   ├── tasks
│   │   ├── buildah.yaml
│   │   ├── git-clone.yaml
│   │   ├── kustomization.yaml
│   │   └── maven-build.yaml
│   └── triggers
│       ├── ingress.yaml
│       ├── kustomization.yaml
│       ├── rbac.yaml
│       └── trigger-template.yaml
├── demo
├── overlays
│   ├── dev
│   │   ├── dashboards.yaml
│   │   └── kustomization.yaml
│   └── prod
│       ├── kustomization.yaml
│       └── dashboards.yaml
└── ktek.sh
```

When `./tekton.sh apply` is executed, the first operation to take place is a sync whereby the remote latest Tekton release is copied to `./base/install`.

Following the sync operation is the creation of the `./overlays/${ENV}/creds` folder in one of the overlay environments that is declared in `.env`. The `${SSH_KEY_PATH}` and `${DOCKER_CONFIG_PATH}` files are copied from the provided paths in `.env` to the temporary creds folder.

The last step is executing kustomize using `kubectl apply -k overlays/${ENV}`. This command will execute the following kustomize configuration in `./overlays/${ENV}`.

After the execution, a cleanup function deletes the creds folder and removes the `$GITHUB_SECRET` value from `./overlays/${ENV}/kustomization.yaml`.

```yaml
# ./overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/
patchesStrategicMerge:
  - dashboards.yaml
generatorOptions:
  disableNameSuffixHash: true
secretGenerator:
  - name: tkn-ssh-credentials
    files:
      - creds/id_rsa
  - name: tkn-docker-credentials
    files:
      - creds/.dockerconfigjson
    type: kubernetes.io/dockerconfigjson
  - name: github-secret
    type: Opaque
    literals:
      - secretToken=
  - name: sonar-token
    type: Opaque
    literals:
      - secretToken=
patchesJson6902:
  - target:
      version: v1
      kind: Secret
      name: github-secret
    patch: |-
      - op: add
        path: /metadata/annotations
        value:
          tekton.dev/git-0: github.com
```

When the base templates are applied using `./overlays/${ENV}/kustomization.yaml`, the kustomization.yaml file at the root of base deploys all the declared manifests in each folder using `./base/kustomization.yaml`.

```yaml
# ./base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ./install
  - ./pipelines
  - ./tasks
  - ./triggers
```

Declaring the folder as a resource will find and execute any kustomization.yaml files within the directories accordingly. All manifests are explicitly declared which allows for resources currently under development to be excluded from the deployment. This eliminates the need for branching when creating new Tekton resources. Aslong as the resources are not declared in Kustomize, the changes will not be breaking.

```yaml
## ./base/install/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - pipelines.yaml
  - triggers.yaml
  - interceptors.yaml
  - dashboards.yaml

## ./base/pipelines/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namePrefix: p-

resources:
  - buildah.yaml
  - maven.yaml

## ./base/tasks/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namePrefix: t-

resources:
  - buildah.yaml
  - git-clone.yaml
  - maven-build.yaml

## ./base/triggers/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ingress.yaml
  - rbac.yaml
  - trigger-template.yaml
```