[back to main README](../README.md)

---
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

---
[back to main README](../README.md)
