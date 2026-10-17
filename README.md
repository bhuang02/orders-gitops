# Orders GitOps — first deployment (no Istio or Rollouts yet)

Use this as the **root contents** of a NEW public GitHub repository named `bhuang02/orders-gitops`.
The first lesson is plain Helm + Argo CD + Kubernetes Deployment. Do not combine this with the older `02-orders-gitops` starter chart yet: that one needs Istio/Argo Rollouts/Prometheus CRDs.

## Preconditions

- Jenkins has **successfully pushed** an image `ghcr.io/bhuang02/orders-api:<FULL_GIT_SHA>` to GHCR.
- The GHCR package is public for this lab, or Kubernetes has an imagePullSecret.
- Argo CD is installed and healthy in the `argocd` namespace on K3s.
- Create an empty public GitHub repository named `orders-gitops` without a README.

## On your Mac

1. Unzip the package; open the `orders-gitops-basic` directory.
2. Replace `REPLACE_WITH_PUBLISHED_SHA` in `charts/orders-api/values-staging.yaml` with the full SHA of the pushed image.
3. Use the following commands from `orders-gitops-basic/`:

```bash
git init -b main
git add .
git commit -m "Add Orders API Helm chart and Argo CD Application"
git remote add origin git@github.com:bhuang02/orders-gitops.git
git push -u origin main
```

4. Validate with Helm where it is installed, e.g. on EC2 after cloning this repo:

```bash
helm lint charts/orders-api -f charts/orders-api/values-staging.yaml
helm template orders-staging charts/orders-api -f charts/orders-api/values-staging.yaml --namespace orders-staging
```

## On EC2 (public Git repository)

```bash
sudo k3s kubectl -n argocd get pods
sudo k3s kubectl apply -f https://raw.githubusercontent.com/bhuang02/orders-gitops/main/argocd/orders-staging.yaml
sudo k3s kubectl -n argocd get applications
sudo k3s kubectl -n orders-staging get deploy,rs,pods,svc
sudo k3s kubectl -n orders-staging rollout status deployment/orders-staging
```

To reach the app, start on **EC2** and keep running:

```bash
sudo k3s kubectl -n orders-staging port-forward service/orders-staging 18082:80
```

Then start on **Mac** and keep running (replace SSH values):

```bash
ssh -i ~/.ssh/YOUR_EC2_KEY.pem -N -L 18082:127.0.0.1:18082 ubuntu@YOUR_EC2_PUBLIC_IP
```

Finally on your Mac:

```bash
curl http://localhost:18082/
curl http://localhost:18082/healthz
```

To prove GitOps, edit `replicaCount` in `charts/orders-api/values-staging.yaml` from 1 to 2, commit and push. Argo CD will eventually sync and the Deployment should have two replicas. Alternatively, change the image tag to a **different image already published to GHCR**.

## Common failures

- `ImagePullBackOff`: verify GHCR visibility and image tag; `kubectl describe pod` will show pull errors.
- `InvalidSpecError` or `Manifest generation error`: verify Git URL, branch, path, values filename and Helm rendering.
- `Sync failed`: inspect the Argo CD Application events, permissions, and namespace existence.
- `CrashLoopBackOff`: inspect container logs; verify your Go app actually listens on port 8080 and implements `/healthz`.

Don't expose admin interfaces publicly. This is a single-EC2 lab, not production. Automated GitOps commits by Jenkins are a later step; initially change Git by hand to see the process.
