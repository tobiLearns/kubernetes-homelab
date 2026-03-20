# Kubernetes HomeLab
This repository documents the setup of a home lab Kubernetes cluster, 
with a focus on **how** to run a cluster properly rather than **what** runs on it.  
It demonstrates GitOps workflows using Flux, secure secret management with SOPS and age, 
automated dependency updates via Mend Renovate, 
and exposing self-hosted services to the internet through Cloudflare Tunnels, without opening inbound ports.  
The applications deployed here (e.g., Linkding), serve primarily as concrete examples 
to illustrate these patterns and workflows.  
The goal is a reproducible, auditable, and secure cluster setup that follows current best practices.

## Used Workflows and Technologies
- ### [GitOps with Flux](./README_Flux.md)
- ### [Helm Charts in Flux](./README_Helm-Charts-in-Flux.md)
- ### [Automated Image Updates](./README_Automated-Image-Updates.md)
- ### [Save handling of Secrets](./README_Secrets.md)
- ### [Encrypted connection with TLS](./README_Encrypted-Connection.md)
- ### [Access to Applications via Cloudflare-Tunnel](./README_Cloudflare-Tunnel.md)

## Applications
- ### [Linkding](./README_Linkding.md)

