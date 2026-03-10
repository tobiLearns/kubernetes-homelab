# HomeLab with k3s
The k3s cluster for my *Kubernetes HomeLab Course*.
_____________________________________________________

## GitOps with Flux

This cluster is managed by [Flux](https://fluxcd.io/), a GitOps operator that runs inside the cluster and continuously reconciles its state with this repository. There is no manual `kubectl apply` — every change is made by editing files in the repo and pushing to `main`.

### How it works

1. Flux watches the Git repository via a `GitRepository` source object.
2. `Kustomization` objects in `clusters/staging/` define which paths in the repo Flux should apply to the cluster, and at what interval.
3. Flux renders the Kustomize overlays and applies the resulting manifests. If the cluster drifts from the desired state, Flux corrects it automatically.
4. Secrets are encrypted with SOPS before being committed. Flux decrypts them during reconciliation using an age key stored in the cluster as a Kubernetes Secret (`sops-age`).

### Repository structure

```
clusters/staging/    # Flux entrypoint — Kustomization objects pointing at app paths
apps/base/           # Base manifests, shared across environments
apps/staging/        # Staging overlays — environment-specific config and secrets
```

_____________________________________________________

## Helm Charts in Flux
- Add certain Helm Repository to the cluster:
  ````yaml
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: <name-of-the-helm-repo>
    namespace: <namespace-name>
  spec:
    interval: 24h
    url: <URL-of-the-helm-repo>
  ````
- Add HelmRelease to the cluster:
  ````yaml
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: <name-of-the-helm-release>
    namespace: <namespace-name>
  spec:
    interval: 30m
    chart:
      spec:
        chart: <name-of-the-helm-chart>
        version: "<version>"

        sourceRef:
          kind: HelmRepository
          name: <name-of-the-helm-repo>
          namespace: <namespace-name>
        interval: 12h
    install:
      crds: Create
    upgrade:
      crds: CreateReplace
    driftDetection:
      mode: enabled
      ignore:
        - paths: ["</path/to/dir-to-ignore/>"]
    values:
      <valueKey>: <value>
  ````
  - CRDs of the helm release can automatically be installed and upgraded.
  - Set desired options (deviating from default values) in the `values`-object
- An example usage of a Helm chart in Flux can be found in the `monitoring`-directory of this repo.
_____________________________________________________

## Save handling of Secrets
### SOPS
Secrets can be encrypted with [SOPS](https://github.com/getsops/sops) using an *age* key and committed to the repository.
#### Setup
- Installation of age and sops on Ubuntu:
    ````bash
    sudo apt install age
    SOPS_VERSION=$(curl -s https://api.github.com/repos/getsops/sops/releases/latest | grep tag_name | cut -d'"' -f4)
    curl -Lo sops.deb "https://github.com/getsops/sops/releases/download/${SOPS_VERSION}/sops_${SOPS_VERSION#v}_amd64.deb"
    sudo dpkg -i sops.deb
    sops --version
    ````
- Generation of age key pair and export of the public key for later usage:
    ````bash
    age-keygen -o age.agekey
    cat age.agekey
    export AGE_PUBLIC=<age_public_key>
    echo $AGE_PUBLIC
    ````
- Generation of secret `sops-age`, which will be used for automatic decryption of other secrets by Flux:
    ````bash
    cat age.agekey | k create secret generic sops-age --namespace=flux-system --from-file=age.agekey=/dev/stdin
    k get secrets -n flux-system
    ````
  **ATTENTION:** This secret MUST NOT be committed to your repo!!!  
  It contains the unencrypted private key of the age-key. 
  Flux uses the private key, contained in this secret, to decrypt the SOPS-encrypted secrets, which are commited to the repo.  
  When the cluster shall be deployed onto a new machine, the file `age.agekey` has to be copied securely to this machine and this `sops-age`-secret again has to be generated manually.   
- Set up Flux-Kustomization for automatic decryption of secrets:
  - The following block in `clusters/staging/apps.yaml` (or here in the app-specific file) tells Flux, how to decrypt secrets:
    ````yaml
    decryption:
      provider: sops
      secretRef:
        name: sops-age
    ````
    
#### Encrypt a Secret
- Generation of a secret (example for user credentials):
    ````bash
    kubectl create secret generic test-secret --from-literal=user=<user_name> --from-literal=password=<user_password> --dry-run=client -o yaml > test-secret.yaml
    k get secrets -A
    vi test-secret.yaml
    echo "<base64_value_from_secret>" | base64 -d
    ````
  The generated secret contains base64-encoded values. These can be decoded to compare as shown above.
- Encryption of the secret values with SOPS:
    ````bash
    sops --age=$AGE_PUBLIC --encrypt --encrypted-regex '^(data|stringData)$' --in-place test-secret.yaml
    ````
  This encrypted secret can be committed safely to a public repo.

- SOPS-configuration file:
  For a more comfortable use of the sops-command, the following SOPS-config file `clusters/staging/.sops.yaml` can be added:
  ````yaml
  creation_rules:
    - path_regex: .*.yaml
      encrypted_regex: ^(data|stringData)$
      age: <age_public_key>
  ````
    - Using this file with the `sops`-command didn't work for me, yet.

__________________________________

## Access to Applications via Cloudflare-Tunnel

To make a locally hosted application securely accessible from the internet, this setup uses a Cloudflare Tunnel. A Cloudflare Tunnel creates an outbound-only connection from a host inside your network to Cloudflare's edge — no open inbound ports or public IP required. The `cloudflared` daemon running on your host establishes the tunnel and proxies traffic to local services. Cloudflare routes incoming requests for your domain through the tunnel to the target service, while your network remains fully shielded behind your firewall.

### Set up Cloudflared
#### Connection Pre-Checks
- Perform [Connectivity pre-checks](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/troubleshoot-tunnels/connectivity-prechecks/):
  - Test DNS with your current resolver:
    ````bash
    dig A us-region1.v2.argotunnel.com
    dig AAAA us-region1.v2.argotunnel.com
    dig A us-region2.v2.argotunnel.com
    dig AAAA us-region2.v2.argotunnel.com
    ````
      - worked also for region1/2, but with lower TTL (Time-To-Live value, indicating the lifetime of the DNS resolvers cache)
  - Test network connectivity:
    ````bash
    nc -uvz -w 3 198.41.192.167 7844
        Connection to 198.41.192.167 7844 port [udp/*] succeeded!
    nc -vz -w 3 198.41.192.167 7844
        nc: connect to 198.41.192.167 port 7844 (tcp) timed out: Operation now in progress
    ````
    - This example output shows a problem, arising due to the firewall configuration of the router:
      - Outbound UDP is allowed, but TCP on port 7844 is blocked or inspected.
      cloudflared will only be able to connect using quic. If you force http2 in your configuration while TCP is blocked, the tunnel will fail.
      To resolve: Either allow TCP on your local network firewall on port 7844 or stop forcing http2 to allow cloudflared to connect over QUIC instead.
      Refer to the Protocol parameter documentation for more information.

#### Install cloudflared
- Add cloudflare gpg key:
  ````bash
  sudo mkdir -p --mode=0755 /usr/share/keyrings
  curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
  ````

- Add this repo to your apt repositories:
  ````bash
  echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
  ````

- install cloudflared:
  ````bash
  sudo apt-get update && sudo apt-get install cloudflared
  ````
- Authenticate cloudflared:
  ````bash
  cloudflared tunnel login
  ````
- This creates a file ~/.cloudflared/cert.pem

#### Create cloudflare tunnel
- Create a cloudflare tunnel:
  ````bash
  cloudflared tunnel create <tunnel-name>
  ````
  This also creates a credentials file in `~/.cloudflared/`.
- Create config file `~/.cloudflared/config.yaml`:
  ````yaml
  url: http://localhost:9090
  tunnel: <tunnel-ID>
  credentials-file: /home/<USER>/.cloudflared/<tunnel-ID>.json
  ````
- Create a CNAME record pointing to `<UUID>.cfargotunnel.com`:
  ````bash
  cloudflared tunnel route dns <tunnel-name> <subdomain>.<domain-at-cloudflare>
  ````
  - Or manually in cloudflare account:
    - Add DNS record CNAME:
      - Under "DNS/Records" add record with type "CNAME"
      - Field "name" represents the later subdomain, under which the tunnel is accessible
      - Field "target" has to be `<tunnel-ID>.cfargotunnel.com`


## Applications

### Linkding
[Linkding](https://github.com/sissbruecker/linkding) is a self-hosted bookmark manager, exposed to the internet via a Cloudflare Tunnel.

#### Structure
```
apps/base/linkding/          # Namespace, storage, deployment, service, superuser secret
apps/staging/linkding/       # Staging overlay: tunnel credentials, cloudflared deployment and config
clusters/staging/
  linkding-app.yaml          # Flux Kustomization — deploys apps/staging/linkding
```

#### Create a Kubernetes Service for linkding
- If not already present, add a service.yaml to linkding app:
  ````yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: linkding
    namespace: linkding
  spec:
    ports:
      - port: 9090
    selector:
      app: linkding
    type: ClusterIP
  ````
- Add service to kustomize.yaml

#### Setup cloudflared to connect with the tunnel
- Create Kubernetes secret for the tunnel credentials:
  ````bash
  kubectl -n linkding create secret generic tunnel-credentials --from-file=credentials.json=~/.cloudflared/<tunnel-ID>.json
  ````

- create cloudflared deployment:
  - Add deployment `cloudflared.yaml` to staging folder.
    - Adapt tunnel name
    - Adapt hostname
- Add deployment to `kustomize.yaml`


#### Cloudflared config: Secret instead of ConfigMap

The cloudflared `config.yaml` is stored as a SOPS-encrypted Kubernetes **Secret** rather than a ConfigMap. The main reason is to avoid publishing the Cloudflare tunnel hostname in the repository. The Secret is mounted as a volume into the cloudflared Deployment, identically to how a ConfigMap would be mounted.

**Alternative considered:** splitting the config into a ConfigMap (non-sensitive settings) and a Secret (hostname only), combined at runtime via an init container. This was rejected because it adds significant complexity (init container, shell script, emptyDir volume) for a small, mostly static config file. Encrypting the whole file is simpler.
