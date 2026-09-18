# Script de demo live ArgoCD

Masterclass EAZYTraining. Duree cible du bloc demo : 45 min.
Environnement : VM VirtualBox "ArgoCD-GitOps" (Ubuntu), ArgoCD deja installe,
cluster Kubernetes en place.

Convention : tout ce qui suit est a taper dans le Terminal Emulator de la VM,
sauf mention "dans l'UI".

--------------------------------------------------------------------------------

## Bloc 0 : Verifications avant de commencer (a faire AVANT la session, 5 min)

Objectif : eviter toute mauvaise surprise devant le public.

Cluster OK :
```
kubectl get nodes
```
Les noeuds doivent etre Ready.

ArgoCD tourne :
```
kubectl get pods -n argocd
```
Tous les pods doivent etre Running.

Recuperer le mot de passe admin de l'UI :
```
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
Notez ce mot de passe. L'utilisateur est : admin

Ouvrir l'acces a l'UI (laisser tourner dans un terminal dedie) :
```
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
Puis dans le navigateur de la VM : https://localhost:8080
(accepter l'avertissement de certificat auto-signe)

Connexion de la CLI argocd (dans un autre terminal) :
```
argocd login localhost:8080 --username admin --password "LE_MOT_DE_PASSE" --insecure
```

Verifier que le repo est pousse sur Git et que repoURL dans apps/demo-app.yaml
pointe bien vers VOTRE repo.

--------------------------------------------------------------------------------

## Bloc 1 : Tour du proprietaire (5 min)

Montrer l'UI vide (ou presque). Expliquer les 3 zones : Applications, settings, repos.

Cote terminal, montrer qu'il n'y a encore rien de deploye :
```
kubectl get ns
kubectl get all -n demo
```
Le namespace demo n'existe pas encore. C'est normal : rien n'a ete synchronise.

Message a faire passer : ArgoCD ne fait rien tant qu'on ne lui a pas dit
"voici un repo, voici un chemin, maintiens cet etat sur le cluster".

--------------------------------------------------------------------------------

## Bloc 2 : Creer l'Application (8 min)

Montrer le fichier qui decrit l'Application :
```
cat apps/demo-app.yaml
```
Expliquer les 3 blocs essentiels :
- source  : ou est l'etat desire (repoURL + path + targetRevision)
- destination : ou on deploie (cluster + namespace)
- syncPolicy : comment on synchronise (ici manuel au depart)

Appliquer l'Application :
```
kubectl apply -f apps/demo-app.yaml
```

Retour dans l'UI : l'application demo-web apparait.
Statut attendu : Missing / OutOfSync.
Expliquer : ArgoCD a compris ce qu'il faut faire, mais n'a encore rien applique,
parce qu'on est en sync manuel.

--------------------------------------------------------------------------------

## Bloc 3 : Premier sync (5 min)

Dans l'UI : cliquer sur l'application, puis sur SYNC, puis SYNCHRONIZE.
(Equivalent CLI, a montrer aussi : )
```
argocd app sync demo-web
```

Observer l'arbre de ressources se construire dans l'UI :
Application -> Deployment -> ReplicaSet -> 2 Pods, plus le Service.

Cote terminal :
```
kubectl get all -n demo
```
On voit le Deployment, le ReplicaSet, les 2 pods, le Service.

Message : Git a produit un etat reel sur le cluster. Statut : Synced / Healthy.

--------------------------------------------------------------------------------

## Bloc 4 : Drift par Git (7 min)

Scenario : on change l'etat desire dans Git. C'est la bonne facon de modifier.

Editer manifests/demo/deployment.yaml et passer replicas de 2 a 3.
Puis committer et pousser :
```
git add manifests/demo/deployment.yaml
git commit -m "demo: passage a 3 replicas"
git push
```

Dans l'UI : cliquer sur REFRESH.
ArgoCD detecte que Git dit 3 mais que le cluster est a 2 : statut OutOfSync.
Montrer le diff dans l'UI (bouton APP DIFF).

Synchroniser (SYNC dans l'UI, ou argocd app sync demo-web).
Le 3e pod apparait.
```
kubectl get pods -n demo
```

Message : la modification part TOUJOURS de Git. On voit qui a change quoi,
et on peut relire l'historique.

--------------------------------------------------------------------------------

## Bloc 5 : Drift manuel, puis self-heal (8 min)

Scenario : quelqu'un modifie le cluster a la main, hors de Git. Le cas a eviter.

Modifier directement le cluster :
```
kubectl -n demo scale deployment demo-web --replicas=1
kubectl get pods -n demo
```
On passe a 1 pod. Dans l'UI, apres REFRESH : statut OutOfSync.
ArgoCD voit l'ecart entre Git (3) et le cluster (1), mais ne corrige pas encore,
parce que le self-heal n'est pas active.

Activer l'automatisation EN DIRECT. Deux facons, montrer l'une ou l'autre :

Via la CLI :
```
argocd app set demo-web --sync-policy automated --self-heal --auto-prune
```

Ou via un patch kubectl :
```
kubectl -n argocd patch application demo-web --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":true,"prune":true}}}}'
```

Refaire le drift manuel :
```
kubectl -n demo scale deployment demo-web --replicas=1
```
Attendre quelques secondes, rafraichir l'UI et le terminal :
```
kubectl get pods -n demo
```
ArgoCD ramene tout seul le nombre de pods a 3. C'est le self-heal.

Message fort : avec le self-heal, le cluster ne peut plus deriver en silence.
Git gagne toujours.

--------------------------------------------------------------------------------

## Bloc 6 : Rollback par Git (5 min)

Scenario : la derniere modif etait mauvaise, on veut revenir en arriere.

Voir l'historique :
```
git log --oneline
```
Annuler le dernier commit (le passage a 3 replicas) :
```
git revert HEAD --no-edit
git push
```
Comme le self-heal est actif, ArgoCD applique le revert automatiquement
et revient a 2 replicas.
```
kubectl get pods -n demo
```

Message : le rollback n'est pas une manip speciale. C'est juste un commit Git.
L'etat du cluster suit l'historique Git.

--------------------------------------------------------------------------------

## Bloc 7 : Nettoyage (facultatif, apres la session)

```
argocd app delete demo-web --yes
```
ArgoCD supprime les ressources gerees (prune actif). Verifier :
```
kubectl get all -n demo
```

--------------------------------------------------------------------------------

## Plan B si le port-forward ou le reseau lache

- Si l'UI est inaccessible : toute la demo se fait en CLI avec argocd app ...
  et kubectl. Prevoir de connaitre les commandes par coeur.
- Si le push Git echoue (reseau) : utiliser un repo Git LOCAL sur la VM.
  Creer un repo bare et pointer repoURL dessus :
  ```
  git init --bare ~/argocd-demo.git
  ```
  puis dans le repo de travail : git remote add origin ~/argocd-demo.git
  et dans demo-app.yaml : repoURL: file:///home/UTILISATEUR/argocd-demo.git
  (le repo-server d'ArgoCD doit pouvoir y acceder ; sur cette VM mono-poste
  le plus simple reste un repo distant type GitHub prepare a l'avance).
- Toujours avoir teste TOUTE la sequence une fois, la veille, sur la VM.

--------------------------------------------------------------------------------

## Antiseche des commandes (a garder sous les yeux)

```
# UI
kubectl -n argocd port-forward svc/argocd-server 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# App
kubectl apply -f apps/demo-app.yaml
argocd app sync demo-web
argocd app set demo-web --sync-policy automated --self-heal --auto-prune
argocd app get demo-web
argocd app delete demo-web --yes

# Observation
kubectl get all -n demo
kubectl get pods -n demo -w
```
