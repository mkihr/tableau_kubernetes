# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains a Kubernetes StatefulSet configuration for deploying Tableau Server in a containerized environment. The deployment is designed for a single-node installation with persistent storage and custom initialization scripts.

## Architecture

### StatefulSet Configuration (`tableau.yaml`)

The deployment is structured as a Kubernetes StatefulSet with the following key components:

**Resource Allocation:**
- Main container: 8 CPU cores, 128GB memory
- Init container: 1 CPU core, 12GB memory
- Designed for nodes labeled with `app: tableau` and `stage: int`

**Storage Architecture:**
- **Primary Data Volume** (`tableau-sts`): Pre-existing PVC `tableau-sts-tableau-0` for `/var/opt/tableau` (commented out in volumeClaimTemplates)
- **Backup Volume** (`tableau-backup`): 5GB PVC mounted at `/backup` and `/srv`
- **Bootstrap Volume**: NFS-backed PVC (`nfs-bootstrap-pvc`) for bootstrap files
- **Machine ID**: EmptyDir volume for `/etc/machine-id` generation

**Initialization Flow:**
1. Init container (`set-volume-permissions-and-cleanup`) runs as privileged to:
   - Optionally clear data directory when `SCRATCH=true` environment variable is set
   - Set ownership to unprivileged Tableau UID/GID on all mount points
   - Clear bootstrap directory
   - Generate `/etc/machine-id` from `MACHINE_ID` environment variable if not exists
2. Main Tableau container starts after permissions are set

**Configuration Management:**
- ConfigMap `tableau`: General configuration and environment variables
- ConfigMap `tableau-docker-configfile`: Docker config JSON mounted at `/docker/config/config.json`
- ConfigMap `tableau-startup`: Post-init command script (executable)
- ConfigMap `tableau-scripts`: Backup, cleanup, and configuration scripts
- Secret `tableau`: Sensitive credentials
- ConfigMap `tableau-license`: Offline activation licensing file

**Health Checks:**
- Readiness probe: `/docker/server-ready-check` (20min delay, 60s interval)
- Liveness probe: `/docker/alive-check` (90min delay, 60s interval)

**Networking:**
- Fixed IP allocation via annotation: `ovn.kubernetes.io/ip_pool: 192.168.2.38/32`
- Service name: `tableau-sts`

## Key Configuration Files Expected

The deployment expects these external ConfigMaps and Secrets to exist:

1. **ConfigMap `tableau`**: Environment variables including `UNPRIVILEGED_TABLEAU_UID`, `UNPRIVILEGED_TABLEAU_GID`, `MACHINE_ID`
2. **ConfigMap `tableau-docker-configfile`**: Contains `config.json` for Tableau Docker configuration
3. **ConfigMap `tableau-startup`**: Contains `post_init_command` script
4. **ConfigMap `tableau-scripts`**: Contains operational scripts:
   - `backup-schedule.sh`, `backup.sh`, `cleanup.sh`
   - `pgsql-auth.sh`, `hba-config.sh`, `reconfigure.sh`
   - Configuration files: `backup-schedule.conf`, `hba-config.conf`
5. **ConfigMap `tableau-license`**: Contains `LICENSE_FILE` (OfflineActivationLicensingAtrs.zip)
6. **Secret `tableau`**: Sensitive credentials for Tableau

## Deployment Commands

Apply the StatefulSet:
```bash
kubectl apply -f tableau.yaml
```

View StatefulSet status:
```bash
kubectl get statefulset tableau
kubectl describe statefulset tableau
```

View pod logs:
```bash
kubectl logs tableau-0 -c container-0
kubectl logs tableau-0 -c set-volume-permissions-and-cleanup
```

Check pod status and events:
```bash
kubectl get pod tableau-0
kubectl describe pod tableau-0
```

Delete and redeploy (preserves PVCs due to retention policy):
```bash
kubectl delete statefulset tableau
kubectl apply -f tableau.yaml
```

## Important Notes

- The StatefulSet uses `persistentVolumeClaimRetentionPolicy: Retain` - PVCs are not deleted when the StatefulSet is deleted or scaled down
- The primary data PVC (`tableau-sts-tableau-0`) is pre-created and referenced directly, not managed by volumeClaimTemplates
- Setting `SCRATCH=true` environment variable will DELETE ALL DATA in `/mnt` during init - use with extreme caution
- The container runs with service account `useroot` and init container is privileged for permission management
- Image registry is `reg.its.zz` (private registry) - requires `ecp-secret` for image pull authentication
