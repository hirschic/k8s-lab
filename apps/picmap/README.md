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

Everything is in the PVC. Copy it out (stop writes first by not uploading):

```sh
POD=$(kubectl get pod -n picmap -l app=picmap -o name | head -1)
kubectl cp -n picmap ${POD#pod/}:/data ./picmap-backup
```
