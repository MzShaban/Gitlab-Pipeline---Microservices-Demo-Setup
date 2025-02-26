
# Microservices Demo Setup with GitLab CI/CD

This document outlines the steps to set up and run the Microservices Demo application using GitLab, Docker, and GitLab CI/CD.

## Prerequisites

- Ubuntu (or a similar Linux distribution)
- Docker
- Git

## Setup Instructions

### Step 1: Run GitLab in Docker

Start the GitLab container on Ubuntu:

```bash
docker run -d \
  --hostname gitlab.localhost \
  --publish 80:80 --publish 443:443 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume gitlab-config:/etc/gitlab \
  --volume gitlab-logs:/var/log/gitlab \
  --volume gitlab-data:/var/opt/gitlab \
  gitlab/gitlab-ee:latest
```

Wait a few minutes for GitLab to start, then access it at:
👉 http://gitlab.localhost

### Step 2: Clone the Microservices Demo Repository

```bash
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo
```

### Step 3: Create a New GitLab Repository

 * Log in to GitLab (http://gitlab.localhost).
 * Go to Projects > Create New Project.
 * Click "Create Blank Project".
 * Set Project Name: microservices-demo.
 * Set Visibility to Private/Public.
 * Click Create Project.

### Step 4: Push the Cloned Repository to GitLab

```bash
git init
git remote add origin http://gitlab.localhost/myprojects/microservices-demo.git
git add .
git commit -m "Initial commit: Microservices Demo"
git push -u origin main
```

Note: Replace myprojects with your GitLab username or group.

![alt text](https://github.com/MzShaban/Devops-projects/blob/main/Images/Images/repository-microservices-demo?raw=true)


### Step 5: Run GitLab Runner in Docker

```bash
docker run -d \
  --name gitlab-runner \
  --restart always \
  --volume /src/gitlab-runner/config:/etc/gitlab-runner \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  --volume gitlab_shared_volume:/path/to/gitlab/project \
  --net gitlab_network \
  gitlab/gitlab-runner:latest
```

Note: You might need to create the gitlab_network and gitlab_shared_volume if they do not exist.

### Step 6: Register GitLab Runner

 * Find the registration token under GitLab > Settings > CI/CD > Runners.
 * Run:

```bash
docker exec gitlab-runner gitlab-runner register -n \
  --url http://gitlab.localhost \
  --registration-token $REGISTRATION_TOKEN \
  --description "Docker Runner" \
  --executor docker \
  --docker-image "alpine:latest" \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
  --docker-network-mode gitlab_network
```

Replace $REGISTRATION_TOKEN with your actual runner token.

### Step 7: Edit config.toml for GitLab Runner

```bash
nano /src/gitlab-runner/config/config.toml
```

Modify:

```toml
[[runners]]
  name = "Docker Runner"
  url = "http://gitlab.localhost"
  executor = "docker"

  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = true
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
    extra_hosts = ["gitlab.localhost:172.20.0.2"]
    network_mode = "host"
```

Save and restart the Runner:

```bash
docker restart gitlab-runner
```

![alt text](https://github.com/MzShaban/Devops-projects/blob/main/Images/runner-microservices-demo?raw=true)


### Step 8: Create Dockerfile for Frontend

```bash
nano src/frontend/Dockerfile
```

Paste:
```dockerfile
# Build Stage
FROM --platform=$BUILDPLATFORM golang:1.23.4-alpine AS builder
ARG TARGETOS
ARG TARGETARCH
WORKDIR /src/frontend

RUN mkdir -p /src/frontend/static /src/frontend/templates
COPY src/frontend/go.mod src/frontend/go.sum ./
RUN go mod download
COPY src/frontend/ /src/frontend/
RUN ls -la /src/frontend
ARG SKAFFOLD_GO_GCFLAGS
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} CGO_ENABLED=0 go build -gcflags="${SKAFFOLD_GO_GCFLAGS}" -o /go/bin/frontend .

# Final Stage
FROM scratch
WORKDIR /src
COPY --from=builder /go/bin/frontend /src/server
COPY --from=builder /src/frontend/templates /src/templates/
COPY --from=builder /src/frontend/static /src/static/
ENV GOTRACEBACK=single
EXPOSE 8080
ENTRYPOINT ["/src/server"]
```

### Step 9: Create GitLab CI/CD Pipeline (.gitlab-ci.yml)

```bash
nano .gitlab-ci.yml
```

Paste:
```yaml
stages:
  - build
  - test
  - deploy

build:
  image: golang:1.23
  stage: build
  script:
    - echo "Building Go services"
    - cd src/frontend
    - mkdir -p static templates
    - go build -o frontend .
    - ls -la
  artifacts:
    paths:
      - src/frontend/frontend
      - src/frontend/static
      - src/frontend/templates
    expire_in: 1h
  cache:
    paths:
      - src/frontend/go.sum
      - src/frontend/go.mod
      - /go/pkg/mod

test:
  image: golang:1.23
  stage: test
  script:
    - echo "Running tests"
    - cd src/frontend && go test ./...
  dependencies:
    - build

deploy:
  image: docker:latest
  stage: deploy
  variables:
    DOCKER_HOST: "unix:///var/run/docker.sock"
  script:
    - ls -la src/frontend
    - docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
    - docker build -t $DOCKER_IMAGE_MIC:latest -f src/frontend/Dockerfile .
    - docker push $DOCKER_IMAGE_MIC:latest
  only:
    - main
  dependencies:
    - build
```

Note: Replace $DOCKER_USERNAME, $DOCKER_PASSWORD, and $DOCKER_IMAGE_MIC with your Docker credentials and image name.

### Step 10: Commit and Push Changes

```bash
git add .gitlab-ci.yml src/frontend/Dockerfile
git commit -m "Added GitLab CI/CD pipeline and Dockerfile"
git push origin main
```

### Step 11: Run the Pipeline

Go to GitLab UI > CI/CD > Pipelines to check the pipeline status.

![alt text](https://github.com/MzShaban/Devops-projects/blob/main/Images/pipeline-microservices-demo?raw=true)

## The Docker image is pushed in the docker hub as a result

![alt text](https://github.com/MzShaban/Devops-projects/blob/main/Images/dockerimagemicroservices?raw=true)
