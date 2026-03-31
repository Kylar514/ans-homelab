# argocd

Installs ArgoCD via Helm onto the k3s cluster using the kubernetes.core.helm
Ansible module. ArgoCD serves as the GitOps engine for all application
deployments after initial cluster bootstrap.

## Dependencies

- `kubernetes.core` Ansible collection
- `python3-kubernetes` on azir-01
- kubeconfig on azir-01 pointing at kaiju-01

#

# Initial Access (manual, one-time)

ArgoCD UI is not exposed externally until the IngressRoute helm chart is
deployed. Use port-forward for initial setup:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

Then access `https://localhost:8080` — username is `admin`.

## Notes

- ArgoCD takes over all application deployments after bootstrap
- Ansible owns infrastructure, ArgoCD owns applications
- First helm chart to deploy via ArgoCD is the Traefik IngressRoute for ArgoCD
  itself
- See k3s-homelab repo for helm charts
