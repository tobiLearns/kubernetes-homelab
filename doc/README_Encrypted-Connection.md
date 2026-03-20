[back to main README](../README.md)

---
## TLS Certificate
To encrypt the connection to an application in the cluster, a TLS certificate can be used:

1. Generate private key and certificate:
  ````bash
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ./tls.key \
  -out ./tls.crt \
  -subj "/C=US/ST=Detroit/L=Basement/O=Home Lab Heroes Inc./OU=Department of Monitoring/CN=grafana.homelab.net" \
  -addext "subjectAltName=DNS:grafana.homelab.net"
  ````
2. create a Kubernetes secret:
  ````yaml
  kubectl create secret tls grafana-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=monitoring \
  --dry-run=client \
  -o yaml > grafana-tls-secret.yaml
  ````
3. encrypt the secret with SOPS
4. Use the secret in an application (e.g., refer to the secret in the `values` of a Helm release, as done in the `monitoring`-directory.)
5. Don't forget to decrypt the secret in the corresponding Kustomization-file.

---
[back to main README](../README.md)
