# Zero-Downtime Blue-Green Deployment Pipeline on ECS Fargate

A CI/CD pipeline that deploys a containerized app to ECS Fargate using a
true blue-green deployment strategy — the new version is checked privately
behind a test header before any real traffic reaches it, then production
traffic shifts all at once once it's approved. A manual approval gate
(covered below) is what actually makes "approved" mean something here —
without it, the shift happens automatically the moment the new version
passes its health checks, whether or not anyone looked at it first.

This uses **Amazon ECS's native blue/green deployment feature**, not AWS
CodeDeploy. ECS folded CodeDeploy's job (traffic shifting, bake time,
rollback) directly into the service itself in July 2025 — no separate
CodeDeploy application, deployment group, or `appspec.yaml` needed. On
top of that, this project adds a `PAUSE` lifecycle hook for manual
approval before cutover — a plain ECS feature, no Lambda required.

![Architecture diagram](./images/architecture-diagram.png)

**Read the full guide: [`guide.md`](./guide.md)** — it covers the
introduction, architecture, a click-by-click walkthrough in the AWS
Console, the full CloudFormation templates with deployment and teardown
commands, and a troubleshooting section covering real errors hit while
building this (Docker permission issues, missing CodeBuild environment
variables, ECR push permissions, the security group egress issue that
causes `ResourceInitializationError` on task start, and a CloudFormation
dependency race that can fail ALB creation).

## Why blue-green instead of a rolling update

A rolling update replaces old tasks with new ones gradually — if the new
version has a bug, some users hit it while you're still detecting the
problem. Blue-green deploys the new version alongside the old one and
lets you check it through a private test route first. With the manual
approval hook this project adds, nothing goes live until you say so; if
it looks wrong, you reject it and the old version never stopped serving
traffic in the first place.

## What's in this repo

```
.
├── README.md                                  ← you are here
├── guide.md                                    ← the full guide: intro, architecture,
│                                                  console walkthrough, CloudFormation,
│                                                  and teardown for both
├── cloudformation/
│   ├── 00-prerequisites-stack.yaml             ← deploy FIRST: just the ECR repo and
│   │                                              GitHub connection, so the two manual
│   │                                              steps can happen before anything else
│   │                                              depends on them
│   └── blue-green-ecs-stack.yaml               ← deploy SECOND: everything else (VPC,
│                                                  ALB, ECS service, pipeline)
└── pipeline-config/
    ├── buildspec.yml                           ← CodeBuild: builds image, pushes it,
    │                                              writes imagedefinitions.json
    ├── Dockerfile                               ← minimal placeholder app for testing the pipeline
    └── index.html                               ← placeholder app content
```

## Before you clone this and start

The pipeline pulls its source from GitHub, so it needs somewhere to pull
from. Clone this repo and push the **whole project as-is** to your own
GitHub repo — keep `pipeline-config/` as a subfolder rather than
flattening it into the root, since the buildspec and CodeBuild config
both expect it to stay there. Do this before starting either the console
or CloudFormation walkthrough — without it, the pipeline's Source stage
has nothing to work with.

## Cost note

This stack has a running cost as long as it's up — mainly the load
balancer and the Fargate tasks. Build it, test it, then tear it down
rather than leaving it running. Set a billing alarm before you start.
