# CI/CD with GitHub Actions and GitOps

This project demonstrates a **CI/CD pipeline using GitHub Actions, Docker Hub, Git, Kubernetes and Argo CD**.


## Architecture Diagram




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
10. Argo CD detects the Git change and synchronizes the Kubernetes cluster.


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


## Technologies

| Technology       | Purpose                         |
|------------------|---------------------------------|
| GitHub           | Source code and Git repository  |
| GitHub Actions   | Continuous integration          |
| Docker           | Container image creation        |
| Docker Hub       | Container image registry        |
| `yq`             | YAML manifest modification      |
| Argo CD          | GitOps continuous delivery      |
| Kubernetes       | Container orchestration         |


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

The goal of this project is to demonstrate a simple and practical **CI/CD + GitOps workflow**:

This approach keeps application delivery automated while maintaining Git as the declarative source of truth for Kubernetes deployments.
