# Kubernetes CI/CD with GitHub Actions and GitOps

This project demonstrates a **Kubernetes CI/CD pipeline using GitHub
Actions, Docker Hub, Git, and Argo CD**.

The pipeline follows a GitOps-style workflow:

``` text
Developer
   |
   v
GitHub Source Repository
   |
   v
GitHub Actions
   |
   +---- Build Docker Image
   |
   +---- Push Image to Docker Hub
   |
   +---- Update Kubernetes Manifest
   |
   +---- Commit & Push Manifest
   |
   v
Git Repository
   |
   v
Argo CD
   |
   | GitOps Sync
   v
Kubernetes Cluster
   |
   v
New Application Version
```

## Architecture

The main components are:

-   **Developer** --- commits application code.
-   **GitHub Repository** --- stores application source code and
    Kubernetes manifests.
-   **GitHub Actions** --- performs the CI pipeline.
-   **Docker** --- builds the application container image.
-   **Docker Hub** --- stores the versioned container image.
-   **Git repository** --- stores the updated Kubernetes deployment
    manifest.
-   **Argo CD** --- watches the Git repository and synchronizes
    Kubernetes resources.
-   **Kubernetes** --- runs the updated application.

## CI/CD Workflow

The GitHub Actions workflow runs when code is pushed to the `dev`
branch.

### Pipeline steps

1.  Checkout the source code.
2.  Build the frontend Docker image.
3.  Authenticate with Docker Hub.
4.  Push the Docker image using the short Git commit SHA as the tag.
5.  Install `yq`.
6.  Update the Kubernetes deployment manifest with the new image tag.
7.  Display the manifest changes.
8.  Commit the updated manifest.
9.  Push the manifest change back to the `dev` branch.
10. Argo CD detects the Git change and synchronizes the Kubernetes
    cluster.

The image tag is based on:

``` bash
${GITHUB_SHA::7}
```

For example:

``` text
xrootms/app-frontend:8f31c2a
```

This makes each deployed image traceable to a specific Git commit.

## Repository Structure

A typical project structure is:

``` text
.
├── .github/
│   └── workflows/
│       └── kubernetes-ci.yml
│
├── app/
│   └── frontend/
│       ├── Dockerfile
│       └── ...
│
├── k8s/
│   └── app/
│       └── frontend/
│           └── frontend-deployment.yaml
│
└── README.md
```

## GitHub Actions Workflow

The workflow is triggered by pushes to `dev`:

``` yaml
name: Kubernetes CI

on:
  push:
    branches:
      - dev
```

### Docker image

The frontend image is built with:

``` bash
docker build \
  -t xrootms/app-frontend:${GITHUB_SHA::7} \
  --build-arg REACT_APP_API_URL=/api/v1 \
  ./app/frontend
```

The image is then pushed to Docker Hub:

``` bash
docker push xrootms/app-frontend:${GITHUB_SHA::7}
```

## Required GitHub Secrets

The workflow requires Docker Hub credentials to be configured in:

**GitHub → Repository → Settings → Secrets and variables → Actions**

Required secrets:

  Secret              Description
  ------------------- -------------------------------------
  `DOCKER_USERNAME`   Docker Hub username
  `DOCKER_PASS`       Docker Hub password or access token

A Docker Hub access token is preferable to using a personal password.

## Updating the Kubernetes Manifest

After the image is pushed, `yq` updates the container image in:

``` text
k8s/app/frontend/frontend-deployment.yaml
```

The workflow changes:

``` yaml
spec:
  template:
    spec:
      containers:
        - image: xrootms/app-frontend:OLD_TAG
```

to:

``` yaml
spec:
  template:
    spec:
      containers:
        - image: xrootms/app-frontend:NEW_TAG
```

The new tag is the short Git commit SHA.

## GitOps with Argo CD

GitHub Actions does **not** directly deploy the application to
Kubernetes.

Instead, it updates the Kubernetes manifest in Git.

Argo CD watches the Git repository:

``` text
GitHub Repository
       |
       | manifest changed
       v
     Argo CD
       |
       | synchronize
       v
Kubernetes Cluster
```

This separation is the GitOps approach:

-   Git is the source of truth.
-   CI builds and publishes the application image.
-   CI updates the desired Kubernetes state.
-   Argo CD reconciles the cluster with Git.

## Example Deployment

The Kubernetes deployment should reference the image repository:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: frontend
          image: xrootms/app-frontend:8f31c2a
```

When a new commit produces a new image:

``` text
8f31c2a
```

the manifest becomes:

``` yaml
image: xrootms/app-frontend:8f31c2a
```

Argo CD then synchronizes this desired state to Kubernetes.

## Recommended GitHub Actions Permissions

If the workflow commits the updated manifest using `GITHUB_TOKEN`,
explicitly allow repository write access:

``` yaml
permissions:
  contents: write
```

Example:

``` yaml
name: Kubernetes CI

on:
  push:
    branches:
      - dev

permissions:
  contents: write
```

Repository settings can also restrict the default workflow token
permissions, so verify that the workflow is allowed to write repository
contents.

## Avoiding Empty Commits

When the manifest has not changed, `git commit` can fail with:

``` text
nothing to commit
```

A safer commit step is:

``` bash
git add k8s/app/frontend/frontend-deployment.yaml

if git diff --cached --quiet; then
  echo "No changes to commit"
  exit 0
fi

git commit -m "docker tag updated by GHA"
git push
```

## Security

Do not hard-code credentials in the workflow.

Use GitHub Secrets:

``` yaml
with:
  username: ${{ secrets.DOCKER_USERNAME }}
  password: ${{ secrets.DOCKER_PASS }}
```

Recommended practices:

-   Use a Docker Hub access token instead of a password.
-   Keep secrets out of source control.
-   Use least-privilege credentials.
-   Pin third-party GitHub Actions versions where practical.
-   Pin tools such as `yq` to a known version rather than downloading
    `latest` in production CI.
-   Consider image scanning before deployment.
-   Consider Kubernetes admission/security policies for production
    clusters.

## Deployment Flow

A complete deployment looks like this:

``` text
1. Developer
      |
      | git push
      v
2. GitHub
      |
      v
3. GitHub Actions
      |
      +--> Checkout
      |
      +--> Docker build
      |
      +--> Docker Hub login
      |
      +--> Docker push
      |
      +--> Update deployment.yaml
      |
      +--> git commit
      |
      +--> git push
      |
      v
4. Git Repository
      |
      | GitOps reconciliation
      v
5. Argo CD
      |
      v
6. Kubernetes
      |
      +--> Deployment updated
      |
      +--> New Pod created
      |
      +--> New Docker image pulled
```

## Technologies

  Technology       Purpose
  ---------------- --------------------------------
  GitHub           Source code and Git repository
  GitHub Actions   Continuous integration
  Docker           Container image creation
  Docker Hub       Container image registry
  `yq`             YAML manifest modification
  Argo CD          GitOps continuous delivery
  Kubernetes       Container orchestration

## Local Development

Build the frontend image locally:

``` bash
docker build \
  -t xrootms/app-frontend:local \
  --build-arg REACT_APP_API_URL=/api/v1 \
  ./app/frontend
```

Run it locally:

``` bash
docker run --rm -p 3000:3000 xrootms/app-frontend:local
```

Adjust the port according to the frontend application's Docker
configuration.

## Troubleshooting

### Docker push fails

Check:

-   `DOCKER_USERNAME` is configured.
-   `DOCKER_PASS` is valid.
-   The Docker Hub repository exists.
-   The GitHub Actions runner can authenticate to Docker Hub.

### Git push fails

Check:

-   Workflow has `contents: write`.
-   Repository Actions permissions allow write access.
-   The workflow is using the expected `GITHUB_TOKEN`.

### Argo CD does not deploy the new image

Check:

-   Argo CD is watching the correct repository.
-   Argo CD is watching the correct branch/path.
-   The Kubernetes manifest contains the new image tag.
-   Argo CD synchronization is enabled or manually synchronized.
-   The Kubernetes cluster can pull the Docker image.

### Kubernetes cannot pull the image

Check:

-   The image exists in Docker Hub.
-   The image tag matches the manifest.
-   The repository is public, or Kubernetes has a valid
    `imagePullSecret` for a private repository.

## Project Goal

The goal of this project is to demonstrate a simple and practical
**CI/CD + GitOps workflow**:

> **Build → Push Image → Update Git Manifest → Argo CD Sync → Kubernetes
> Deploy**

This approach keeps application delivery automated while maintaining Git
as the declarative source of truth for Kubernetes deployments.
