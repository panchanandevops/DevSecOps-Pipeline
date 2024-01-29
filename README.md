# DevSecOps-Pipeline
In this repo we will create github workflows for building and pushing docker images. And we will use super linter to lint our code.

Lets understand our workflows one by one.

## Building and pushing Docker Images
Lets break this workflow into Job levels.

### JOB1: build-test-image

 Let's break down the `build-test-image` job in a clear and clean manner:

```yaml
jobs:
  build-test-image:
    steps:
```

**Step 1:** Set up Docker Buildx.
```yaml
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
```
This step initializes Docker Buildx, providing the necessary environment for building and pushing multi-platform Docker images.

**Step 2:** Log in to Docker Hub.
```yaml
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```
This step authenticates with Docker Hub using the provided username and token stored in GitHub secrets.

**Step 3:** Log in to ghcr.io registry.
```yaml
      - name: Login to ghcr.io registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```
Here, the job logs in to the GitHub Container Registry (ghcr.io) using the GitHub actor's credentials and the GitHub token stored in secrets.

**Step 4:** Build and Push to GHCR.
```yaml
      - name: Build and Push to GHCR
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/panchanandevops/docker-ci-automation:${{ github.run_id }}
          target: test
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64
```
This final step builds the Docker image, tags it with the repository path and the GitHub run ID, and then pushes it to the GitHub Container Registry. The job is configured to support multi-platform builds for the specified architecture.


### JOB2: test-unit
Let's break down the `test-unit` job:
```yaml
test-unit:
        name: Unit tests in Docker
        needs: [build-test-image]
        runs-on: ubuntu-latest
    
        permissions:
          packages: read
          
        steps:
          
          - name: Login to ghcr.io registry
            uses: docker/login-action@v3
            with:
              registry: ghcr.io
              username: ${{ github.actor }}
              password: ${{ secrets.GITHUB_TOKEN }}
          
          - name: Unit Testing in Docker
            run: docker run --rm ghcr.io/panchanandevops/docker-ci-automation:"$GITHUB_RUN_ID" echo "run test commands here"
```
In this job, we will test the docker image by running it.
### JOB2: scan-image
Let's break down the `scan-image` job:

```yaml
- scan-image:
    name: Scan Image with Trivy
    needs: [build-test-image]
    runs-on: ubuntu-latest

    permissions:
      contents: read # for actions/checkout to fetch code
      packages: read # needed to pull docker image to ghcr.io
      security-events: write # for github/codeql-action/upload-sarif to upload SARIF results

    steps:
```

**Step 1:** Checkout git repo.
```yaml
      - name: Checkout git repo
        uses: actions/checkout@v4
```
This step checks out the code repository, allowing subsequent steps to access and analyze the code.

**Step 2:** Login to Docker Hub.
```yaml
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```
It authenticates with Docker Hub using the provided username and token stored in GitHub secrets.

**Step 3:** Login to ghcr.io registry.
```yaml
      - name: Login to ghcr.io registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```
This step logs in to the GitHub Container Registry (ghcr.io) using the GitHub actor's credentials and the GitHub token stored in secrets.

**Step 4:** Pull image to scan.
```yaml
      - name: Pull image to scan
        run: docker pull ghcr.io/panchanandevops/docker-ci-automation:"$GITHUB_RUN_ID"
```
It pulls the Docker image from the GitHub Container Registry that was built in the previous job, using the GitHub run ID as the tag.

**Step 5:** Run Trivy for HIGH, CRITICAL CVEs and report.
```yaml
      - name: Run Trivy for HIGH, CRITICAL CVEs and report (blocking)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/panchanandevops/docker-ci-automation:${{ github.run_id }}
          exit-code: 1
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'HIGH,CRITICAL'
          format: 'sarif'
          output: 'trivy-results.sarif'
```
This step uses Trivy to scan the Docker image for vulnerabilities of HIGH and CRITICAL severity. It generates a SARIF-formatted report and sets the exit code to 1 if vulnerabilities are found.

**Step 6:** Upload Trivy scan results to GitHub Security tab.
```yaml
      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v1
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```
Finally, this step uploads the Trivy scan results in SARIF format to the GitHub Security tab. The `if: always()` ensures that this step runs even if previous steps fail.

### JOB3: build-final-image

Let's break down the `build-final-image` job:

```yaml
- build-final-image:
    name: Build Final Image
    needs: [scan-image]
    runs-on: ubuntu-latest

    permissions:
      packages: write # needed to push docker image to ghcr.io
      pull-requests: write # needed to create and update comments in PRs

    steps:
```

**Step 1:** Set up QEMU.
```yaml
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
```
This step sets up QEMU, a generic and open-source emulator, which can be useful for multi-platform Docker builds.

**Step 2:** Set up Docker Buildx.
```yaml
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
```
This step sets up Docker Buildx, an extension to the Docker CLI that provides additional features for building multi-platform Docker images.

**Step 3:** Login to Docker Hub.
```yaml
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```
It authenticates with Docker Hub using the provided username and token stored in GitHub secrets.

**Step 4:** Login to ghcr.io registry.
```yaml
      - name: Login to ghcr.io registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```
This step logs in to the GitHub Container Registry (ghcr.io) using the GitHub actor's credentials and the GitHub token stored in secrets.

**Step 5:** Docker Metadata for Final Image Build.
```yaml
      - name: Docker Metadata for Final Image Build
        id: docker_meta
        uses: docker/metadata-action@v5
        with:
          images: panchanandevops/docker-ci-automation,ghcr.io/panchanandevops/docker-ci-automation
          flavor: |
            latest=false
          tags: |
            type=raw,value=99
            type=raw,value=latest,enable=${{ endsWith(github.ref, github.event.repository.default_branch) }}
            type=ref,event=pr
            type=ref,event=branch
            type=semver,pattern={{version}}
```
This step defines metadata for the final image build, including image names, flavor, and tags based on various conditions.

**Step 6:** Docker Build and Push to GHCR and Docker Hub.
```yaml
      - name: Docker Build and Push to GHCR and Docker Hub
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.docker_meta.outputs.tags }}
          labels: ${{ steps.docker_meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64,linux/arm/v7
```
This step uses Docker Buildx to build the final Docker image and pushes it to both GitHub Container Registry (ghcr.io) and Docker Hub. It includes tags and labels based on the metadata defined in the previous step.

**Step 7:** Find comment for image tags (for PRs).
```yaml
      - name: Find comment for image tags
        uses: peter-evans/find-comment@v1
        if: github.event_name == 'pull_request'
        id: fc
        with:
          issue-number: ${{ github.event.pull_request.number }}
          comment-author: 'github-actions[bot]'
          body-includes: Docker image tag(s) pushed
```
This step looks for an existing comment in the pull request that mentions the Docker image tag(s) being pushed.

**Step 8:** Create or update comment for image tags (for PRs).
```yaml
      - name: Create or update comment for image tags
        uses: peter-evans/create-or-update-comment@v1
        if: github.event_name == 'pull_request'
        with:
          comment-id: ${{ steps.fc.outputs.comment-id }}
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            Docker image tag(s) pushed:
            ```text
            ${{ steps.docker_meta.outputs.tags }}
            ```

            Labels added to images:
            ```text
            ${{ steps.docker_meta.outputs.labels }}
            ```
          edit-mode: replace
```
This step creates or updates a comment in the pull request with information about the Docker image tags and labels that have been pushed. It includes a formatted text block with the relevant details.

## Lint Source Code
Lets break this workflow into Job levels.


### JOB1: SAST-Scan
Let's break down the `SAST-Scan` job:

```yaml
jobs:
  SAST-Scan:
    name: Lint
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: read
      statuses: write # To report GitHub Actions status checks

    steps:
```

**Step 1:** Checkout code.
```yaml
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
```
This step checks out the source code repository, ensuring a full git history is fetched (`fetch-depth: 0`). A complete history is required for Super-Linter to analyze the list of files changed across commits.

**Step 2:** Super-linter.
```yaml
      - name: Super-linter
        uses: super-linter/super-linter/slim@v5.7.2
        env:
          DEFAULT_BRANCH: main
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
This step uses Super-Linter to perform linting across various programming languages and file types. Super-Linter is a combination of multiple linters and static analysis tools. It checks for code quality, formatting, and potential issues. The `DEFAULT_BRANCH` is set to `main`, and the `GITHUB_TOKEN` is provided for reporting GitHub Actions status checks.

The job is named `SAST-Scan` and is designed for linting the codebase. It runs on an Ubuntu-latest environment with specified permissions for reading contents, packages, and writing statuses. The Super-Linter is configured to fetch the full git history and report GitHub Actions status checks using the provided GitHub token.

## Acknowledgments
I would like to express my gratitude to my teacher, **Bret Fisher**, for his invaluable guidance and support throughout the development of this project. His expertise and dedication have been instrumental in shaping my understanding and skills. I highly recommend checking out his insightful tutorials on his YouTube channel [Bret Fisher Docker and Devops](https://www.youtube.com/@BretFisher) for further learning and inspiration.

