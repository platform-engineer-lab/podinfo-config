# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Delivery pipeline

```mermaid
flowchart LR
    subgraph github["GitHub"]
        SRC["sample-service\nsource + Dockerfile + CI"]
        CFG["sample-service-config\ndry Helm chart"]
        ADD["platform-addons\nroles/<role>/"]
        APP["platform-apps\nregistry/*.yaml"]
    end

    GHCR[("GHCR\nghcr.io/…/sample-service:<sha>")]

    subgraph mgmt["management cluster"]
        AC(["Argo CD"])
        HY["Source\nHydrator"]
        GP["gitops-\npromoter"]
    end

    DEV(["dev spoke"])
    PROD(["prod spoke"])

    SRC -->|"CI: build + push :sha"| GHCR
    SRC -->|"CI: PR bump image.tag"| CFG
    ADD -->|"App-of-Apps"| AC
    APP -->|"cd-apps ApplicationSet"| AC
    CFG -->|"dry source HEAD"| HY
    HY -->|"push env/dev-next\nenv/prod-next"| CFG
    CFG -->|"env/dev · env/prod"| AC
    GP -->|"merge env/*-next → env/*"| CFG
    AC -->|"sync"| DEV
    AC -->|"sync"| PROD
    GHCR -.->|"pull"| DEV
    GHCR -.->|"pull"| PROD
    DEV -->|"argocd-health ✓\nunlocks prod"| GP
```

## Architecture

`podinfo-config` holds the Helm values for podinfo. It is the `$values` source referenced by `platform-apps/registry/podinfo.yaml` — the `cd-apps` ApplicationSet reads that registry entry and generates `podinfo-dev` and `podinfo-prod` Argo CD Applications, each combining the upstream podinfo Helm chart with values from this repo.

- `values/default-values.yaml` — base values shared across all envs
- `values/dev-values.yaml` — dev overrides (1 replica, green UI)
- `values/prod-values.yaml` — prod overrides (2 replicas, blue UI)

## Key conventions

- **Never use `destination.server`** — always `destination.name` (`dev` or `prod`).
- Value file paths are referenced from `platform-apps/registry/podinfo.yaml` as `$values/values/<file>` — if you rename or move files, update that registry entry too.
