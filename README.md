# Brad's homelab

## Principles

- Homelab is a scratch pad and test bed for production level tinkering
- It makes use of a monorepo structure
  - apps - contains the applilcation manifests (app deployments)
  - infrastructure - contains infrastructure manifestsa
  - clusters - contains the flux cluster monitoring manifests
- Each app and infra maintains a base layer and is patched for production or staging
- CI is maintained via [Flux](https://fluxcd.io)
- Cloudflare is used to provide DNS:
  - To create the tunnel
  ```sh
  sudo apt install cloudflard
  cloudflared tunnel login
  cloudflared tunnel create <tunnel name> # we used "ldpi" for "LinkDing pi"
  # That will create a file in $HOME/.cloudflared/*.json
  # if in a dev container copy that into the repo
  cp $HOME/.cloudflared/*.json .
  ```
  - To create the secret (not secure)
  ```sh
  # This creates the file but doesn't store it as code, which is not what we want long-term
  kubectl create secret generic tunnel-credentials \
  --from-file=credentials.json=$HOME/.cloudflared/*.json
  ```
  - From inside the cloudflare website:
    - Create a new DNS entry on the host
    - Type = CNAME, Name = ldpi, target: <id>.cfargotunnel.com
    - Ensure it says "Proxy Enabled"
  - Then link it to the cluster
    - Add a service that matches app and port to the deployment (9090 + linkding)

## Secret Managements

Secrets will be managed with SOPS, which was initially created by Mozilla.
The CLI encrypts with OpenPGP, AWS, Azure Key Vault, etc.

### Install

```sh
# Install
brew install aage sops

# Create a age (better than PGP per Flux docs)
# https://fluxcd.io/docs/components/source/gitrepositories/#age-encryption
age-keygen -o age.agekey
cat age.agekey
# Store this somewhere safe

# Create a dummy secret
kubectl create secret generic test-secret \
--from-literal=user=admin \
--from-literal=password=brad \
--dry-run=client \
-o yaml > test-secret.yaml

# Store the public key in AGE_PUBLIC, then encrypt a test secret
sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place test-secret.yaml

# Store the secret + age key in the cluster
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin

```
