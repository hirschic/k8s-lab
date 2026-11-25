# recipe-depot

The household recipe service ([hirschic/recipe-depot](https://github.com/hirschic/recipe-depot)).

| File | What it is |
|---|---|
| `deployment.yaml` | the app: 1 replica, Recreate (SQLite), image bumped by the recipe-depot repo's CI |
| `service.yaml` | in-cluster address `http://recipe-depot.recipe-depot.svc` (Ingress + Cloudflare Tunnel) |
| `ingress.yaml` | home network: `http://recipes.192.168.4.65.nip.io` (no sign-in) |
| `pvc.yaml` | `/data`: `recipes.sqlite3` and `photos/` |
| `backup.yaml` | nightly 3:45am backup (db snapshot + photos) to the USB drive on the Pi |

Internet: `https://recipes.remainingnameless.org`, a public hostname on PicMap's Cloudflare Tunnel,
behind Cloudflare Access (Google sign-in, "Household" policy). Set up in the Cloudflare dashboard,
not in Git. To let someone in, add them to the Access policy **and** as a test user of the Google
OAuth app.

## One-time setup (not in Git)

Secrets, in namespace `recipe-depot`:

- `ghcr-pull`: a copy of PicMap's image pull Secret
- `recipe-depot-secrets`: `api-key` (the API key; the voice assistant uses the same), `openai-api-key` (optional: photo reading)

Backup folder on the USB drive (the drive itself is set up in `../picmap/README.md`), on the Pi over SSH:

```sh
findmnt /mnt/picmap-backup                              # must show the USB device
sudo mkdir /mnt/picmap-backup/recipe-depot
sudo chown 10001:10001 /mnt/picmap-backup/recipe-depot  # 10001 = the app's user, which the job runs as
```

## Backups

Written to `/mnt/picmap-backup/recipe-depot/`:

- `db/recipes-<date>_<time>Z.sqlite3.gz`: a consistent SQLite snapshot (safe while the app runs), last 30 kept. The time is UTC.
- `photos/`: a mirror of `/data/photos`; only new files are copied each night

```sh
kubectl get cronjob -n recipe-depot                                                    # last run
kubectl create job -n recipe-depot --from=cronjob/recipe-depot-backup recipe-depot-backup-manual   # run one now
kubectl logs -n recipe-depot job/recipe-depot-backup-manual                            # "OK ... N recipes ..."
kubectl delete job -n recipe-depot recipe-depot-backup-manual                          # tidy up
```

### Restore

On the Pi (it has `kubectl`), with the snapshot to restore:

```sh
gunzip -k /mnt/picmap-backup/recipe-depot/db/recipes-<date>.sqlite3.gz      # -> recipes-<date>.sqlite3
POD=$(kubectl get pod -n recipe-depot -l app=recipe-depot -o name | head -1); POD=${POD#pod/}
kubectl exec -n recipe-depot $POD -- mkdir -p /data/import
kubectl cp recipes-<date>.sqlite3 recipe-depot/$POD:/data/import/recipes.sqlite3
kubectl cp /mnt/picmap-backup/recipe-depot/photos recipe-depot/$POD:/data/import/photos
kubectl exec -n recipe-depot $POD -- sh -c 'cp -r /data/import/photos/. /data/photos/ && rm -f /data/recipes.sqlite3-wal /data/recipes.sqlite3-shm && mv /data/import/recipes.sqlite3 /data/recipes.sqlite3 && rm -rf /data/import'
kubectl delete pod -n recipe-depot $POD                                     # restart on the restored data
```
