# Technical Notes & Improvements

This document records the technical issues encountered during the course and the improvements made beyond the original implementation.

The goal was to make the CI/CD pipeline more robust, reproducible, and closer to real-world production standards.

---

## 1. ECS Deployment Rollback Issue (ARM vs AMD64)

### Problem

After updating the ECS service with a new task definition, deployments consistently rolled back with the message:

> Service deployment rolled back because the circuit breaker threshold was exceeded.

The root cause was an **architecture mismatch**:

- Docker images were built on an Apple Silicon (ARM64) machine
- Amazon ECS/Fargate defaults to **AMD64**
- Single-architecture images cannot run on a mismatched platform

This is similar to building a Docker image on macOS (ARM) and attempting to run it on a Windows or x86 Linux machine.

---

### Solution

To fix this, the pipeline was updated to build images for **both architectures**.

```
sh'''
    docker buildx create --use --name multiarch || true

    docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t $AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION \
    --push \
    .
'''
```

**Why `docker buildx create` Is Required In Jenkins**

In local development (Docker Desktop):

- Buildx is enabled by default
- A builder already exists

In Jenkins:

- Docker runs inside a container
- Buildx may not be configured
- A builder must be explicitly created before use

The || true ensures the pipeline does not fail if the builder already exists.

---

## 2. Improve ECS Task Definition Version Handling

### Problem

The course uses the following command to update the image version:

```
sed -i "s/#APP_VERSION#/$REACT_APP_VERSION/g" aws/task-definition-prod.json
```
This approach avoids hardcoding a version number, but it still directly modifies a tracked file and performs a string-based replacement without understanding JSON structure.

---

### Solution

Instead of editing the file in place, this project uses jq to generate a new task definition file dynamically:

```
IMAGE_URI="$AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION"

jq \
  --arg IMAGE_URI "$IMAGE_URI" \
  '.containerDefinitions[0].image = $IMAGE_URI' \
  aws/task-definition-prod.json \
  > aws/task-definition-prod-final.json
```

Key advantages:

- The original task-definition-prod.json remains unchanged
- Changes are explicit and structured
- Easier to extend (multiple containers, environment variables, resource configs)
- Better suited for production CI/CD pipelines

The generated file is then used for task registration and deployment, ensuring repeatable and traceable releases.

