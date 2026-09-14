# Nextcloud
- Nextcloud is a personal cloud platform that provides browser-based access to files.
- It integrates *Collabora Online* for in-browser document editing and automatically opens certain file types with Collabora.

## NFS as External Storage
The NFS share can be connected through Nextcloud's *External Storage* app, configured in the
Nextcloud admin UI after deployment. This means:
- Files stay on the NFS in their original location and directory structure.
- Files can be created, edited, and deleted directly on the NFS at any time, bypassing Nextcloud.
- When opening a folder in Nextcloud, it performs a lightweight scan and updates its index
  automatically - no manual rescan or scheduled job is needed.
- The index can therefore be stale between direct NFS edits and the next Nextcloud browser session, 
  which is acceptable for the present use case (single user, no parallel access).

## NFS as Internal Storage
The NFS share can alternatively be mounted as Nextcloud's primary *Internal Storage*.
This means:
- Nextcloud takes ownership over the mounted directories.
- Accessing files on the NFS manually, bypassing Nextcloud, wouldn't be recognized by Nextcloud
  and thus its index would be out of sync.
- This would require periodically re-scans of the mounted NFS.

## Architecture

In this setup Nextcloud acts as one client of the NFS share among potentially others. 
Thus, the NFS is mounted as *External Storage*. Files are never moved or reorganised by Nextcloud. 
If Nextcloud were removed, all files would remain on the NFS in their original structure.

```
NFS Share  <──── directly accessible at any time (no lock-in)
    │
    │  mounted as External Storage inside Nextcloud
    ▼
Nextcloud  ──── browser access from any device
    │
    └──► Collabora  ──── opens office documents in the browser for editing
```

## Database

Nextcloud uses the built-in **SQLite** database (`internalDatabase.enabled: true`).

MariaDB or PostgreSQL would be necessary for multi-user setups or high write concurrency.
Since this is a single-user homelab with no parallel file operations, SQLite is sufficient
and avoids the operational overhead of a separate database pod and its own secrets.

## TLS and Ingress

TLS is terminated by Traefik at the ingress. Nextcloud itself receives plain HTTP from
Traefik and never handles TLS directly. `phpClientHttpsFix` is enabled so that Nextcloud's
PHP layer generates `https://` links correctly despite receiving plain HTTP internally.

## Secrets

Admin credentials are stored in `secret-superuser.yaml` (SOPS-encrypted). The TLS certificate
is stored in `secret-tls.yaml`. Neither file contains plaintext credentials in the repository.

## Collabora Integration

Collabora is deployed as a separate application under `apps/base/collabora/`. The connection
between Nextcloud and Collabora is configured in the Nextcloud admin UI under
**Settings → Office**, by entering the in-cluster URL of the Collabora service.

## Background jobs (cronjob)                                                                                                                                                               
Nextcloud relies heavily on background tasks (cleanup, notifications, previews). 
Without this, things silently degrade over time:                                                          
````yaml
  cronjob:
    enabled: true
    type: cronjob
````
Using type: 'cronjob' (a Kubernetes CronJob) is cleaner than the sidecar type for a single-replica homelab setup.
In the current setup, the cronjob isn't enabled yet, since Nextcloud uses the external storage only.
If this leads to some trouble in the future, warnings in the Nextcloud dashboard should appear and the cronjobs could be configured.
                       