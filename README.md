# argocd-demo

Repo de demonstration pour la masterclass ArgoCD (EAZYTraining).
Il contient une petite application web (nginx) deployee sur Kubernetes via ArgoCD,
structuree pour rendre visibles la detection de drift et le self-heal.

## Structure

```
argocd-demo/
  apps/
    demo-app.yaml            Application ArgoCD (le "pointeur" vers les manifestes)
  manifests/
    demo/
      deployment.yaml        Deployment nginx, 2 replicas
      service.yaml           Service ClusterIP
```

## Le principe en une phrase

Le dossier manifests/demo decrit l'etat desire. ArgoCD lit ce dossier depuis Git
et le maintient en permanence sur le cluster. Git est la source de verite.

## Mise en route

1. Creer un repo Git (GitHub, GitLab, ou local) et y pousser ce dossier.
2. Ouvrir apps/demo-app.yaml et remplacer repoURL par l'URL de VOTRE repo.
3. Appliquer l'Application : kubectl apply -f apps/demo-app.yaml
4. Suivre le script de demo (script-demo.md).

## Points de demo prevus

- Premier sync : l'app se deploie depuis Git.
- Drift par Git : on change replicas ou l'image, ArgoCD voit OutOfSync.
- Drift manuel : on modifie le cluster a la main, le self-heal corrige tout seul.
- Rollback : un git revert suffit a revenir en arriere.
# argo-project
