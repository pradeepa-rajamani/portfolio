# Azure Functions CI/CD Framework (Azure DevOps)

A reusable, template-based Azure DevOps pipeline framework for building and deploying
.NET Isolated Azure Functions — designed to scale to many function apps without
touching shared pipeline code.

This repo mirrors the pipeline YAML from a working Azure DevOps project
(`WeatherFunctionApp-CI` / `WeatherFunctionApp-CD`) and documents the design
decisions and real deployment issues encountered while building it out, including
a Flex Consumption gotcha that isn't well documented by Microsoft.

> **Note:** This is a documentation/showcase mirror of pipelines that run in Azure
> DevOps. The YAML here is not runnable as-is on GitHub Actions — see
> [Architecture](#architecture) for why ADO was the right fit for this project.

---

## Why this exists

Most "getting started" Azure Functions pipeline examples hardcode everything into
a single YAML file per app. That doesn't scale once you have more than a couple
of function apps — every new app means copy-pasting and drifting pipeline logic.

This framework was built around five constraints:

- **YAML-only pipelines** — no classic UI-configured steps
- **No Variable Groups** — secrets come from Key Vault via Managed Identity, not
  pipeline variables
- **Config-driven per app** — each function app supplies its own parameters;
  shared logic lives in one place
- **Per-app deployment visibility** — each app gets its own pipeline run history,
  not a shared monolithic pipeline
- **Zero shared-template changes to onboard a new app** — adding an app means
  writing a new thin pipeline + parameters, never touching the templates

## Architecture

```mermaid
flowchart LR
    subgraph CI["WeatherFunctionApp-CI (trigger: main)"]
        A[Checkout] --> B[functionapp-build.yml]
        B --> B1[Install .NET SDK]
        B1 --> B2[Restore / Build / Publish]
        B2 --> B3[Zip artifact]
        B3 --> B4[Publish Build Artifact]
    end

    subgraph CD["WeatherFunctionApp-CD (trigger: none, manual/pipeline-triggered)"]
        C[Checkout] --> D[functionapp-deploy.yml]
        D --> D1[Download artifact from ciBuild resource]
        D1 --> D2{isFlexConsumption?}
        D2 -- true --> D3[AzureFunctionApp@2<br/>Flex Consumption path]
        D2 -- false --> D4[AzureFunctionApp@2<br/>zipDeploy path]
    end

    B4 -.->|pipeline resource: ciBuild| D1

    T1[(devops-platform-repo<br/>shared templates)] -.-> B
    T1 -.-> D
```

Two pipelines, one shared template repo:

| Pipeline | Trigger | Responsibility |
|---|---|---|
| `azure-pipelines-ci.yml` | `main` branch push | Build, publish, and zip the .NET Isolated Function project |
| `azure-pipelines-cd.yml` | `trigger: none` (manual or resource-triggered) | Download the CI artifact and deploy it to Azure |

Both call into shared templates from a separate `devops-platform-repo`:
`functionapp-build.yml` and `functionapp-deploy.yml`. A new function app is
onboarded by writing a new pair of thin pipeline files with different
parameters — the templates never change.

## Repo layout

```
pipelines/
├── azure-pipelines-ci.yml       # CI: build + publish artifact
├── azure-pipelines-cd.yml       # CD: download + deploy artifact
└── templates/
    ├── build/
    │   └── functionapp-build.yml    # shared build logic
    └── deploy/
        └── functionapp-deploy.yml   # shared deploy logic (Flex + classic plans)
```

## How it works

**CI (`azure-pipelines-ci.yml`)**
1. Checks out the app repo
2. Calls the shared build template with app-specific parameters (`.csproj` path,
   .NET version, artifact name)
3. Template installs the SDK, restores/builds/publishes the Function project,
   zips the output, and publishes it as a named pipeline artifact
   (`PublishBuildArtifacts@1`)

**CD (`azure-pipelines-cd.yml`)**
1. Declares a `resources.pipelines` reference to the CI pipeline by name
   (`ciBuild`) — this is what lets CD pull an artifact built in a *different*
   pipeline run
2. Calls the shared deploy template, which downloads that artifact and deploys
   it with `AzureFunctionApp@2`
3. The template branches on `isFlexConsumption` to pick the correct deployment
   API (see below)

## Troubleshooting journal

These are real failures hit while wiring this up, kept here because the fixes
weren't obvious from the task's own error messages.

### 1. `Artifact weather-func was not found for build 39`

**Symptom:** CD's download step failed even though CI had clearly published an
artifact with that exact name.

**Cause:** the deploy template used `download: current`, which only looks for
artifacts published *earlier in the same pipeline run*. CD is a separate,
`trigger: none` pipeline that never builds anything itself — it has no artifacts
of its own to find.

**Fix:** add a `resources.pipelines` block in the CD pipeline pointing at the CI
pipeline by name, and change `download: current` to `download: ciBuild`
(the resource alias). Pipeline-resource downloads also land under
`$(Pipeline.Workspace)/<alias>/<artifact-name>/...` rather than
`$(Pipeline.Workspace)/<artifact-name>/...`, so the downstream `packagePath`
needed updating too.

### 2. `Failed to deploy web package to App Service. Not Found (CODE: 404)` on `/api/zipdeploy`

**Symptom:** artifact download worked, but deployment failed with a 404 hitting
Kudu's zip deploy endpoint.

**Cause:** the template originally used `AzureWebApp@1`, a task built for App
Service web apps, not Function Apps. Swapping to `AzureFunctionApp@2` alone
didn't fix it either — the 404 persisted.

**Real cause:** the Function App is on a **Flex Consumption plan**, and
Flex Consumption doesn't support zip deployment at all. Every classic
Consumption/Premium/Dedicated plan defaults to it; Flex Consumption uses a
different deployment API (`/api/publish`) entirely.

**Fix:** `AzureFunctionApp@2` has a dedicated `isFlexConsumption: true` input
that switches it to the correct API path. Once set, `deploymentMethod` and
slot-related inputs (`deployToSlotOrASE`, `slotName`, `resourceGroupName`) are
no longer valid for that branch and had to be removed — mixing them back in
causes input-conflict errors.

### Key takeaway

The SCM hostname pattern (`<app>-<random-suffix>.scm.<region>.azurewebsites.net`)
is a reliable visual tell for Flex Consumption apps before you even check the
portal — the randomized suffix doesn't appear on classic plans.

## Tech stack

- Azure DevOps YAML pipelines (multi-repo: app repo + shared template repo)
- .NET 8 Isolated Azure Functions
- Azure Functions **Flex Consumption** plan (Linux)
- Key Vault + Managed Identity for secrets (no Variable Groups)
