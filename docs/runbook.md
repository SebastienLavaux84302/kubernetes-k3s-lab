# Exercices et runbook de diagnostic

Ce guide reprend les opérations réalisées sur le lab. Les commandes sont adaptées pour être rejouées depuis n'importe quel contexte `kubectl` en précisant `-n lab`. Ne les exécuter que dans un environnement de test dédié.

## TP 01 — Pod autonome

```bash
kubectl apply -f manifests/namespace.yaml
kubectl -n lab run nginx-test --image=nginx:alpine
kubectl -n lab wait --for=condition=Ready pod/nginx-test --timeout=120s
kubectl -n lab get pod nginx-test -o wide
kubectl -n lab describe pod nginx-test
kubectl -n lab exec -it nginx-test -- sh
# Dans le conteneur : hostname; cat /etc/os-release; uname -r; ps; exit
kubectl -n lab logs nginx-test
kubectl -n lab port-forward pod/nginx-test 8080:80
```

Depuis un autre terminal, `curl http://127.0.0.1:8080/`. Arrêter le port-forward avec Ctrl+C, puis supprimer le Pod autonome :

```bash
kubectl -n lab delete pod nginx-test
```

## TP 02 et 03 — Deployment, scaling, self-healing, Service

Sur un namespace ne contenant pas encore le Deployment :

```bash
kubectl -n lab create deployment nginx --image=nginx:alpine --replicas=3
kubectl -n lab rollout status deployment/nginx --timeout=120s
kubectl -n lab get deploy,rs,pods -o wide
kubectl -n lab scale deployment nginx --replicas=5
kubectl -n lab rollout status deployment/nginx --timeout=120s
kubectl -n lab scale deployment nginx --replicas=3
kubectl -n lab rollout status deployment/nginx --timeout=120s
```

Dans un terminal, observer `kubectl -n lab get pods -w`. Dans un autre :

```bash
POD=$(kubectl -n lab get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl -n lab delete pod "$POD"
kubectl -n lab expose deployment nginx --port=80 --target-port=80 --type=ClusterIP
kubectl -n lab describe svc nginx
kubectl -n lab get endpointslices
```

Le ReplicaSet recrée un Pod ; son nom et son IP peuvent changer. Le Service conserve son ClusterIP tant qu'il n'est pas recréé. La suppression d'un Pod autonome ne provoque pas cette recréation.

## TP 04 — Gestion déclarative

```bash
kubectl -n lab create deployment nginx --image=nginx:alpine --replicas=3 --dry-run=client -o yaml
kubectl diff -f manifests/nginx/deployment.yaml
kubectl diff -f manifests/nginx/service.yaml
kubectl apply --dry-run=server -f manifests/nginx/deployment.yaml -f manifests/nginx/service.yaml
kubectl apply -f manifests/nginx/deployment.yaml -f manifests/nginx/service.yaml
kubectl -n lab rollout status deployment/nginx --timeout=120s
kubectl -n lab get rs
```

`kubectl diff` retourne 1 quand des différences existent, ce n'est pas en soi une erreur. Lors du premier `apply` d'une ressource créée impérativement, l'annotation `last-applied-configuration` manquante a été ajoutée automatiquement.

Une readiness probe a brièvement rencontré `connection refused` pendant le démarrage d'un nouveau Pod. Le rollout a ensuite réussi sans modification supplémentaire : une sonde de readiness peut échouer transitoirement avant que l'application soit prête.

## TP 05 — Incident volontaire et restauration

À exécuter après le TP 04, sur le Deployment sain :

```bash
kubectl -n lab rollout history deployment/nginx
kubectl -n lab set image deployment/nginx nginx=nginx:version-inexistante
kubectl -n lab get pods -w
```

Arrêter le watch après l'apparition de `ErrImagePull` ou `ImagePullBackOff` (Ctrl+C). Identifier le Pod en erreur, puis :

```bash
kubectl -n lab get events --sort-by=.metadata.creationTimestamp
kubectl -n lab describe pod NOM_DU_POD_EN_ERREUR
kubectl -n lab rollout undo deployment/nginx
kubectl -n lab rollout status deployment/nginx --timeout=120s
kubectl -n lab get pods
kubectl diff -f manifests/nginx/deployment.yaml
```

**Cause observée :** le registre a renvoyé `NotFound` pour le tag inexistant. Les trois anciens Pods étaient toujours prêts. `ImagePullBackOff` signale que Kubernetes espace les tentatives après un échec de récupération de l'image.

**Restauration observée :** rollback réussi, trois Pods prêts, aucun écart affiché par `kubectl diff`.

Le rollback ne met pas à jour l'annotation `last-applied-configuration`, comme l'a indiqué `kubectl`. Dans une gestion déclarative, restaurer également la configuration versionnée et la réappliquer si nécessaire :

```bash
kubectl apply -f manifests/nginx/deployment.yaml
```

## TP 06 — DNS et Ingress

```bash
kubectl -n lab run debug --image=busybox:1.36 --restart=Never -it --rm -- sh
```

Dans BusyBox :

```sh
nslookup nginx
nslookup nginx.lab.svc.cluster.local
wget -qO- http://nginx
cat /etc/resolv.conf
exit
```

Le nom court a produit plusieurs réponses NXDOMAIN pour des suffixes de recherche non pertinents, mais le FQDN correct a été résolu et la requête HTTP a abouti. Le FQDN est préférable pour lever cette ambiguïté lors d'un diagnostic.

```bash
kubectl get ingressclass
kubectl apply -f manifests/nginx/ingress.yaml
kubectl -n lab describe ingress nginx
NODE_IP=192.0.2.10 # Remplacer par l'adresse réelle du nœud
curl -I --resolve "nginx.lab.test:80:${NODE_IP}" http://nginx.lab.test/
kubectl -n lab top pods
```

## Réflexes de diagnostic

| Symptôme | Vérifications prioritaires |
| --- | --- |
| ImagePullBackOff | Image/tag, registre, droits, DNS, connectivité ; `describe pod`, événements |
| Pod Pending | Événements du Scheduler, requests, nœuds, taints et PVC |
| CrashLoopBackOff | `kubectl logs POD --previous`, code de sortie, config et liveness |
| OOMKilled | Last State dans `describe`, limite RAM et métriques |
| Service inaccessible | Labels, sélecteur, EndpointSlices, targetPort et readiness |
| Ingress inaccessible | Host HTTP, classe, contrôleur, Service et firewall |

Les lignes Pending, CrashLoopBackOff et OOMKilled sont des pistes de diagnostic à approfondir ; aucun incident de ces types n'a été volontairement simulé à ce stade, hormis la phase Pending liée au pull d'image.

## Nettoyage (facultatif)

La commande suivante supprime toutes les ressources du namespace de test, pas uniquement Nginx. Ne l'exécuter que si le namespace `lab` est entièrement dédié aux exercices et ne contient aucune donnée à conserver.

```bash
kubectl delete namespace lab
```
