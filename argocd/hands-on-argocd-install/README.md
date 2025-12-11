# Hands-on lab ArgoCD install on Kubernetes

## Preparing environment
- Check kubernetes cluster nodes. In this lab, I have a kubernetes cluster with: 01 master and 02 worker.
```bash
kubectl get nodes -owide
```
![Alt text](./images/k8s-nodes.png)

- Check ingress service on cluster
```bash
kubectl get svc -n ingress-controller
```
![Alt text](./images/ingress-controller.png)


