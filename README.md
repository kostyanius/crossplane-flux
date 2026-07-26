# my-configuration

Crossplane Configuration exposing a simplified `Database` claim API.
Under the hood it creates a Postgres Deployment + Service + PVC as plain
Kubernetes objects via **provider-kubernetes** — no cloud provider needed.
Matches an environment where Crossplane + provider-kubernetes are already
installed (e.g. a kind cluster in WSL).

## Structure

```
my-configuration/
├── crossplane.yaml          # package metadata + dependencies
├── apis/
│   ├── definition.yaml       # XRD: defines the XDatabase/Database API
│   └── composition.yaml      # Composition: XR -> PVC + Deployment + Service
└── examples/
    └── database-claim.yaml   # sample claim (not packaged)
```

## Assumes you already have

```bash
kubectl get providers.pkg.crossplane.io
# provider-kubernetes   True   True   xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.14.1

kubectl get providerconfigs
# kubernetes-provider
```

(the Composition's `providerConfigRef` is hardcoded to `kubernetes-provider`
— rename it in `apis/composition.yaml` if yours is named differently)

## 1. Build the package (needs normal internet access)

```bash
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/main/install.sh | sh
sudo mv crossplane /usr/local/bin/

crossplane xpkg build \
  --ignore="examples/*,.github/workflows/*" \
  --package-file=package.xpkg
```

(`--ignore` needs a pattern per directory level — `.github/*` alone does
**not** match a file nested one level deeper at `.github/workflows/*.yaml`;
that mismatch was the cause of the earlier parse error. There is no
officially supported `.crossplaneignore` file — the `--ignore` flag is
the actual mechanism.)

## 2. Push to a registry

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u <your-username> --password-stdin
crossplane xpkg push ghcr.io/<your-org>/my-configuration:v0.1.0 -f package.xpkg
```

## 3. Install into your cluster

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: my-configuration
spec:
  package: ghcr.io/<your-org>/my-configuration:v0.1.0
```

```bash
kubectl apply -f configuration.yaml
kubectl get configurations
kubectl get configurationrevisions
```

## 4. Try it

```bash
kubectl apply -f examples/database-claim.yaml
kubectl get databases
kubectl get objects.kubernetes.crossplane.io
kubectl get pods -n default        # should see a postgres-* pod
kubectl get svc -n default
```

## Known limitation (fine for a single test instance)

The Deployment/Service `app: postgres` label selector is static, so
creating a second `Database` claim in the same namespace would collide
with the first. For multiple instances, parameterize the label with the
XR name via an extra patch — ask if you want that added.

## Why no tunnels/VPN are needed

Build+push happens from outside, pushing into a public registry. Your
cluster pulls the package and applies it itself — only outbound traffic,
no inbound access into your machine/network required.
