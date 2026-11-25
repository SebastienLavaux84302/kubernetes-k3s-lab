# Installation et préparation du lab

## Environnement observé

| Élément | Valeur |
| --- | --- |
| OS | AlmaLinux 10.3 |
| Kubernetes | K3s v1.37.2+k3s1 |
| Runtime | containerd 2.3.7-k3s1 |
| Topologie | 1 nœud, control plane + workloads |
| Accès | Connexion Internet pour les images et l'installation |

Ces versions correspondent aux sorties du lab, pas à une recommandation de version actuelle. Utiliser une machine de test dédiée, idéalement avec un snapshot. Pour une nouvelle installation, vérifier les [prérequis K3s](https://docs.k3s.io/installation/requirements) et les versions prises en charge.

## Installation K3s

Exemple de préparation (adapter les paquets et le firewall à l'environnement ; toutes les étapes préparatoires ne sont pas documentées par les traces du lab) :

```bash
sudo -i
dnf install -y curl vim bash-completion kernel-modules-extra
modprobe overlay
modprobe br_netfilter
modprobe vxlan
sysctl -w net.ipv4.ip_forward=1
```

Vérifier SELinux, swap, les réseaux existants et la politique de filtrage avant d'installer. Ne pas désactiver les protections sans justification. Si firewalld est actif, suivre la documentation K3s pour le CNI et les ports requis ; limiter les ouvertures aux interfaces et sources nécessaires. Éviter d'exposer l'API ou VXLAN à un réseau non fiable.

Commande d'installation réalisée dans le lab (le canal stable est évolutif) :

```bash
curl -fsSL https://get.k3s.io -o /root/install-k3s.sh
vim /root/install-k3s.sh
sh /root/install-k3s.sh
systemctl status k3s
```

Pour une version déterminée, utiliser `INSTALL_K3S_VERSION` avec une release vérifiée. Le paquet `k3s-selinux` a été installé par l'installateur ; cela ne suffit pas à démontrer que le runtime applique SELinux aux conteneurs.

### Incident rencontré : kubectl introuvable

L'installation avait créé le lien `/usr/local/bin/kubectl -> k3s`, mais `/usr/local/bin` était absent du `PATH` de root.

```bash
echo "$PATH"
ls -l /usr/local/bin/{k3s,kubectl,crictl,ctr}
/usr/local/bin/kubectl get nodes
export PATH="/usr/local/bin:$PATH"
```

Ajouter ce dernier export dans `~/.bashrc` s'il manque, puis recharger le fichier. Le binaire était présent ; aucune réinstallation n'était nécessaire.

### Validation

```bash
kubectl version
kubectl get nodes -o wide
kubectl get pods -A
kubectl get svc -A
kubectl get storageclass
kubectl top nodes
crictl ps
crictl images
```

Une erreur HTTP 503 de l'API Metrics Server a été observée au tout début du démarrage. Les contrôles suivants ont confirmé le retour à la normale : Pod Metrics Server prêt et `kubectl top nodes` fonctionnel. Les Jobs d'installation Helm étaient `Completed`, ce qui est attendu. Le provisionneur `local-path` était présent ; aucun PVC n'a encore été créé.

## Contexte et autocomplétion

```bash
kubectl create namespace lab
kubectl config set-context --current --namespace=lab
kubectl config get-contexts
```

Optionnel dans `~/.bashrc` :

```bash
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
```

Les commandes du README créent le namespace via YAML à la place de la commande impérative. Ne jamais publier le kubeconfig administrateur (`/etc/rancher/k3s/k3s.yaml`), les jetons de jointure ou les clés privées.
