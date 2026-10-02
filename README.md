# Zero-Downtime Blue-Green Deployment Pipeline on ECS Fargate

A CI/CD pipeline that deploys a containerized app to ECS Fargate using a
true blue-green deployment strategy — the new version is tested privately
behind a second listener before any real traffic reaches it, and rollback
means simply not shifting traffic to it, not scrambling to undo a bad
rolling update.

This uses **Amazon ECS's native blue/green deployment feature**, not AWS
CodeDeploy. ECS folded CodeDeploy's job (traffic shifting, bake time,
rollback) directly into the service itself in July 2025 — no separate
CodeDeploy application, deployment group, or `appspec.yaml` needed.

**Read the full guide: [`guide.md`](./guide.md)** — it covers the
introduction, architecture, a click-by-click walkthrough in the AWS
Console, the full CloudFormation template with deployment and teardown
commands, and a troubleshooting section covering real errors hit while
building this (Docker permission issues, missing CodeBuild environment
variables, ECR push permissions, and the security group egress issue
that causes `ResourceInitializationError` on task start).

## Why blue-green instead of a rolling update

A rolling update replaces old tasks with new ones gradually — if the new
version has a bug, some users hit it while you're still detecting the
problem. Blue-green deploys the new version alongside the old one, tests
it against real infrastructure through a separate listener, and only then
switches production traffic over. If something's wrong, you just don't
switch — the old version never stopped running.

## What's in this repo

```
.
├── README.md                                  ← you are here
├── guide.md                                    ← the full guide: intro, architecture,
│                                                  console walkthrough, CloudFormation,
│                                                  and teardown for both
├── cloudformation/
│   └── blue-green-ecs-stack.yaml               ← the CloudFormation template on its own,
│                                                  if you just want the file to deploy
└── pipeline-config/
    ├── buildspec.yml                           ← CodeBuild: builds image, pushes it,
    │                                              writes imagedefinitions.json
    ├── Dockerfile                        ← minimal placeholder app for testing the pipeline
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
