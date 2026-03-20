[<p style="text-align:right;">back to main README</p>](./README.md)
## Automated Image Updates
### Image Updates with *Mend Renovate*
- Mend Renovate is an automated dependency update tool that helps you update dependencies in your code. When Renovate runs in your repo, it looks for references to dependencies (both public and private) and, if newer versions are available, Renovate can create pull requests to update the versions. [[from the Renovate-Docs]](https://docs.mend.io/renovate/latest/)
- To setup renovate in the cluster, a Kubernetes Cronjob can be used.
- Renovate needs an access token to the corresponding git-repository in order to be allowed to generate merge requests.
  This token can be stored as encrypted secret:
  ````bash
  kubectl create secret generic renovate-container-env \
  --from-literal=RENOVATE_TOKEN=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 \
  --dry-run=client \
  -o yaml > renovate-container-env.yaml
  ````
  And encrypted with SOPS.
- In the repository root directory a file `renovate.json` is needed, which defines, which files in the repository shall be checked for new versions.

[<p style="text-align:right;">back to main README</p>](./README.md)
