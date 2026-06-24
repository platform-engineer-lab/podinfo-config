# podinfo-config

Helm values for [podinfo](https://github.com/stefanprodan/podinfo), delivered to dev and prod spokes via the `cd-apps` ApplicationSet in [platform-control-plane](https://github.com/platform-engineer-lab/platform-control-plane).

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

## Repository layout

```
values/
  default-values.yaml   base Helm values shared across all envs
  dev-values.yaml       dev-specific overrides (1 replica, green UI)
  prod-values.yaml      prod-specific overrides (2 replicas, blue UI)
```

## How it fits in

`platform-apps/registry/podinfo.yaml` registers podinfo with the `cd-apps` ApplicationSet and points `$values` at this repo. The ApplicationSet generates `podinfo-dev` and `podinfo-prod` Argo CD Applications — each a multi-source Helm release combining the upstream podinfo chart with values from this repo.

To change podinfo configuration, edit the values files here and push — Argo CD picks up the change on next sync.
