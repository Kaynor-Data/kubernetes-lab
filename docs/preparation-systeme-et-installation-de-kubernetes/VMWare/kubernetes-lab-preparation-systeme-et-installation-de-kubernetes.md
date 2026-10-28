# Kubernetes Lab - Préparation système et installation de Kubernetes

## 1. Présentation

Ce document décrit les étapes qui suivent la configuration de l'adressage IP statique des trois machines virtuelles Ubuntu Server 24.04 sur VMware Workstation.

L'objectif est de préparer les serveurs, de vérifier les conditions nécessaires au fonctionnement de Kubernetes, puis de déployer un cluster composé d'un Control Plane et de deux Workers.

### Objectifs

- Vérifier la connectivité réseau entre les trois machines.
- Contrôler le pare-feu et les ports nécessaires.
- Désactiver le swap conformément à la configuration retenue pour kubelet.
- Préparer les paramètres système Linux.
- Installer et configurer le runtime de conteneurs `containerd`.
- Installer `kubeadm`, `kubelet` et `kubectl`.
- Initialiser le Control Plane.
- Installer un plugin réseau CNI.
- Joindre les deux Workers au cluster.
- Vérifier le fonctionnement du cluster.

## 2. Architecture cible

| Machine | Nom d'hôte | Rôle | Adresse IP |
|---|---|---|---|
| VM 1 | `ubuntu-server-01` | Control Plane | 192.168.189.10/24 |
| VM 2 | `ubuntu-worker-01` | Worker 1 | 192.168.189.11/24 (à confirmer) |
| VM 3 | `ubuntu-worker-02` | Worker 2 | 192.168.189.12/24 (à confirmer) |

Les adresses IP doivent correspondre au plan d'adressage documenté dans le guide réseau précédent.

### Environnement technique

- VMware Workstation
- Ubuntu Server 24.04 LTS
- Netplan
- OpenSSH, si l'administration à distance est utilisée
- UFW, si le pare-feu est activé
- containerd
- kubeadm
- kubelet
- kubectl
- Un plugin réseau CNI

---

## 3. Vérifier la connectivité réseau

Cette étape doit être terminée avant la préparation de Kubernetes.

Sur chaque machine, exécuter :

```bash
hostname
ip -br address
ip route
getent hosts ubuntu-server-01
getent hosts ubuntu-worker-01
getent hosts ubuntu-worker-02
```

Depuis le Control Plane, tester les deux Workers :

```bash
# ping -c 4 k8s-worker1
# ping -c 4 k8s-worker2

ping -c 4 ubuntu-server-02
ping -c 4 ubuntu-server-03
```

Depuis chaque Worker, tester également le Control Plane et l'autre Worker.

### Résultat attendu

- Chaque machine possède une adresse IP stable.
- Les trois noms d'hôte sont résolus vers les bonnes adresses.
- Les routes sont cohérentes avec le réseau VMware.
- Les machines peuvent communiquer entre elles.

Un échec du ping ne signifie pas nécessairement que toute communication est impossible : ICMP peut être bloqué par un pare-feu. Il faut alors vérifier la route et les ports TCP nécessaires.

---

## 4. Vérifier et configurer le pare-feu

### 4.1. Option A - UFW désactivé

Vérifier l'état :

```bash
sudo ufw status verbose
```

Si le résultat indique `Status: inactive`, UFW n'applique actuellement pas de règles de filtrage.

Cela ne garantit pas l'absence d'autres règles de pare-feu : les règles `nftables`, celles de l'hôte ou celles du réseau peuvent également intervenir.

**Avantage :** configuration initiale plus simple pour diagnostiquer un laboratoire isolé.

**Limite :** aucune protection supplémentaire n'est fournie par UFW lorsqu'il est inactif.

Ne pas activer ou désactiver UFW sans comprendre les conséquences, notamment si la machine est accessible depuis un réseau physique ou par SSH.

### 4.2. Option B - UFW activé avec des règles adaptées

Avant d'activer UFW, vérifier les règles existantes et, si SSH est utilisé, autoriser l'accès SSH depuis le réseau d'administration.

Exemple pour un réseau de laboratoire :

```bash
sudo ufw allow from 192.168.189.0/24 to any port 22 proto tcp
```

Cette règle est un exemple. Remplacer le sous-réseau par le réseau d'administration réellement utilisé.

Puis examiner les besoins du cluster avant d'autoriser les ports Kubernetes.

| Port ou plage | Protocole | Usage principal |
|---|---|---|
| 6443 | TCP | API Server Kubernetes |
| 2379-2380 | TCP | etcd, principalement sur le Control Plane |
| 10250 | TCP | Kubelet |
| 10257 | TCP | Controller Manager, selon la configuration |
| 10259 | TCP | Scheduler, selon la configuration |
| 30000-32767 | TCP/UDP selon le service | NodePort, uniquement si utilisé |

Ces ports ne doivent pas être ouverts indistinctement à tous les réseaux. Restreindre les sources et les destinations selon le rôle des nœuds et la documentation du CNI.

Les ports utilisés par le plugin réseau choisi doivent également être pris en compte.

**Important :** les règles de pare-feu du Control Plane et des Workers ne sont pas nécessairement identiques.

Après validation des règles :

```bash
sudo ufw status numbered
```

N'activer UFW qu'après avoir vérifié que les règles permettent toujours l'accès d'administration et les communications nécessaires au cluster.

### 4.3. Option C - Examiner directement les règles système

Pour examiner les règles nftables :

```bash
sudo nft list ruleset
```

Pour rechercher les services qui écoutent :

```bash
sudo ss -lntup
```

Cette méthode permet de diagnostiquer des blocages qui ne sont pas gérés par UFW.

---

## 5. Vérifier et désactiver le swap

Kubelet utilise des paramètres liés à la mémoire et au swap. Pour une installation kubeadm classique, la désactivation du swap est la méthode de préparation habituelle, sauf si une configuration Kubernetes explicitement compatible avec le swap est retenue.

### 5.1. Examiner l'état actuel

Sur les trois machines :

```bash
swapon --show
free -h
```

Si `swapon --show` ne retourne aucune ligne, aucun espace swap n'est actuellement actif.

### 5.2. Option A - Désactivation temporaire

Pour désactiver les espaces swap actifs jusqu'au prochain redémarrage :

```bash
sudo swapoff -a
```

Vérifier :

```bash
swapon --show
free -h
```

**Avantage :** méthode simple pour préparer ou tester la machine.

**Limite :** le swap peut être réactivé au redémarrage si la configuration persistante n'est pas modifiée.

### 5.3. Option B - Désactivation persistante dans `/etc/fstab`

Examiner d'abord le fichier :

```bash
cat /etc/fstab
```

Si une entrée correspond à un fichier ou une partition swap, sauvegarder le fichier :

```bash
sudo cp -a /etc/fstab /etc/fstab.backup
```

Modifier ensuite le fichier :

```bash
sudo nano /etc/fstab
```

Commenter uniquement l'entrée swap concernée en ajoutant `#` au début de la ligne.

Ne pas commenter les montages nécessaires au démarrage du système.

Désactiver le swap actuellement actif :

```bash
sudo swapoff -a
```

Puis vérifier :

```bash
swapon --show
```

**Avantage :** le swap déclaré dans `/etc/fstab` ne sera plus activé automatiquement.

**Limite :** cette opération ne couvre pas nécessairement tous les mécanismes d'activation possibles. Il faut contrôler la configuration réellement utilisée par la machine.

### 5.4. Option C - Vérifier une configuration d'activation supplémentaire

Si le swap réapparaît après un redémarrage, examiner les services et configurations qui peuvent l'activer, notamment les unités systemd liées au swap :

```bash
systemctl list-units --type=swap
systemctl list-unit-files --type=swap
```

Ne désactiver un service ou une unité qu'après avoir identifié précisément son rôle.

### Validation

Sur les trois serveurs :

```bash
swapon --show
```

Pour la configuration kubeadm classique retenue dans ce laboratoire, aucune ligne ne doit indiquer un swap actif.

---

## 6. Préparer les paramètres système Linux

Les paramètres requis dépendent du runtime et du plugin réseau sélectionnés. Les opérations ci-dessous constituent la préparation courante d'un cluster kubeadm avec containerd.

### 6.1. Vérifier l'identité des machines

Sur chaque VM :

```bash
hostname
ip -br address
ip route
cat /sys/class/dmi/id/product_uuid
```

Les noms d'hôte, adresses IP et identifiants système doivent être cohérents et distincts entre les trois machines.

### 6.2. Charger les modules nécessaires

Créer un fichier de configuration :

```bash
sudo nano /etc/modules-load.d/k8s.conf
```

Contenu courant :

```text
overlay
br_netfilter
```

Charger les modules :

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Vérifier :

```bash
lsmod | grep -E 'overlay|br_netfilter'
```

### 6.3. Configurer les paramètres réseau du noyau

Créer le fichier :

```bash
sudo nano /etc/sysctl.d/k8s.conf
```

Ajouter :

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

Appliquer les paramètres :

```bash
sudo sysctl --system
```

Vérifier le transfert IPv4 :

```bash
sysctl net.ipv4.ip_forward
```

Résultat attendu :

```text
net.ipv4.ip_forward = 1
```

Ces paramètres sont une configuration courante. Les besoins IPv6 et les paramètres complémentaires dépendent de l'architecture et du plugin CNI.

---

## 7. Installer le runtime de conteneurs containerd

Kubernetes a besoin d'un runtime CRI pour exécuter les conteneurs.

### 7.1. Option A - Installer containerd depuis les dépôts Ubuntu

Examiner les paquets disponibles :

```bash
sudo apt update
apt-cache policy containerd
```

Si une version appropriée est disponible, l'installer :

```bash
sudo apt install -y containerd
```

**Avantage :** installation simple et intégration avec les dépôts Ubuntu.

**Limite :** la version proposée dépend des dépôts et des mises à jour disponibles.

Vérifier la version et le service :

```bash
containerd --version
sudo systemctl status containerd --no-pager
```

### 7.2. Option B - Installer containerd depuis une source maintenue par son éditeur

Une autre possibilité consiste à utiliser les paquets officiels du projet containerd ou ceux d'une distribution maintenue par son éditeur.

Avant de choisir cette méthode, vérifier la compatibilité de la version avec Ubuntu 24.04, l'architecture de la machine et les instructions officielles.

Éviter de mélanger plusieurs sources de paquets sans nécessité.

### 7.3. Générer une configuration containerd

Sauvegarder une configuration existante avant de la remplacer :

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Cette commande génère une configuration par défaut. Si un fichier de configuration existe déjà et contient des paramètres spécifiques, sauvegarder son contenu avant de le régénérer.

Dans `/etc/containerd/config.toml`, vérifier que le runtime runc utilise le gestionnaire de cgroups systemd. Dans les configurations containerd 1.x et 2.x, la section concernée peut différer.

Pour une configuration containerd 1.x classique, le paramètre se trouve notamment dans la section `plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options` :

```toml
SystemdCgroup = true
```

Ne pas copier cette ligne à un emplacement arbitraire. La structure exacte doit correspondre à la version installée.

Redémarrer et activer le service :

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager
```

### 7.4. Validation

```bash
containerd --version
sudo systemctl is-active containerd
```

Le service doit être actif.

---

## 8. Installer kubeadm, kubelet et kubectl

Les trois machines doivent utiliser une version compatible de Kubernetes. La procédure officielle utilise le dépôt de paquets correspondant à la branche mineure choisie.

### 8.1. Choisir une version

Avant l'installation :

- consulter la documentation officielle de kubeadm ;
- sélectionner une branche mineure prise en charge ;
- utiliser la même branche mineure sur les trois machines ;
- éviter de copier un dépôt ou une clé de version ancienne sans vérification.

Documentation :  
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### 8.2. Préparer les prérequis

Sur les trois machines, vérifier que les paquets requis sont disponibles :

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

Suivre ensuite les instructions officielles pour ajouter la clé et le dépôt correspondant à la version Kubernetes retenue.

### 8.3. Installer les composants

Une fois le dépôt configuré :

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Le maintien des paquets évite les mises à jour accidentelles de ces composants ; il faut toutefois prévoir les mises à niveau Kubernetes de manière contrôlée.

Vérifier les versions :

```bash
kubeadm version
kubectl version --client
kubelet --version
```

Répéter la procédure sur le Control Plane et les deux Workers.

---

## 9. Initialiser le Control Plane

Cette opération ne doit être effectuée qu'après validation du réseau, de containerd et des prérequis Kubernetes.

Sur `k8s-master`, préparer la commande en utilisant :

- l'adresse IP statique réelle du Control Plane ;
- le CIDR de pods prévu ;
- un plugin CNI compatible avec ce CIDR.

Exemple de structure :

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<IP_DU_MASTER> \
  --pod-network-cidr=<CIDR_DES_PODS>
```

Les valeurs entre chevrons sont des paramètres à remplacer.

Le CIDR des pods ne doit pas entrer en conflit avec le réseau des VM ou d'autres réseaux accessibles au cluster.

À la fin de l'initialisation, suivre les instructions affichées pour configurer `kubectl` et conserver la commande `kubeadm join` dans un endroit sécurisé.

Ne pas publier le jeton de jointure dans le dépôt GitHub.

---

## 10. Installer le plugin réseau CNI

Kubernetes nécessite un plugin réseau de pods pour assurer leur connectivité.

### Options possibles

| Plugin | Quand l'envisager |
|---|---|
| Calico | Réseau de pods, politiques réseau et configurations variées |
| Flannel | Laboratoire simple avec des besoins réseau classiques |
| Cilium | Fonctions réseau avancées et observabilité, selon les besoins |

Le choix doit être effectué avant l'initialisation du cluster afin d'utiliser un CIDR compatible et une procédure d'installation adaptée.

Installer le plugin choisi en suivant sa documentation officielle et la version compatible avec Kubernetes.

Après installation :

```bash
kubectl get pods -A
kubectl get nodes -o wide
```

Les composants du réseau doivent devenir opérationnels.

---

## 11. Joindre les Workers au cluster

Sur le Control Plane, récupérer la commande de jointure générée par kubeadm si elle n'a pas été conservée.

La commande prend généralement la forme suivante :

```bash
sudo kubeadm join <IP_DU_MASTER>:6443 \
  --token <JETON> \
  --discovery-token-ca-cert-hash sha256:<EMPREINTE>
```

Remplacer les paramètres par les valeurs générées pour le cluster.

Exécuter cette commande sur `k8s-worker1`, puis sur `k8s-worker2`.

Ne pas partager le jeton ou l'empreinte de jointure dans un dépôt public.


Une fois les trois adresses confirmées, ajouter sur chaque machine :

192.168.189.10 ubuntu-server-01
192.168.189.11 ubuntu-worker-01
192.168.189.12 ubuntu-worker-02
---

## 12. Valider le cluster

Depuis le Control Plane :

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

### Résultat attendu

- Trois nœuds apparaissent dans le cluster.
- Le Control Plane est enregistré.
- Les deux Workers sont enregistrés.
- Les pods système sont opérationnels.
- Les trois nœuds atteignent l'état `Ready` une fois le CNI et les autres composants opérationnels.

Si un nœud reste `NotReady`, examiner les événements et les pods système avant de poursuivre.

---

## 13. Dépannage

### containerd est inactif

```bash
sudo systemctl status containerd --no-pager
sudo journalctl -u containerd -n 100 --no-pager
```

### Le swap est encore actif

```bash
swapon --show
cat /etc/fstab
systemctl list-units --type=swap
```

### Un Worker ne rejoint pas le cluster

```bash
ip route
getent hosts k8s-master
sudo systemctl status kubelet --no-pager
sudo journalctl -u kubelet -n 100 --no-pager
```

Vérifier également l'accès TCP au port 6443 du Control Plane, les règles de pare-feu et les paramètres de la commande de jointure.

### Les nœuds restent `NotReady`

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl describe node k8s-worker1
```

Examiner le runtime, le CNI et les événements Kubernetes.

---

## 14. État d'avancement

| Étape | Statut |
|---|---|
| Installation des trois VM | Terminée |
| Configuration IP statique | À valider |
| Vérification de la communication réseau | À réaliser |
| Vérification du pare-feu | À réaliser |
| Désactivation et vérification du swap | À réaliser |
| Paramètres système Linux | À réaliser |
| Installation de containerd | À réaliser |
| Installation de kubeadm, kubelet et kubectl | À réaliser |
| Initialisation du Control Plane | À réaliser |
| Installation du CNI | À réaliser |
| Jointure des deux Workers | À réaliser |
| Validation des trois nœuds | À réaliser |

## 15. Références officielles

- Ubuntu Server : https://ubuntu.com/server/docs
- Netplan : https://netplan.readthedocs.io/
- Kubernetes - Installation de kubeadm : https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Kubernetes - Création d'un cluster kubeadm : https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- Kubernetes - Ports et protocoles : https://kubernetes.io/docs/reference/networking/ports-and-protocols/
- containerd : https://containerd.io/
- Calico : https://docs.tigera.io/calico/latest
- Flannel : https://github.com/flannel-io/flannel
- Cilium : https://docs.cilium.io/

---

**État du document :** guide de préparation système et de déploiement initial d'un cluster Kubernetes. Les commandes doivent être exécutées après confirmation des adresses IP, du mode réseau VMware, des versions retenues et du plugin CNI choisi.
