# CI/CD Pipeline — End-to-End Test Guide

This guide walks you through verifying every stage of the pipeline is working correctly.

---

## Pre-flight Checks

Before triggering the pipeline, confirm all components are healthy.

**1. Docker is running:**
```powershell
docker info | Select-String "ServerVersion"
or for windows 
docker version --format "Server Version: {{.Server.Version}}"
```
Expect: a version number, no errors.

**2. Kind cluster is up:**
```powershell
kubectl get nodes
```
Expect: `STATUS=Ready`

**3. ArgoCD pods are running:**
```powershell
kubectl get pods -n argocd
```
Expect: all pods in `Running` state.

**4. Self-hosted runner is registered:**
```powershell
kubectl get pods -n actions-runner-system
```
Expect: runner pod in `Running` state. Also check GitHub: repo → Settings → Actions → Runners — should show **Idle**.

**5. App is currently running:**
```powershell
kubectl get pods
```
Expect: `python-test-xxx` pod in `Running 1/1` state.

---

## Stage 1 — Trigger the CI Pipeline

Make a small change to the app and push it:

```powershell
# Open src/app.py and change the message value, e.g.:
# "message": "Hello from the API! v2"

git add src/app.py
git commit -m "test: trigger ci pipeline"
git push origin main
```

---

## Stage 2 — Verify CI Job (Build & Push)

Go to: **GitHub repo → Actions tab**

You should see a workflow run named `ci-cd` triggered.

**CI job steps to confirm:**
- `Checkout code` — green
- `Login to Docker Hub` — green
- `Compute short SHA` — green, outputs a 7-char commit hash
- `Build and push` — green, image pushed to Docker Hub

**Verify the image was pushed to Docker Hub:**

Go to: `https://hub.docker.com/r/asherif310/python-test/tags`

You should see a new tag matching the short SHA of your commit (e.g. `a1b2c3d`).

---

## Stage 3 — Verify Manifest Update Job

Back in GitHub Actions, the `update-manifest` job runs after CI:

**Steps to confirm:**
- `Update Image Tag in Helm values` — green
- `Commit and push changes` — green, commits `chore: update image tag to <sha> [skip ci]`

**Verify `values.yaml` was updated:**
```powershell
git pull
git log --oneline -3
```
Expect: a commit from `github-actions[bot]` updating the image tag.

```powershell
grep "tag:" charts/python-test/values.yaml
```
Expect: `tag: "<new-sha>"`

---

## Stage 4 — Verify CD Job (ArgoCD Sync)

The `cd` job runs on your self-hosted runner and calls ArgoCD.

**In GitHub Actions**, confirm:
- `argocd app sync python-test` — green

**Check ArgoCD synced the new version:**
```powershell
argocd app get python-test
```
Expect:
- `Sync Status: Synced`
- `Health Status: Healthy`
- Image tag in the manifest matches the new SHA

Or check via ArgoCD UI at `https://localhost:8080`.

---

## Stage 5 — Verify New Pod is Running

```powershell
kubectl get pods -w
```

You should see the old pod terminating and a new pod starting:
```
python-test-<old>   1/1   Running     → Terminating
python-test-<new>   0/1   Pending     → Running
```

Confirm the new pod uses the new image:
```powershell
kubectl get pod <new-pod-name> -o jsonpath="{.spec.containers[0].image}"
```
Expect: `asherif310/python-test:<new-sha>`

---

## Stage 6 — Verify the App Responds

**Via port-forward (quickest):**
```powershell
kubectl port-forward svc/python-test 5000:5000
```

```powershell
curl http://localhost:5000/api/v1/health
curl http://localhost:5000/api/v1/details
```

Expected health response:
```json
{"status": "healthy"}
```

Expected details response:
```json
{
  "name": "Sample API",
  "version": "1.0",
  "description": "This is a sample API endpoint.",
  "hostname": "python-test-xxx",
  "message": "Hello from the API! v2"
}
```

The `hostname` field shows the pod name — confirms it's live inside the cluster.
The `message` field should reflect the change you made in Stage 1.

**Via Ingress (end-to-end test):**

Make sure your hosts file has the entry (run PowerShell as Administrator):
```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1  python-test.example.com"
```

Then:
```powershell
curl http://python-test.example.com/api/v1/health
curl http://python-test.example.com/api/v1/details
```

---

## Full Pipeline Summary

```
git push
   ↓
[CI] Build Docker image → push to Docker Hub with short SHA tag
   ↓
[update-manifest] Commit new tag into charts/python-test/values.yaml
   ↓
[CD] argocd app sync → ArgoCD detects new values.yaml → deploys new pod
   ↓
New pod running with updated image → app responds with new code
```

If all 6 stages pass, the full CI/CD pipeline is working correctly.

---

## Quick Troubleshooting

**Pipeline not triggered:**
- Check the push was to `main` branch
- Check the changed file matches the path filter: `src/**` or `.github/workflows/cidc.yaml`

**CD job fails with ArgoCD connection error:**
- Confirm `ARGOCD_SERVER` secret is `argocd-server.argocd.svc.cluster.local`
- Confirm `ARGOCD_AUTH_TOKEN` secret is set and not expired
- Check runner pod logs: `kubectl logs -l app=self-hosted-runner -n actions-runner-system`

**New pod stuck in ImagePullBackOff:**
- Confirm the new tag exists on Docker Hub
- Run: `kubectl describe pod <new-pod-name>` and check Events section

**App returns old response after deployment:**
- Hard refresh the browser (`Ctrl+Shift+R`)
- Confirm the pod was actually replaced: `kubectl get pods` — AGE should be recent
