# eks-gitops-app

**Application repository** for Project 4 of my DevOps portfolio — three FastAPI
microservices deployed to Amazon EKS using a **GitOps** workflow (Argo CD).

This is the *app* half of a deliberate two-repository split:

| Repo | Role |
|------|------|
| **`eks-gitops-app`** (this repo) | Application source code. CI **builds** images, **pushes** to ECR, and **promotes** the new image tag by committing to the config repo. It never talks to the cluster. |
| [`eks-gitops-config`](https://github.com/fuhchu/eks-gitops-config) | The GitOps source of truth. Holds the Helm charts and Argo CD `Application`s. Argo CD, running *inside* the cluster, watches this repo and reconciles the cluster to match it. |

## Why two repos?

In [Project 3](https://github.com/fuhchu/eks-helm) CI ran `helm upgrade`
directly against the cluster — a **push-based** deploy where CI held cluster
credentials and could drift from git. Project 4 inverts that into **pull-based**
GitOps:

1. A commit to a service here triggers CI.
2. CI runs tests, builds the image, and pushes it to ECR tagged with the commit SHA.
3. CI then **commits the new tag to `eks-gitops-config`** — it does *not* deploy.
4. Argo CD notices the config repo changed and pulls the new image into the cluster.

Benefits: git is the deploy audit log, CI holds **no** cluster credentials
(smaller blast radius), Argo CD self-heals manual drift, and a rollback is a
`git revert`.

A config-only change (scaling replicas, editing a dashboard) lives entirely in
the config repo and never rebuilds an image — that separation is the reason the
two concerns are split across two repos.

## Services

| Service | Port | Responsibility |
|---------|------|----------------|
| `api-gateway` | 8000 | Reverse proxy / single entry point; fans out to users and items |
| `users` | 8001 | User CRUD, backed by PostgreSQL |
| `items` | 8002 | Item CRUD, backed by PostgreSQL; validates owners via the users service |

Each service is a multi-stage Docker build running as a non-root user.

## Portfolio series

1. [ecs-fargate-rest-api](https://github.com/fuhchu/ecs-fargate-rest-api) — FastAPI on ECS Fargate
2. [ecs-microservices](https://github.com/fuhchu/ecs-microservices) — ECS microservices + RDS
3. [eks-helm](https://github.com/fuhchu/eks-helm) — EKS + Helm
4. **eks-gitops-app / eks-gitops-config** — EKS + GitOps (Argo CD) + Observability ← *you are here*
