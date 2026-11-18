# PicMap

Map of photos taken at Midtown Atlanta's IKE kiosks. Source: github.com/hirschic/PicMap (private).
Image: `ghcr.io/hirschic/picmap`, built for arm64 by GitHub Actions on every push to PicMap's `main`.

| File | What it is |
|---|---|
| `pvc.yaml` | 5 Gi volume at `/data`: SQLite db, photos, login keys |
| `deployment.yaml` | the app (1 replica, `Recreate`, probes on `/healthz`) |
| `service.yaml` | in-cluster address `picmap.picmap.svc:80` → container 8080 |
| `ingress.yaml` | home network: http://picmap.192.168.4.65.nip.io |
| `cloudflared.yaml` | public HTTPS via a Cloudflare Quick Tunnel (random address) |
| `backup.yaml` | nightly 3:30am backup (db snapshot + photos) to a USB drive on the Pi |

## One-time setup (not in Git)

The Google Maps key is a Secret, created by hand so it never lands in this repo:

```sh
kubectl create namespace picmap
kubectl create secret generic picmap-secrets -n picmap --from-literal=google-maps-api-key=AIza...
```

Then register the app with Argo: `kubectl apply -f argocd/picmap.yaml`

## Everyday

- **Public address:** `kubectl logs -n picmap deploy/cloudflared | grep trycloudflare.com`
- **Deploy a new build:** copy the commit SHA from the PicMap Actions run into `image:` in
  `deployment.yaml`, commit, push. Argo rolls it out.
- **Forgot the password:** delete the account row, then `kubectl port-forward -n picmap svc/picmap 8080:80`
  and open http://localhost:8080/Login. First-time setup only works from localhost, never through the
  ingress or tunnel.

## Backups

`backup.yaml` runs every night at 3:30am (Atlanta time). It writes to a USB drive mounted on the
Pi at `/mnt/picmap-backup`:

- `db/picmap-<date>.db.gz`: a consistent SQLite snapshot (safe while the app is running), last 30 kept
- `photos/`: a mirror of `/data/photos`; only new files are copied each night

### One-time: prepare the USB drive (on the Pi, over SSH)

```sh
lsblk -f                                   # find the drive, e.g. sda1. Note its FSTYPE and UUID
# Only if it needs formatting. ERASES THE DRIVE:  sudo mkfs.ext4 -L picmap-backup /dev/sda1
sudo mkdir -p /mnt/picmap-backup
echo 'UUID=<the-uuid>  /mnt/picmap-backup  ext4  defaults,nofail,noatime  0  2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload && sudo mount -a   # nofail: the Pi still boots if the drive is unplugged
findmnt /mnt/picmap-backup                 # must show the USB device, not nothing
sudo touch /mnt/picmap-backup/.picmap-backup-drive   # marker: "this really is the backup drive"
sudo chown -R 1654:1654 /mnt/picmap-backup           # 1654 = the "app" user the job runs as
```

Use ext4 (or another Linux filesystem). FAT/exFAT can't store file owners, so the `chown` won't stick.
The marker file is a safety check. If the drive ever isn't mounted, `/mnt/picmap-backup` is just an
empty folder on the SD card. The job sees no marker and fails instead of quietly backing up onto
the card it's meant to protect.

### Check on it

```sh
kubectl get cronjob,jobs -n picmap                          # last schedule / recent runs
kubectl create job -n picmap --from=cronjob/picmap-backup picmap-backup-manual   # run one now
kubectl logs -n picmap job/picmap-backup-manual             # "OK ... 20 kiosks, ..."
kubectl delete job -n picmap picmap-backup-manual           # tidy up the manual run
```

### Restore

Same swap as the original move from Windows. Copy the files in under a temporary name, swap, then
restart the pod:

```sh
gunzip -k /mnt/picmap-backup/db/picmap-<date>.db.gz        # on the Pi -> picmap-<date>.db
POD=$(kubectl get pod -n picmap -l app=picmap -o name | head -1); POD=${POD#pod/}
kubectl exec -n picmap $POD -- mkdir -p /data/import
kubectl cp picmap-<date>.db picmap/$POD:/data/import/picmap.db
kubectl cp /mnt/picmap-backup/photos picmap/$POD:/data/import/photos
kubectl exec -n picmap $POD -- sh -c 'cp -r /data/import/photos/. /data/photos/ && rm -f /data/picmap.db-wal /data/picmap.db-shm && mv /data/import/picmap.db /data/picmap.db && rm -rf /data/import'
kubectl delete pod -n picmap $POD
```

The login-cookie keys (`/data/keys`) aren't backed up. After a full restore you just log in again.
