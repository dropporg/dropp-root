# Hands-on

How the three repositories fit together, and how to get from an empty machine to
a running cluster serving the application.

## What this is

**Heal** is a network monitoring service. It checks the availability,
reachability and latency of configured hostnames, and flags targets that look
filtered or blocked. Target configuration and latest state live in MySQL;
historical measurements go to InfluxDB.

It is delivered by GitOps. Nothing is deployed by hand and no CI job ever runs
`kubectl apply`. A change reaches a cluster by being committed.

## The three repositories

| Repo | Holds | Read by |
| --- | --- | --- |
| `dropp-heal` | The application, its Helm chart, and the GitHub Actions that build them | Developers; CI |
| `dropp-infra` | Cluster API cluster definitions, the CNI, and each cluster's Flux configuration | Flux on the management cluster |
| `dropp-heal-gitops` | What each cluster should be running: chart versions, image tags, credentials | Flux inside each workload cluster |

They are separate on purpose. `dropp-heal` says what the software *is*,
`dropp-infra` says what clusters *exist*, and `dropp-heal-gitops` says what a
given cluster *runs right now*. A rollback is a revert in the third, with no
rebuild and no cluster access.

## How code reaches a cluster

```
you ──► git push a tag in dropp-heal
                │
                ▼
        GitHub Actions: test, lint, audit, build
                │
                ├──► ghcr.io/dropporg/heal-<component>:<version>
                └──► dropporg.github.io/dropp-heal   (Helm repo on gh-pages)
                │
                ▼
        Flux image automation notices the new tag
                │
                ▼
        commits the bump to dropp-heal-gitops
                │
                ▼
        Flux in the cluster reconciles the HelmRelease
                │
                ▼
        pods roll
```

CI stops at publishing. Git carries the change the rest of the way.

Tags decide what gets published:

| Tag | Effect |
| --- | --- |
| `api/*`, `worker/*`, `dashboard/*`, `migrate/*` | Build that one component's image |
| `helm/v0.1.0` | Publish chart `0.1.0` |
| `release/*` | The product-wide version, stamped into the chart as `appVersion` |
| a branch or PR | Packages chart `0.0.0-sha.<short>` and publishes nothing |

Component versions are independent of each other and of `release/*`. An api at
`v0.0.1-beta.2` beside a dashboard at `v0.0.1-beta.1` is correct, not drift.

## How a cluster comes to exist

Only the first step is imperative.

1. **Three scripts** create a k3d management cluster, install Cluster API
   (CAPD + cluster-api-k3s), and install Flux. This is the only part anyone
   runs by hand.
2. Flux on the management cluster reads `capi/clusters/`, and **CAPI creates
   the workload cluster's nodes** as Docker containers.
3. `capi/addons/` delivers **Calico** to them as a `ClusterResourceSet`.
4. A Flux `Kustomization` on the management cluster uses the kubeconfig **CAPI
   generated** to reach into the new cluster and **install Flux there**. A
   workload cluster is never bootstrapped by hand.
5. That Flux installs cert-manager and Envoy Gateway, then points at
   `dropp-heal-gitops` for the application.

Creating a cluster and giving it a source of truth are both `git push`.

## Getting it running

Everything below is in `dropp-infra`.

```bash
cd dropp-infra
just doctor         # what is installed, what is reachable
just prepare        # install anything missing
```

You need one thing that is not a tool: the **management GPG key** at
`~/.dropp-heal/mgmt.asc`. Every encrypted file in `dropp-infra` is encrypted to
it, so without it Flux installs and then cannot decrypt anything.

```bash
just bootstrap
```

That creates the management cluster and then waits for Git to deliver the rest.
On a first run it stops and prints an SSH public key: register it as a
**read-only deploy key** on `dropp-infra`, then run it again. It resumes.

When it finishes:

```bash
just status                      # Flux on every cluster
just kubeconfig heal-k3d-dev     # writes ~/.kube/heal-k3d-dev.kubeconfig
```

To reach the application, add the hostnames to `/etc/hosts` pointing at the
Envoy dataplane, or port-forward to it:

| Hostname | Serves |
| --- | --- |
| `heal-dev.heal.local` | The dashboard, and the API under `/api` |
| `influx-dev.heal.local` | The InfluxDB UI |
| `pma-dev.heal.local` | phpMyAdmin |

Tearing down:

```bash
just teardown heal-k3d-dev        # one cluster
just teardown all true            # everything, with its disks
```

## Working on the application

```bash
cd dropp-heal
just infra-up      # mysql + influxdb in containers
just upgrade       # apply migrations
just run           # api on :8000
just run-worker    # monitoring engine on :8001
just run-dashboard # dashboard on :3000
just test          # the suite
just lint          # ruff, exactly as CI runs it
```

On NixOS run these inside `nix-shell`; greenlet needs `libstdc++` on the
library path, which `shell.nix` provides.

To ship a change: merge it, tag the component (`api/v0.0.2`), and let the chain
above carry it. To change *how much* of something runs, or which version, edit
`dropp-heal-gitops` instead — that is configuration, not code.

## Things that will bite you

These each cost real debugging time. They are documented where they matter, but
they are worth knowing before you start.

**Enrolling a cluster is a commit.** Flux reads GitHub, not your working tree.
`heal-k3d-prod` is fully declared and deliberately not enrolled; uncommenting
the two lines does nothing until it is pushed.

**Editing a live cluster does not stick.** Flux reverts hand edits on the next
reconcile, and a `kubectl delete` of a declared object is answered by
recreating it. To remove something, remove it from Git — or suspend the
Kustomization first, which is what `just teardown` does.

**Hostnames must match the Gateway listener.** It is `*.heal.local`, and a
Gateway API wildcard matches exactly one label and never the apex.
`heal-dev.heal.local` works; `heal.local` and `*.dev.heal.local` do not. A
route outside the pattern attaches silently and never serves, with no error
anywhere.

**`sops` reads its config from your working directory**, not from the file you
are editing. Encrypt from inside the right directory or pass `--config`, or the
file ends up encrypted to a key the target cluster does not have.

**Service link environment variables collide with the config prefix.** The
kubelet sets `HEAL_API_PORT` for the `heal-api` Service, the app reads config
from the `HEAL_API_` prefix, and start-up dies in pydantic before logging
anything. Every pod in the chart sets `enableServiceLinks: false`. It only
appears once the Service exists, so a first install can pass and every rollout
after it fail.

**The `$imagepolicy` markers are load-bearing.** In `dropp-heal-gitops`, the
comment after each `image.tag` is what Flux rewrites. Move or reformat that
line and updates stop, with no error.

## Where to read more

- `dropp-infra/README.md` — the cluster chain, Calico, SOPS key split
- `dropp-heal/README.md` — running the app, the API, the dashboard
- `dropp-heal-gitops/SOPS-GUIDE.md` — adding and rotating secrets
