# Cluster Infra and applications deployment

## Prerequisites

* You will need a Kubernetes cluster version 1.21 or newer. Any other Kubernetes setup will work as well though.

* In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

* Install the Flux CLI on MacOS or Linux using Homebrew or install the CLI by downloading precompiled binaries using a Bash script:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **infrastructure** dir contains common infra tools such as ingress-nginx and cert-manager
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── production 
│   └── staging
├── infrastructure
│   ├── configs
│   └── controllers
└── clusters
    ├── production
    └── staging
```
## Initialize operation

### Cluster bootstraping

#### 1. Defining credentials

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```
#### 2. Check flux command line

Verify that your staging cluster satisfies the prerequisites with:

```bash
flux check --pre
```

#### 3. Initiate cluster

Set the kubectl context to your staging cluster and bootstrap Flux:

```bash
flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/staging
```

> For other environments just change `--path=clusters/<ENV-NAME>` like `--path=clusters/production`

#### 4. Manual force for implement changes  

```bash
flux reconcile source git flux-system
flux reconcile kustomization apps
```

Other applicable commands are

```bash
flux get kustomizations --watch
watch flux get helmreleases --all-namespaces
```

## Access the Flux UI

To access the Flux UI on a cluster, first start port forwarding with:

```bash
kubectl -n flux-system port-forward svc/weave-gitops 9001:9001
```

Navigate to http://localhost:9001 and login using the username `admin` and the password `flux`.

[Weave GitOps](https://docs.gitops.weave.works/) provides insights into your application deployments,
and makes continuous delivery with Flux easier to adopt and scale across your teams.
The GUI provides a guided experience to build understanding and simplify getting started for new users;
they can easily discover the relationship between Flux objects and navigate to deeper levels of information as required.

![flux-ui-depends-on](.github/screens/flux-ui-depends-on.png)

You can change the admin password bcrypt hash in **infrastructure/controllers/weave-gitops.yaml**:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: weave-gitops
  namespace: flux-system
spec:
  # ...omitted for brevity
  values:
    adminUser:
      create: true
      username: admin
      # bcrypt hash for password "flux"
      passwordHash: "$2a$10$P/tHQ1DNFXdvX0zRGA8LPeSOyb0JXq9rP3fZ4W8HGTpLV7qHDlWhe"
```

To generate a bcrypt hash please see Weave GitOps
[documentation](https://docs.gitops.weave.works/docs/configuration/securing-access-to-the-dashboard/#login-via-a-cluster-user-account). 

Note that on production systems it is recommended to expose Weave GitOps over TLS with an ingress controller and
to enable OIDC authentication for your organisation members.
To configure OIDC with Dex and GitHub please see this [guide](https://docs.gitops.weave.works/docs/guides/setting-up-dex/).

## Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```bash
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```bash
mkdir -p clusters/dev
```

Copy the sync manifests from staging:

```bash
cp clusters/staging/infrastructure.yaml clusters/dev
cp clusters/staging/apps.yaml clusters/dev
```

You could create a dev overlay inside `apps`, make sure
to change the `spec.path` inside `clusters/dev/apps.yaml` to `path: ./apps/dev`. 

Push the changes to the main branch:

```bash
git add -A && git commit -m "add dev cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```bash
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev
```

## Identical environments

If you want to spin up an identical environment, you can bootstrap a cluster
e.g. `production-clone` and reuse the `production` definitions.

Bootstrap the `production-clone` cluster:

```bash
flux bootstrap github \
    --context=production-clone \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production-clone
```

Pull the changes locally:

```bash
git pull origin main
```

Create a `kustomization.yaml` inside the `clusters/production-clone` dir:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - ../production/infrastructure.yaml
  - ../production/apps.yaml
```

Note that besides the `flux-system` kustomize overlay, we also include
the `infrastructure` and `apps` manifests from the production dir.

Push the changes to the main branch:

```bash
git add -A && git commit -m "add production clone" && git push
```

Tell Flux to deploy the production workloads on the `production-clone` cluster:

```bash
flux reconcile kustomization flux-system \
    --context=production-clone \
    --with-source 
```
