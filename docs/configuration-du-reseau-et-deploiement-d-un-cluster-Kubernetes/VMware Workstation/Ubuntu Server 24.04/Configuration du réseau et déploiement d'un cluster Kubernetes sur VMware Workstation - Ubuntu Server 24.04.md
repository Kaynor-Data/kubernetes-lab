# Configuration du réseau et déploiement d'un cluster Kubernetes sur VMware Workstation - Ubuntu Server 24.04

## 1. Présentation du projet

Ce projet consiste à déployer un cluster Kubernetes composé de trois machines virtuelles Ubuntu Server 24.04 sur VMware Workstation.

L'objectif est de mettre en place une infrastructure fonctionnelle, de la configuration réseau jusqu'à l'administration du cluster Kubernetes.

La configuration réseau constitue la première étape du projet. Chaque machine virtuelle doit disposer d'une adresse IP stable afin de communiquer avec les autres nœuds du cluster.

### Objectifs

- Configurer le réseau virtuel dans VMware Workstation.
- Identifier les interfaces réseau, les adresses IP et les passerelles depuis le terminal Ubuntu.
- Configurer des adresses IP statiques avec Netplan.
- Configurer les noms d'hôte des trois serveurs.
- Vérifier la communication réseau entre les machines.
- Installer et configurer Kubernetes avec `kubeadm`.
- Initialiser le nœud de contrôle et joindre les deux workers.
- Vérifier le fonctionnement du cluster.

### Environnement technique

| Composant | Technologie |
|---|---|
| Hyperviseur | VMware Workstation |
| Système d'exploitation | Ubuntu Server 24.04 LTS |
| Configuration réseau | Netplan |
| Administration | Terminal Linux |
| Orchestration de conteneurs | Kubernetes |
| Installation du cluster | kubeadm |
| Runtime de conteneurs | containerd |
| Interface d'administration du cluster | kubectl |

---

## 2. Architecture du cluster

Le laboratoire comprend trois machines virtuelles.

| Machine virtuelle | Nom d'hôte | Rôle | Adresse IP prévue |
|---|---|---|---|
| VM 1 | `k8s-master` | Control Plane | À définir |
| VM 2 | `k8s-worker1` | Worker 1 | À définir |
| VM 3 | `k8s-worker2` | Worker 2 | À définir |

### Rôle du Control Plane

Le Control Plane gère le cluster Kubernetes.

Il comprend notamment les composants suivants :

- API Server ;
- Scheduler ;
- Controller Manager ;
- etcd.

### Rôle des Workers

Les workers exécutent les applications conteneurisées sous forme de pods.

Ils communiquent avec le Control Plane pour recevoir les instructions nécessaires à l'exécution des charges de travail.

### Principe de communication

Les trois machines doivent pouvoir communiquer sur le réseau utilisé par le cluster.

Chaque nœud doit posséder :
- une adresse IP unique ;
- un nom d'hôte unique ;
- une connectivité réseau fonctionnelle ;
- une adresse IP stable, adaptée à l'architecture retenue.

Les adresses IP définitives seront choisies après vérification du réseau VMware.

---

## 3. Prérequis

Avant de commencer, vérifier les éléments suivants :

- VMware Workstation est installé sur l'ordinateur hôte.
- Les trois machines virtuelles Ubuntu Server 24.04 sont installées.
- Chaque machine virtuelle possède une carte réseau virtuelle.
- L'accès au terminal Ubuntu est fonctionnel.
- L'utilisateur possède les droits `sudo`.
- Les machines disposent de ressources suffisantes pour exécuter Kubernetes.
- Le réseau virtuel permet aux trois machines de communiquer.

Les commandes de ce guide sont exécutées depuis le terminal Ubuntu, sauf lorsqu'une manipulation VMware est explicitement indiquée.

---

## 4. Choisir le mode réseau dans VMware Workstation

VMware Workstation propose plusieurs modes de connexion réseau. Le choix dépend de l'architecture recherchée.

### Option 1 - NAT

Avec le mode NAT, les machines virtuelles utilisent le réseau de l'hôte pour accéder au réseau externe.

**Avantages :**
- Configuration généralement simple.
- Accès à Internet possible si la connexion de l'hôte fonctionne.
- Adapté à un laboratoire Kubernetes local.

**Inconvénients :**
- Les machines virtuelles ne sont pas directement exposées comme des machines physiques du réseau local.
- La configuration de l'accès depuis d'autres appareils peut nécessiter des règles supplémentaires.

**Utilisation recommandée :** laboratoire local lorsque l'on souhaite conserver une configuration réseau relativement simple.

### Option 2 - Bridged

Avec le mode Bridged, la carte réseau virtuelle est connectée au réseau physique de l'ordinateur hôte.

Les machines virtuelles peuvent obtenir des adresses IP sur le même réseau local que les autres appareils, sous réserve de la configuration du réseau physique.

**Avantages :**
- Communication possible avec les autres appareils du réseau local.
- Accès direct aux machines virtuelles depuis d'autres appareils lorsque le réseau et les pare-feu l'autorisent.
- Adapté à un laboratoire qui doit se comporter comme un ensemble de serveurs du réseau local.

**Inconvénients :**
- Les adresses IP doivent appartenir au sous-réseau approprié.
- Le réseau physique doit accepter les connexions des machines virtuelles.
- Certaines connexions Wi-Fi, politiques réseau ou configurations DHCP peuvent compliquer le fonctionnement.

**Utilisation recommandée :** laboratoire dont les serveurs doivent être accessibles sur le réseau local.

### Option 3 - Host-only

Le mode Host-only crée un réseau privé entre l'ordinateur hôte et les machines virtuelles connectées à ce réseau.

**Avantages :**
- Réseau isolé du réseau physique dans sa configuration habituelle.
- Pratique pour tester les communications entre les trois machines.
- Permet de travailler dans un environnement de laboratoire isolé.

**Inconvénients :**
- Pas d'accès direct à Internet dans la configuration habituelle.
- L'accès aux ressources externes nécessite une configuration supplémentaire.

**Utilisation recommandée :** laboratoire isolé destiné aux tests réseau et à l'apprentissage.

### Comparaison

| Critère | NAT | Bridged | Host-only |
|---|---|---|---|
| Communication entre les VM | Oui | Oui | Oui |
| Accès à Internet | Généralement oui | Selon le réseau physique | Non, généralement |
| Accès depuis le réseau local | Configuration supplémentaire possible | Oui, si le réseau l'autorise | Non, généralement |
| Isolation du réseau physique | Partielle | Faible | Forte |
| Adapté à Kubernetes | Oui | Oui | Oui, pour un laboratoire isolé |

### Choix retenu

Le mode réseau définitif sera choisi après vérification des besoins d'accès et de la configuration réseau existante.

Pour ce laboratoire, le mode Bridged peut être retenu si l'objectif est de rendre les trois serveurs accessibles sur le réseau local.

Le mode NAT reste une alternative adaptée si le cluster doit principalement fonctionner sur l'ordinateur hôte.

**Important :** choisir Bridged dans VMware ne configure pas automatiquement une adresse IP statique dans Ubuntu. Les deux opérations sont distinctes.

---

## 5. Configurer le réseau virtuel dans VMware Workstation

Cette étape s'effectue sur l'ordinateur hôte.

### Procédure

1. Ouvrir VMware Workstation.
2. Sélectionner la première machine virtuelle.
3. Cliquer sur `Edit virtual machine settings`.
4. Sélectionner `Network Adapter`.
5. Choisir le mode réseau souhaité : NAT, Bridged ou Host-only.
6. Vérifier que l'option `Connected` est activée.
7. Vérifier que `Connect at power on` est activée.
8. Répéter la procédure pour les deux autres machines.

Pour simplifier la configuration initiale, les trois machines doivent être connectées à un réseau qui permet leur communication.

En mode Bridged, il faut également vérifier que VMware utilise la bonne interface physique de l'hôte, notamment si l'ordinateur possède à la fois une carte Ethernet et une carte Wi-Fi.

---

## 6. Identifier la configuration réseau depuis le terminal Ubuntu

Cette étape est essentielle.

Avant de modifier les adresses IP, il faut identifier la configuration réellement utilisée par chaque machine.

Les commandes suivantes sont exécutées séparément sur le master et sur les deux workers.

### 6.1. Identifier le nom d'hôte

```bash
hostname
```

Cette commande affiche le nom actuel de la machine.

### 6.2. Identifier les interfaces réseau

```bash
ip -br link
```

Puis :

```bash
ip -br address
```

La première commande affiche les interfaces réseau et leur état.

La seconde affiche les interfaces et leurs adresses IP.

Exemple de résultat :

```text
lo       UNKNOWN   127.0.0.1/8
ens33    UP        192.168.189.128/24
```

Dans cet exemple :

- `lo` correspond à l'interface locale.
- `ens33` correspond à l'interface réseau de la machine virtuelle.
- `192.168.189.128/24` est une adresse IPv4 attribuée à l'interface.

**Attention :** `ens33` est un exemple. Le nom réel peut être différent sur chaque machine.

Il faut utiliser le nom réellement affiché par la commande.

### 6.3. Identifier la passerelle

```bash
ip route
```

Exemple :

```text
default via 192.168.189.2 dev ens33
192.168.189.0/24 dev ens33 proto kernel scope link src 192.168.189.128
```

Dans cet exemple :

- `192.168.189.2` est la passerelle par défaut.
- `ens33` est l'interface utilisée pour la route par défaut.
- `192.168.189.0/24` représente le sous-réseau directement connecté.

Ces valeurs ne sont que des exemples. La passerelle peut être différente en mode Bridged, NAT ou Host-only.

### 6.4. Identifier les serveurs DNS

```bash
resolvectl status
```

Cette commande permet d'examiner la configuration DNS utilisée par le système.

Il faut relever les serveurs DNS appropriés à l'environnement réseau.

### 6.5. Examiner les fichiers Netplan

```bash
ls -l /etc/netplan/
```

Puis :

```bash
sudo cat /etc/netplan/*.yaml
```

Ces commandes permettent d'identifier les fichiers de configuration réseau existants.

Il faut les examiner avant de créer ou de modifier un fichier, afin d'éviter plusieurs configurations contradictoires sur la même interface.

### Résumé des informations à relever

Pour chaque machine, noter :

| Information | Commande |
|---|---|
| Nom d'hôte | `hostname` |
| Interfaces réseau | `ip -br link` |
| Adresses IP | `ip -br address` |
| Passerelle | `ip route` |
| Serveurs DNS | `resolvectl status` |
| Fichiers Netplan | `ls -l /etc/netplan/` |

---

## 7. Déterminer les adresses IP statiques

Une adresse IP statique doit être choisie en fonction du sous-réseau réellement utilisé.

Il ne faut pas choisir arbitrairement les adresses du master et des workers.

### 7.1. Vérifier le sous-réseau

Supposons que la configuration actuelle affiche :

```text
192.168.189.128/24
```

et que la route par défaut soit :

```text
default via 192.168.189.2 dev ens33
```

Cela indique que l'interface utilise le sous-réseau `192.168.189.0/24` et que la passerelle indiquée est `192.168.189.2`.

Ce résultat doit être vérifié sur ta machine avant de retenir ce sous-réseau.

Si tu utilises le mode Bridged, tu peux obtenir un sous-réseau complètement différent.

### 7.2. Exemple de plan d'adressage

Le tableau suivant est un exemple de plan d'adressage si le réseau choisi est bien `192.168.189.0/24`.

| Machine | Adresse IP | Préfixe |
|---|---|---|
| Master | `192.168.189.130` | `/24` |
| Worker 1 | `192.168.189.131` | `/24` |
| Worker 2 | `192.168.189.132` | `/24` |

Ces adresses sont des exemples à valider.

Avant de les utiliser, vérifier :

- qu'elles appartiennent au bon sous-réseau ;
- qu'elles ne sont pas déjà utilisées ;
- qu'elles ne risquent pas d'être attribuées à un autre appareil par DHCP ;
- qu'elles respectent les règles du réseau choisi dans VMware.

Il est préférable de réserver ces adresses dans le DHCP lorsqu'on utilise un réseau administré par un routeur. Dans un laboratoire NAT VMware, il faut également tenir compte de la plage DHCP du réseau virtuel.

---

## 8. Configurer une adresse IP statique avec Netplan

Ubuntu Server 24.04 utilise Netplan pour gérer sa configuration réseau.

Cette section explique comment réaliser la configuration depuis le terminal, sans passer par une interface graphique de configuration réseau.

### 8.1. Sauvegarder les fichiers existants

Avant toute modification, créer une sauvegarde des fichiers Netplan.

```bash
sudo cp -a /etc/netplan "/root/netplan-backup-$(date +%F-%H%M%S)"
```

Cette commande sauvegarde le répertoire existant dans un nouveau répertoire situé sous `/root`.

### 8.2. Identifier le fichier à modifier

Afficher les fichiers présents :

```bash
ls -l /etc/netplan/
```

Examiner leur contenu :

```bash
sudo cat /etc/netplan/*.yaml
```

Sur une installation Ubuntu Server, le fichier peut notamment être nommé `50-cloud-init.yaml` ou porter un autre nom.

Ne pas supposer qu'un fichier précis existe.

Si la configuration actuelle est gérée par cloud-init, il faut également tenir compte de cette gestion afin d'éviter que la configuration soit régénérée ultérieurement.

Pour ce laboratoire, modifier la configuration existante de manière contrôlée est préférable à l'ajout aveugle d'un second fichier contenant des paramètres contradictoires.

### 8.3. Exemple de configuration du master

Supposons que les vérifications précédentes aient confirmé les valeurs suivantes :

- Interface : `ens33`
- Adresse du master : `192.168.189.130/24`
- Passerelle : `192.168.189.2`
- DNS : `192.168.189.2` et `1.1.1.1`

Le fichier Netplan correspondant pourrait contenir :

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.189.130/24
      routes:
        - to: default
          via: 192.168.189.2
      nameservers:
        addresses:
          - 192.168.189.2
          - 1.1.1.1
```

**Cette configuration est un exemple à adapter aux résultats réels de ton réseau.**

La passerelle et les serveurs DNS doivent être adaptés au réseau utilisé. En particulier, `192.168.189.2` ne doit pas être conservé si cette adresse n'est pas la passerelle réelle.

### 8.4. Modifier le fichier depuis le terminal

Supposons que le fichier existant soit `/etc/netplan/50-cloud-init.yaml`.

Ouvrir ce fichier :

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Remplacer le contenu approprié par la configuration adaptée à la machine.

Pour enregistrer dans Nano :

1. Appuyer sur `Ctrl + O`.
2. Appuyer sur `Entrée`.
3. Appuyer sur `Ctrl + X`.

Le nom du fichier est un exemple. Il faut utiliser celui qui existe réellement sur le serveur.

Ne pas modifier plusieurs fichiers simultanément sans comprendre comment leurs configurations seront fusionnées.

### 8.5. Vérifier la syntaxe

Avant d'appliquer la configuration :

```bash
sudo netplan generate
```

Cette commande permet de détecter notamment les erreurs de syntaxe.

Si une erreur apparaît, corriger le fichier avant de continuer.

### 8.6. Tester la configuration

Si tu travailles depuis la console VMware, ou si tu disposes d'un accès de secours à la machine :

```bash
sudo netplan try
```

Cette commande permet de tester la configuration et de la confirmer si elle fonctionne.

Elle est particulièrement utile lorsqu'une modification risque de couper une connexion SSH.

Si tu travailles à distance, conserve un accès à la console VMware afin de pouvoir récupérer la machine en cas de problème.

### 8.7. Appliquer et vérifier

Après validation :

```bash
sudo netplan apply
```

Puis :

```bash
ip -br address
ip route
```

Vérifier que l'adresse IP et la route par défaut correspondent à la configuration prévue.

Enfin, tester la passerelle :

```bash
ping -c 4 192.168.189.2
```

L'adresse de cette commande doit correspondre à la passerelle réellement configurée.

---

## 9. Configurer les deux workers

La procédure est identique pour les deux workers, mais chaque machine doit recevoir sa propre adresse IP.

### 9.1. Worker 1

Exemple de paramètres :

| Paramètre | Valeur d'exemple |
|---|---|
| Nom d'hôte | `k8s-worker1` |
| Interface | `ens33` |
| Adresse IP | `192.168.189.131/24` |
| Passerelle | `192.168.189.2` |

Exemple de configuration Netplan :

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.189.131/24
      routes:
        - to: default
          via: 192.168.189.2
      nameservers:
        addresses:
          - 192.168.189.2
          - 1.1.1.1
```

### 9.2. Worker 2

Exemple de paramètres :

| Paramètre | Valeur d'exemple |
|---|---|
| Nom d'hôte | `k8s-worker2` |
| Interface | `ens33` |
| Adresse IP | `192.168.189.132/24` |
| Passerelle | `192.168.189.2` |

Exemple de configuration Netplan :

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.189.132/24
      routes:
        - to: default
          via: 192.168.189.2
      nameservers:
        addresses:
          - 192.168.189.2
          - 1.1.1.1
```

**Important :** les exemples des sections 8 et 9 supposent le même sous-réseau et la même passerelle pour les trois machines. Ils doivent être adaptés si les résultats de l'inspection réseau montrent une configuration différente.

Sur chaque machine, effectuer les étapes suivantes :

1. Sauvegarder les fichiers existants.
2. Modifier la configuration Netplan.
3. Exécuter `sudo netplan generate`.
4. Tester avec `sudo netplan try`.
5. Appliquer la configuration validée.
6. Vérifier l'adresse IP et la passerelle.

---

## 10. Configurer les noms d'hôte

Une fois les adresses IP stables, attribuer un nom distinct à chaque machine.

### Sur le master

```bash
sudo hostnamectl set-hostname k8s-master
```

### Sur le worker 1

```bash
sudo hostnamectl set-hostname k8s-worker1
```

### Sur le worker 2

```bash
sudo hostnamectl set-hostname k8s-worker2
```

Vérifier le nom d'hôte :

```bash
hostname
```

Cette configuration facilite l'identification des machines lors de l'administration du cluster.

---

## 11. Configurer la résolution des noms entre les trois serveurs

Pour un laboratoire local, le fichier `/etc/hosts` peut permettre de résoudre les noms des machines sans mettre en place un serveur DNS spécifique.

Sur chacune des trois machines, ouvrir le fichier :

```bash
sudo nano /etc/hosts
```

Ajouter les trois lignes suivantes, en remplaçant les adresses par celles réellement retenues :

```text
192.168.189.130 k8s-master
192.168.189.131 k8s-worker1
192.168.189.132 k8s-worker2
```

Enregistrer le fichier.

Vérifier la résolution :

```bash
getent hosts k8s-master
getent hosts k8s-worker1
getent hosts k8s-worker2
```

Ces commandes doivent renvoyer les adresses associées aux noms d'hôte.

La résolution des noms ne garantit pas à elle seule que les machines sont joignables. Il faut également effectuer des tests réseau.

---

## 12. Vérifier la communication réseau

Cette étape doit être terminée avant l'installation de Kubernetes.

### 12.1. Vérifier les interfaces

Sur chaque machine :

```bash
ip -br address
```

### 12.2. Vérifier la route par défaut

```bash
ip route
```

### 12.3. Tester la communication depuis le master

Depuis `k8s-master` :

```bash
ping -c 4 k8s-worker1
ping -c 4 k8s-worker2
```

### 12.4. Tester la communication depuis le worker 1

Depuis `k8s-worker1` :

```bash
ping -c 4 k8s-master
ping -c 4 k8s-worker2
```

### 12.5. Tester la communication depuis le worker 2

Depuis `k8s-worker2` :

```bash
ping -c 4 k8s-master
ping -c 4 k8s-worker1
```

### Résultat attendu

Les machines doivent pouvoir joindre leurs adresses IP respectives.

Si les tests échouent, vérifier :

- les adresses IP et les masques réseau ;
- le mode réseau configuré dans VMware ;
- la passerelle et les routes ;
- la résolution des noms dans `/etc/hosts` ;
- les règles de pare-feu ;
- les éventuelles restrictions du réseau physique.

L'absence de réponse au ping ne signifie pas toujours que toute communication réseau est impossible : certaines règles de pare-feu bloquent les requêtes ICMP.

Il faut donc diagnostiquer l'origine du problème avant de poursuivre.

---

## 13. Préparer les serveurs pour Kubernetes

Une fois le réseau fonctionnel, préparer les trois machines pour Kubernetes.

Les opérations suivantes doivent être réalisées sur le master et les deux workers.

### 13.1. Mettre à jour les paquets

```bash
sudo apt update
sudo apt upgrade -y
```

### 13.2. Vérifier les ressources

```bash
nproc
free -h
df -h
```

Ces commandes permettent de vérifier le nombre de processeurs disponibles, la mémoire et l'espace disque.

### 13.3. Vérifier l'unicité des identifiants des machines

```bash
ip link
```

Puis :

```bash
sudo cat /sys/class/dmi/id/product_uuid
```

Les adresses MAC et les `product_uuid` doivent être uniques pour les trois nœuds.

### 13.4. Vérifier la connectivité

Avant de poursuivre, vérifier que les trois serveurs communiquent correctement et que les adresses IP ne changent pas de manière inattendue.

---

## 14. Installer Kubernetes avec kubeadm

Cette partie sera réalisée après validation de la configuration réseau.

Les principales étapes sont les suivantes :

1. Installer un runtime de conteneurs compatible, tel que containerd.
2. Configurer le runtime pour Kubernetes.
3. Préparer les paramètres système nécessaires.
4. Installer `kubeadm`, `kubelet` et `kubectl`.
5. Initialiser le Control Plane.
6. Installer un plugin réseau pour les pods.
7. Joindre les deux workers au cluster.
8. Vérifier l'état des nœuds.

Les versions des composants devront être choisies de manière cohérente au moment de l'installation.

### Initialiser le Control Plane

L'initialisation sera réalisée sur le master avec `kubeadm init`, en utilisant l'adresse IP retenue pour le Control Plane et un CIDR de pods adapté au plugin réseau choisi.

Exemple de structure de commande, à adapter :

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<IP_DU_MASTER> \
  --pod-network-cidr=<CIDR_DES_PODS>
```

Les valeurs entre chevrons sont des paramètres à remplacer, et non des valeurs à copier littéralement.

Le CIDR des pods doit être compatible avec le plugin réseau choisi et ne doit pas entrer en conflit avec le réseau des machines.

Après l'initialisation, configurer `kubectl` conformément aux instructions affichées par `kubeadm`.

### Joindre les workers

Après l'initialisation du Control Plane, Kubernetes fournit une commande `kubeadm join` destinée aux workers.

Cette commande doit être exécutée sur chacun des deux workers, avec les paramètres générés pour le cluster.

Ne pas partager publiquement le jeton de jointure ni les informations d'authentification associées.

### Vérifier le cluster

Depuis le master :

```bash
kubectl get nodes -o wide
```

L'objectif est d'obtenir trois nœuds enregistrés dans le cluster.

Une fois le réseau des pods correctement installé et les composants opérationnels, les nœuds devraient atteindre l'état `Ready`.

---

## 15. Dépannage

### Problème : l'adresse IP ne change pas

Vérifier :

```bash
ip -br address
sudo netplan get
sudo netplan generate
```

Examiner également tous les fichiers de `/etc/netplan/` afin de repérer une configuration concurrente.

### Problème : erreur lors de l'application de Netplan

Vérifier l'indentation du fichier YAML.

Netplan utilise une syntaxe sensible à l'indentation. Il faut employer des espaces et conserver une structure cohérente.

Exécuter :

```bash
sudo netplan generate
```

Corriger les erreurs avant d'appliquer la configuration.

### Problème : le master ne communique pas avec un worker

Vérifier :

```bash
ip -br address
ip route
getent hosts k8s-worker1
```

Depuis le master, tester également l'adresse IP réelle du worker :

```bash
ping -c 4 <IP_DU_WORKER>
```

Vérifier le mode réseau VMware, les masques réseau et les règles de pare-feu.

### Problème : Internet ne fonctionne plus après la configuration

Vérifier :

```bash
ip route
resolvectl status
```

La passerelle doit être celle du réseau utilisé. Les serveurs DNS doivent également être accessibles et adaptés à l'environnement.

### Problème : les nœuds Kubernetes restent NotReady

Vérifier :

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

Examiner ensuite le runtime de conteneurs, les journaux des composants concernés et la configuration du plugin réseau des pods.

---

## 16. Bonnes pratiques

- Documenter les adresses IP réellement attribuées.
- Conserver une sauvegarde des fichiers Netplan avant chaque modification.
- Vérifier la configuration avant de l'appliquer.
- Garder un accès à la console VMware pendant les modifications réseau.
- Utiliser des noms d'hôte uniques.
- Éviter les conflits d'adresses IP.
- Ne pas installer Kubernetes avant d'avoir validé la connectivité entre les nœuds.
- Ne pas publier les jetons de jointure ou les informations d'authentification du cluster.
- Documenter les commandes exécutées et leurs résultats.

---

## 17. Références officielles

### Ubuntu Server et Netplan

- [Documentation Ubuntu - Configuration réseau](https://ubuntu.com/server/docs/configuring-networks/)
- [Documentation Ubuntu - Netplan](https://ubuntu.com/server/docs/about-netplan/)

### Kubernetes

- [Documentation Kubernetes - Installation de kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Documentation Kubernetes - Création d'un cluster avec kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

### VMware

- [Documentation officielle VMware](https://docs.vmware.com/)

---

## 18. État d'avancement du projet

| Étape | Statut |
|---|---|
| Installation des trois VM Ubuntu Server 24.04 | Terminée |
| Choix du mode réseau VMware | À confirmer |
| Identification des interfaces réseau | À vérifier |
| Détermination du plan d'adressage IP | À réaliser |
| Configuration IP statique du master | À réaliser |
| Configuration IP statique du worker 1 | À réaliser |
| Configuration IP statique du worker 2 | À réaliser |
| Configuration des noms d'hôte | À réaliser |
| Vérification de la communication réseau | À réaliser |
| Préparation du runtime de conteneurs | À réaliser |
| Installation de Kubernetes | À réaliser |
| Initialisation du Control Plane | À réaliser |
| Jointure des workers | À réaliser |
| Vérification du cluster | À réaliser |

**État du projet :** préparation du réseau et documentation initiale.

La prochaine étape consiste à identifier les paramètres réseau réels des trois machines, puis à configurer leurs adresses IP statiques depuis le terminal Ubuntu.