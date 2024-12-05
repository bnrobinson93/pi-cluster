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

Note that step #4 must be done for ALL secrets

```sh
# 1 Install
brew install aage sops

# 2 Create a age (better than PGP per Flux docs)
## Store this somewhere safe
age-keygen -o age.agekey
cat age.agekey

# 3 Create a dummy secret
kubectl create secret generic test-secret \
--from-literal=user=admin \
--from-literal=password=brad \
--dry-run=client \
-o yaml > test-secret.yaml

# 4 Store the public key in AGE_PUBLIC, then encrypt a test secret
export AGE_PUBLIC=$(cat age.agekey | awk -F: '/public/{print $2}' | tr -d '[[:blank:]]')
sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place test-secret.yaml

# 5 Store the secret + age key in the cluster
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin

# 6 Delete the dummy secret (it was just for fun)
kubectl delete secret test-secret

# 7 Add the one we actually care about
kubectl create secret generic tunnel-credentials \
--from-file=credentials.json=$HOME/.cloudflared/*.json \
--dry-run=client -o=yaml > tunnel-credentials.yaml

# 8 Encrypt that one
export AGE_PUBLIC=$(cat age.agekey | awk -F: '/public/{print $2}' | tr -d '[[:blank:]]')
sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place tunnel-crednetials.yaml
```
