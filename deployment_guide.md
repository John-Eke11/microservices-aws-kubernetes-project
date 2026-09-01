# Coworking Space Analytics — Deployment Guide

This service reports on coworking check-in activity through two endpoints, `/api/reports/daily_usage` and `/api/reports/user_visits`, backed by a Postgres database. It runs on AWS as a small Kubernetes deployment on EKS.

## How it's built and deployed

The Flask app is packaged into a Docker image (`analytics/Dockerfile`), built on a slim Python base chosen to match the app's dependencies. AWS CodeBuild watches this repo through a GitHub webhook: every push to `main` triggers `buildspec.yml`, which builds the image, tags it with a semantic version tied to the CodeBuild build number, and pushes it to Amazon ECR. That versioned tag is what actually gets referenced in the Kubernetes Deployment, so it's always obvious which build is currently running.

Kubernetes takes over from there. The manifests in `deployment/` define a Service (type `LoadBalancer`, so AWS provisions a public endpoint automatically) and a Deployment that pulls the image, sets CPU/memory limits, and wires in environment variables from two separate sources: a ConfigMap for plain settings like the database host and port, and a Secret for the database password. Liveness and readiness probes hit the app's `/health_check` and `/readiness_check` routes, so a pod won't receive traffic until it can actually reach the database, and it gets restarted automatically if it stops responding.

Postgres itself runs inside the cluster too, installed via the Bitnami PostgreSQL Helm chart rather than hand-written manifests — that keeps storage, credentials, and networking consistent with a well-tested upstream chart instead of reinventing it. Application logs and health status stream out through the CloudWatch Container Insights EKS add-on, so you can watch what the app is actually doing from the AWS console without needing cluster access.

## Releasing a new build

Push your change to `main`. CodeBuild picks it up automatically, builds a fresh image, and pushes a new version tag to ECR — no manual Docker commands needed. Update the `image:` field in `deployment/coworking.yaml` to that new tag and run `kubectl apply -f deployment/`. Kubernetes then performs a rolling update, and because of the readiness probe, it won't shift traffic to the new pod until it's healthy and actually connected to the database.

## Stand-out suggestions

**Memory and CPU allocation:** The Deployment requests 250m CPU / 256Mi memory per pod and caps at 500m CPU / 512Mi — enough headroom for Flask plus the periodic background job without one pod starving its neighbors. These numbers came from watching the app's actual resource use rather than guesswork, and they're easy to adjust later with `kubectl top pod`.

**Instance type:** A burstable, general-purpose instance like `t3.medium` or `m7i-flex.large` fits this workload well, since the app spends most of its time waiting on Postgres rather than burning CPU. The bigger constraint to watch for is AWS's per-node pod limit, which is driven by available network interfaces rather than CPU or memory — it's worth sizing up rather than assuming a small instance will comfortably fit both system pods and the app.

**Cost savings:** The biggest lever is right-sizing the node group and letting it scale down automatically during quiet periods, rather than running one oversized node around the clock. Beyond that, Spot instances for the stateless app pods (keeping Postgres and its EBS volume off Spot) and periodically cleaning up old ECR image tags are both low-risk ways to trim the bill.
