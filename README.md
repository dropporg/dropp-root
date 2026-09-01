# Dropp

The Dropp platform, as three repositories tracked here as submodules.

**Heal** is a network monitoring service: it checks the availability,
reachability and latency of configured hostnames, and flags targets that look
filtered or blocked. It runs on Kubernetes clusters that are themselves
declared in Git.

| Submodule | Owns |
| --- | --- |
| [`dropp-heal`](https://github.com/dropporg/dropp-heal) | The application — FastAPI api and worker, Next.js dashboard, migrations, Helm chart, GitHub Actions |
| [`dropp-infra`](https://github.com/dropporg/dropp-infra) | Cluster API cluster definitions, cluster addons, and every cluster's Flux configuration |
| [`dropp-heal-gitops`](https://github.com/dropporg/dropp-heal-gitops) | What each cluster runs: chart versions, image tags, credentials |

New here? Read [HANDSON.md](HANDSON.md).

## Clone

The submodules are not fetched by default.

```bash
git clone --recurse-submodules https://github.com/dropporg/dropp-root.git
cd dropp-root
```

Already cloned without them:

```bash
git submodule update --init --recursive
```

## Working with the submodules

Each submodule is an ordinary repository with its own remote, its own history
and its own CI. This repository only records **which commit** of each one
belongs to a given state of the platform.

A submodule starts in detached HEAD at the recorded commit. Check out a branch
before making changes:

```bash
cd dropp-heal
git checkout main
git pull
```

Commit and push inside the submodule as usual. Then record the new pointer
here:

```bash
cd ..
git add dropp-heal
git commit -m "Bumped - dropp-heal to <short sha>"
```

Useful across all three:

```bash
git submodule status              # the recorded commit of each
git submodule foreach 'git status --short'
git submodule update --remote     # move every pointer to its remote main
```

`git submodule update --remote` changes what this repository records; review
the diff before committing it.

## Layout

```
dropp-root/
├── HANDSON.md            how the three fit together, and how to run it
├── README.md
├── dropp-heal/           submodule
├── dropp-heal-gitops/    submodule
└── dropp-infra/          submodule
```

## Getting it running

In short:

```bash
cd dropp-infra
just doctor       # what is installed and reachable
just prepare      # install anything missing
just bootstrap    # create the management cluster; Git delivers the rest
just status       # watch it converge
```

The full walkthrough, including the GPG key and deploy key you need first, is
in [HANDSON.md](HANDSON.md).

## How a change ships

```
tag in dropp-heal ─► Actions build the image and chart
                  ─► Flux image automation commits the new tag to dropp-heal-gitops
                  ─► Flux in the cluster reconciles and the pods roll
```

CI never deploys. Every change reaches a cluster by being committed.

## Licence

See [LICENSE.md](LICENSE.md).
