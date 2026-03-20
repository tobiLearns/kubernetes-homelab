[<p style="text-align:right;">back to main README</p>](./README.md)
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

[<p style="text-align:right;">back to main README</p>](./README.md)
