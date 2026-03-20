[back to main README](../../../README.md)

---
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

---
[back to main README](../../../README.md)
