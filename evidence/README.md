# Preuves d'exécution anonymisées

Ces fichiers contiennent des extraits sélectionnés des deux sessions de travaux pratiques. Les sorties ne sont pas générées par un simulateur ; elles proviennent des résultats de terminal transmis pendant la réalisation du lab. Les titres, commentaires et séparateurs sont éditoriaux. Certaines lignes sont omises pour la lisibilité.

| Fichier | Contenu |
| --- | --- |
| [01-pods-services.txt](01-pods-services.txt) | Cluster, Pod, scaling, self-healing, Service, endpoints et métriques |
| [02-declarative-network.txt](02-declarative-network.txt) | YAML, rollout, incident d'image, rollback, DNS, Ingress et validation HTTP |

## Anonymisation

- Nom de la machine remplacé par `lab-node` ; prompts shell simplifiés.
- IP du nœud, Pods et Services remplacées par des adresses fictives dans les blocs réservés à la documentation (RFC 5737). Elles ne sont pas à utiliser telles quelles pour reproduire les tests.
- Horodatages absolus, identifiants de runtime, empreintes, chemins temporaires et détails non nécessaires écartés des extraits. Les durées relatives sont conservées.
- Les noms génériques de ressources et les suffixes Kubernetes sont conservés afin de suivre le remplacement des Pods et les révisions.
- Aucun fichier de credentials, certificat, secret ou kubeconfig n'est inclus.

Il s'agit d'une trace de réalisation personnelle, pas d'une certification externe ou d'une garantie de reproductibilité bit à bit.
