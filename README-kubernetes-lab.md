# Kubernetes Lab — Ubuntu Server 24.04

Projet de laboratoire pour déployer un cluster Kubernetes sur trois machines virtuelles Ubuntu Server 24.04 sous VMware, puis intégrer Rancher pour l'administration du cluster.


> [!NOTE]
> **État du projet :** dépôt Git créé. La documentation et les procédures seront complétées et validées au fur et à mesure de l'installation.


---

## Illustrations du projet

### Google Cloud Platform
![Google Cloud Platform](docs/images/Google-Cloud-Platform-01.png)

### Kubernetes
![Kubernetes](docs/images/Kubernetes-01.png)

### Cluster Kubernetes
![Architecture du cluster Kubernetes](docs/images/Kubernetes-Cluster-01.png)
![Architecture du cluster Kubernetes](docs/images/Cluster-Manager-01.png)

### Rancher
![Interface Rancher](docs/images/Rancher-01.png)

### Workflow Rancher
![Workflow Rancher](docs/images/Rancher-workflow-01.png)


<!-- [`docs/images/Cluster-Manager-01.png`](docs/images/Cluster-Manager-01.png) -->


## 1. Architecture cible

### Architecture du cluster

| Rôle | Nom d'hôte | Adresse IP prévue |
|---|---|---|
| Control Plane | `k8s-server-01` | `192.168.189.10` |
| Worker | `k8s-server-02` | `192.168.189.11` |
| Worker | `k8s-server-03` | `192.168.189.12` |

- **Système :** Ubuntu Server 24.04 LTS
- **Virtualisation :** VMware
- **Réseau prévu :** `192.168.189.0/24`, à confirmer dans la configuration VMware
- **Initialisation :** `kubeadm`
- **Runtime envisagé :** `containerd`
- **Gestion du cluster :** Rancher

> [!WARNING]
> Ces adresses IP sont prévisionnelles. Avant de les configurer, vérifie le sous-réseau, la passerelle, la plage DHCP et l'absence de conflits IP dans VMware.

## 2. Objectifs

1. Créer trois VM Ubuntu Server.
2. Configurer un nom d'hôte unique sur chaque VM.
3. Configurer les adresses IP statiques et la résolution des noms.
4. Vérifier la connectivité entre les nœuds.
5. Préparer Ubuntu et installer un runtime compatible avec Kubernetes.
6. Installer Kubernetes avec `kubeadm`.
7. Installer un plugin réseau CNI et joindre les deux Workers.
8. Vérifier le cluster avec `kubectl`.
9. Installer Rancher et vérifier son accès.
10. Documenter les commandes, les résultats, les erreurs et les corrections.

## 3. Prérequis

- Trois VM Ubuntu Server 24.04 LTS.
- Accès administrateur sudo sur chaque VM.
- Réseau virtuel permettant la communication entre les nœuds.
- Connectivité Internet pour télécharger les paquets et les images.
- Ressources CPU, mémoire et stockage adaptées au laboratoire.
- VMware Workstation, un terminal Windows et un client SSH sur le poste d'administration.
- Vérifier l'état du swap sur les trois VM et appliquer la configuration compatible avec le mode de fonctionnement Kubernetes retenu.
- Vérifier les modules noyau et les paramètres réseau nécessaires au runtime et au plugin CNI.
- Identifier l'état du pare-feu UFW et prévoir les règles nécessaires aux communications entre les nœuds.
- Vérifier la compatibilité des versions de Kubernetes, de containerd, du plugin CNI et de Rancher.
- Prévoir Helm si Rancher est installé sur Kubernetes à l'aide de son chart Helm.

<!-- - Trois VM Ubuntu Server 24.04 LTS.
- Accès administrateur `sudo` sur chaque VM.
- Réseau virtuel permettant la communication entre les nœuds.
- Connectivité Internet pour télécharger les paquets et les images.
- Ressources CPU, mémoire et stockage adaptées au laboratoire.
- VMware, un terminal Windows et un client SSH sur le poste d'administration.
- Vérifier l'état du swap sur les trois VM et appliquer la configuration compatible avec le mode de fonctionnement Kubernetes retenu.
- Vérifier la prise en charge des modules noyau et des paramètres réseau nécessaires au runtime et au plugin CNI.
- Identifier l'état du pare-feu UFW et prévoir les règles nécessaires aux communications entre les nœuds.
- Vérifier la compatibilité des versions de Kubernetes, de containerd, du plugin CNI et de Rancher.
- Prévoir Helm si Rancher est installé sur Kubernetes à l'aide de son chart Helm. -->

> [!IMPORTANT]
> Kubernetes et Rancher peuvent demander davantage de ressources selon les composants installés. Une configuration minimale ne garantit pas de bonnes performances.

## 4. Structure de la documentation

```text
kubernetes-lab/
├── README-kubernetes-lab.md
├── LICENSE
├── .gitignore
└── docs/
    ├── 01-vm-setup.md
    ├── 02-network-dns.md
    ├── 03-container-runtime.md
    ├── 04-kubernetes.md
    ├── 05-rancher.md
    ├── 06-validation.md
    ├── 07-troubleshooting.md
    └── images/
        ├── Google-Cloud-Platform-01.png
        ├── Kubernetes-01.png
        ├── Kubernetes-Cluster-01.png
        ├── Cluster-Manager-01.png
        ├── Rancher-01.png
        ├── Rancher-workflow-01.png
        └── network-dns/
            ├── vm-settings/
            └── virtual-network-editor/
<!-- 
kubernetes-lab/
├── README-kubernetes-lab.md
├── LICENSE
├── .gitignore
└── docs/
    ├── 01-vm-setup.md
    ├── 02-network-dns.md
    ├── 03-container-runtime.md
    ├── 04-kubernetes.md
    ├── 05-rancher.md
    ├── 06-validation.md
    ├── 07-troubleshooting.md
    └── images/ -->
```

Créer les fichiers référencés avant de considérer la documentation comme complète.

## 5. Parcours d'installation

### Étape 1 - Préparer les VM

![Preparer-les-vm](docs/images/network-dns/preparer-les-VM/confiduration-des-vm.png)


#### 1.1 - Créer une machine virtuelle personnalisée avec VMware

![create-a-new-virtual-machine](docs/images/network-dns/preparer-les-VM/confiduration-des-vm.png)

#### 1.1.1 Choisir le type de machines virtuelle

---

Nous pouvons choisir entre deux types de machines virtuelles : 

- « Typical » : Permet de créer une machine virtuelle simplement en quelques étapes mais sans entrer dans la configuration (choix du 
système d’exploitation, nom de la machine virtuelle, lieu de stockage de la machine virtuelle et taille du disque dur virtuelle). 
- « Custom » : Permet de créer une machine virtuelle avec une configuration personnalisée et précise. 

Dans cet article, nous allons choisir le type de machines virtuelle « Custom » pour pouvoir entrer dans les détails de configuration et de 
création de notre VM. 


![create-a-new-virtual-machine](docs/images/creation-de-vm/create-a-new-virtual-machine-02.png)


choisissez I will install the operating system later pour plus de manoeuvre.

![create-a-new-virtual-machine](docs/images/creation-de-vm/create-a-new-virtual-machine-03.png)


choisissez Linux si votre server ou Machine virtuelle a un iso Linux

![create-a-new-virtual-machine](docs/images/creation-de-vm/create-a-new-virtual-machine-04.png)


configuirer la virtual machine name :

![create-a-new-virtual-machine](docs/images/creation-de-vm/create-a-new-virtual-machine-05.png)


Choisissez la taille de la mémoire du disque dure

![create-a-new-virtual-machine](docs/images/creation-de-vm/create-a-new-virtual-machine-06.png)


finisser et vous aller choisir l'iso

![create-a-new-virtual-machine](docs/images/creation-de-vm/create-a-new-virtual-machine-07.png)


#### 1.1.2 choix de l'iso

clique droit sur la Virtual Machine (VM) cliquer ensuite sur setting

![choix-de-l-iso](docs/images/choix-de-l-iso/iso.png)

clique sur CD/DVD (SATA), 


dans la partie de l'image a droite, cliquer sur Use ISO image file, Browser :

le Browser cher moi est : D:\Download\software\linux\Ubuntu\Server\ISO 2 il pourais être diférrent chez vous, et le fichier est : 


![choix-de-l-iso](docs/images/choix-de-l-iso/iso-01.png)

finalisez en cliquant sur OK

![choix-de-l-iso](docs/images/choix-de-l-iso/iso-03.png)


> [!NOTE]
> Documenter la version d'Ubuntu, le nombre de CPU, la mémoire, le disque, le mode réseau VMware, les noms d'hôtes et la méthode d'accès SSH.

> [!NOTE]
> Vérifier également l'état du swap et préparer les paramètres système nécessaires à Kubernetes. Pour le déploiement standard retenu dans ce laboratoire, le swap doit être désactivé, sauf si une configuration compatible avec son utilisation est explicitement mise en place.

> [!NOTE]
> Documenter les modifications apportées à la configuration persistante afin qu'elles restent cohérentes après un redémarrage.



<!-- [`docs/01-vm-setup.md`](docs/01-vm-setup.md) -->
[`docs/01-vm-setup.md`](docs/01-vm-setup.md)

---

### Étape 2 - Configuration du réseau et déploiement d'un cluster Kubernetes sur VMware Workstation - Ubuntu Server 24.04



[`docs/configuration-du-reseau-et-deploiement-d-un-cluster-Kubernetes/VMware-Workstation/Ubuntu-Server-24.04/configuration-du-reseau-et-deploiement-d-un-cluster-Kubernetes-sur-VMware-Workstation-Ubuntu-Server-24.04.md`](docs/configuration-du-reseau-et-deploiement-d-un-cluster-Kubernetes/VMware-Workstation/Ubuntu-Server-24.04/configuration-du-reseau-et-deploiement-d-un-cluster-Kubernetes-sur-VMware-Workstation-Ubuntu-Server-24.04.md)

---

### Étape 2 - Configurer le réseau et les noms

Cette étape consiste à préparer la communication entre les trois machines Ubuntu Server 24.04, à configurer leurs adresses IP statiques et à vérifier la résolution DNS avant l'installation de Kubernetes.

**Attention :** les adresses proposées ci-dessous doivent être confirmées à partir de la configuration VMware. Ne configure pas une passerelle ou une adresse IP avant d'avoir vérifié le sous-réseau, la passerelle réelle, la plage DHCP et les éventuels conflits d'adresses.


#### 2.1 — Vérifier la configuration VMware

Avant de modifier Netplan, vérifier le mode réseau sélectionné dans les paramètres de chaque machine virtuelle et consulter le Virtual Network Editor.

Les captures de référence doivent être conservées dans le dossier docs/images/network-dns/.

- Captures des paramètres réseau de la VM
- Captures du Virtual Network Editor

Les captures doivent permettre d'identifier le mode réseau (NAT, Bridged ou Host-only), le sous-réseau, la passerelle NAT et, si nécessaire, la plage DHCP.

Consulter la procédure détaillée : [`docs/02-network-dns.md`](docs/02-network-dns.md)

#### 2.2 — Diagnostiquer le réseau Ubuntu actuel

Sur chaque machine, exécuter :

```bash
ip -br address
ip route
resolvectl status
hostnamectl
ls -la /etc/netplan/
sudo netplan get
```

Vérifier également le pare-feu UFW sur chaque machine :

```bash
sudo ufw status verbose
```

Si le pare-feu est actif, identifier les règles nécessaires à l'API Kubernetes, aux communications entre les nœuds et au plugin CNI choisi. Ne pas désactiver systématiquement le pare-feu.


#### 2.3 — Plan d'adressage prévu

| Rôle | Nom d'hôte | Interface | Adresse IP prévue |
|---|---|---|---|
| Control Plane | `k8s-server-01` | `ens33` | `192.168.189.10/24` |
| Worker 01 | `k8s-server-02` | `ens33` | `192.168.189.11/24` |
| Worker 02 | `k8s-server-03` | `ens33` | `192.168.189.12/24` |

La passerelle `192.168.189.2` et les DNS `1.1.1.1` et `8.8.8.8` sont des valeurs envisagées, à confirmer avant utilisation.

#### 2.4 — Préparer Netplan

Ubuntu Server 24.04 utilise généralement Netplan pour configurer le réseau.

Lister les fichiers existants :

```bash
ls -la /etc/netplan/
```

Examiner ensuite leur contenu avant toute modification :

```bash
sudo netplan get
```

Sauvegarder le fichier de configuration concerné avant de le modifier. Ne pas supprimer les fichiers Netplan à l'aveugle : vérifier d'abord leur rôle et déterminer quelle configuration doit être conservée.

#### 2.5 — Configurer les adresses IP statiques

Lorsque le sous-réseau et la passerelle auront été confirmés, configurer l'adresse IP propre à chaque machine dans son fichier Netplan.

- `k8s-server-01` : `192.168.189.10/24`
- `k8s-server-02` : `192.168.189.11/24`
- `k8s-server-03` : `192.168.189.12/24`

La configuration doit préciser l'interface réseau, désactiver DHCP pour IPv4 si l'on choisit une adresse statique, définir la route par défaut et déclarer les serveurs DNS.

Ne pas appliquer les exemples de configuration avant d'avoir confirmé les paramètres réseau réels.

#### 2.6 — Appliquer et vérifier

Après avoir sauvegardé et modifié le fichier Netplan sur chaque machine, vérifier la syntaxe :

```bash
sudo netplan generate
```

Si la configuration est valide et que tu disposes d'un moyen de récupérer la console en cas de perte du réseau, appliquer les changements :

```bash
sudo netplan apply
```

Vérifier ensuite l'adresse IP :

```bash
ip -br address
```

Vérifier la route par défaut :

```bash
ip route
```

Tester la communication entre les nœuds depuis le Control Plane :

```bash
ping -c 3 192.168.189.11
ping -c 3 192.168.189.12
```

Vérifier la résolution DNS :

```bash
getent hosts archive.ubuntu.com
```

Un échec de `ping` ne prouve pas à lui seul que le réseau est défaillant : certains équipements bloquent les requêtes ICMP. Il faut aussi vérifier les adresses IP, les routes et la résolution DNS.

Document détaillé : [`docs/02-network-dns.md`](docs/02-network-dns.md)


<!-- ### Étape 2 — Configurer le réseau et les noms

Vérifier le réseau VMware avant d'attribuer les adresses. Ubuntu Server utilise généralement Netplan pour la configuration réseau. Identifier la passerelle et les DNS réels, sans reprendre des valeurs supposées.

**Commandes de diagnostic :**

```bash
ip -br address
ip -br addr
ip route
resolvectl status
hostnamectl
```

**Vérifier également l'état du pare-feu UFW sur chaque machine :**

```bash
sudo ufw status verbose
```

Si le pare-feu est actif, identifier les règles nécessaires à l'API Kubernetes, aux communications entre les nœuds et au plugin CNI choisi. Ne pas désactiver systématiquement le pare-feu : adapter les règles aux composants effectivement installés et au réseau du laboratoire. -->



<!-- [`docs/02-network-dns.md`](docs/02-network-dns.md) -->



#### 2.7 — Examiner la configuration réseau VMware

Avant de configurer les adresses IP statiques, vérifier le mode réseau des trois machines virtuelles et les paramètres du réseau virtuel VMware.

Les captures d’écran de référence sont disponibles dans le dossier [`docs/images/network-dns/`](docs/images/network-dns/).

- [Paramètres réseau de la machine virtuelle](docs/images/network-dns/Paramètres%20réseau%20de%20la%20VM/)
- [Virtual Network Editor](docs/images/network-dns/Virtual%20Network%20Editor/)

Ces captures permettent de documenter le mode réseau choisi (NAT, Bridged ou Host-only), le sous-réseau, la passerelle NAT et, si nécessaire, la plage DHCP.

**Attention :** les adresses IP et la passerelle présentées dans le plan d’adressage restent prévisionnelles tant que la configuration VMware n’a pas été vérifiée.

Consulter la procédure détaillée : [`docs/02-network-dns.md`](docs/02-network-dns.md).


<!-- ### Étape 3 — Installer le runtime de conteneurs

Pour ce laboratoire, containerd est le runtime de conteneurs retenu pour Kubernetes. Il doit être configuré avec une interface CRI compatible avec la version de Kubernetes installée.

Avant l'installation, préparer les modules noyau nécessaires, notamment overlay et br_netfilter selon la configuration retenue, puis vérifier les paramètres réseau du noyau, dont net.ipv4.ip_forward et les paramètres de traitement du trafic des ponts réseau.

Vérifier également la configuration de systemd cgroups, le fonctionnement du service containerd et la disponibilité de son interface CRI.

Précision sur Docker : depuis Kubernetes 1.24, Docker Engine n'est plus pris en charge directement comme runtime natif par Kubernetes. Il n'est pas nécessaire d'installer Docker Engine sur les nœuds pour utiliser containerd. Docker peut être utilisé séparément ou avec un adaptateur CRI approprié.

docs/03-container-runtime.md -->
Document container-runtime : [`docs/03-container-runtime.md`](docs/03-container-runtime.md)




### Étape 3 — Installer le runtime de conteneurs


Pour ce laboratoire, containerd est le runtime de conteneurs retenu pour Kubernetes. Il doit être configuré avec une interface CRI compatible avec la version de Kubernetes installée.

Avant l'installation, préparer les modules noyau nécessaires, notamment overlay et br_netfilter selon la configuration retenue, puis vérifier les paramètres réseau du noyau, dont net.ipv4.ip_forward et les paramètres de traitement du trafic des ponts réseau.

Vérifier également la configuration de systemd cgroups, le fonctionnement du service containerd et la disponibilité de son interface CRI.

Précision sur Docker : depuis Kubernetes 1.24, Docker Engine n'est plus pris en charge directement comme runtime natif par Kubernetes. Il n'est pas nécessaire d'installer Docker Engine sur les nœuds pour utiliser containerd. Docker peut être utilisé séparément ou avec un adaptateur CRI approprié.


<!-- Pour un déploiement `kubeadm`, `containerd` est un choix courant de runtime compatible CRI. Suivre la documentation officielle pour préparer le système, configurer le runtime et installer les outils Kubernetes. Vérifier la compatibilité des versions avant d'exécuter les commandes. -->

[`docs/03-container-runtime.md`](docs/03-container-runtime.md)

### Étape 4 — Installer Kubernetes

Installer les composants requis sur les trois machines, puis initialiser le Control Plane sur `k8s-server-01`. Installer le plugin réseau CNI retenu et joindre les Workers à l'aide de la commande `kubeadm join` générée par le Control Plane.

Avant l'initialisation du Control Plane, vérifier sur les trois VM :

- que le runtime fonctionne et que son interface CRI est disponible ;
- que les modules noyau et les paramètres réseau nécessaires sont actifs ;
- que la configuration du swap correspond au mode de fonctionnement choisi ;
- que les versions de kubeadm, kubelet et kubectl sont compatibles ;
- que le réseau permet les communications nécessaires entre les composants.

Après l'installation du plugin CNI, vérifier la disponibilité des nœuds et l'état des pods système avant de poursuivre.


> [!CAUTION]
> Ne publie jamais dans le dépôt une commande `kubeadm join` contenant son jeton, une clé privée SSH, un mot de passe ou un kubeconfig sensible.

[`docs/04-kubernetes.md`](docs/04-kubernetes.md)

### Étape 5 — Installer Rancher


Choisir une méthode compatible avec les versions de Kubernetes et de Rancher retenues. Pour ce laboratoire, étudier l'installation de Rancher sur Kubernetes avec Helm.

Avant le déploiement, vérifier les prérequis officiels, installer Helm sur le poste d'administration ou sur une machine disposant de l'accès nécessaire au cluster, puis préparer les composants requis par la méthode choisie, notamment l'accès HTTPS et le certificat adapté.

Un conteneur Docker Rancher unique peut servir à des tests, mais cette méthode ne doit pas être présentée comme une architecture de production.


<!-- Choisir une méthode compatible avec la version de Kubernetes et les prérequis de Rancher. Pour un déploiement destiné à durer, étudier l'installation sur Kubernetes avec Helm. Un conteneur Docker Rancher unique peut servir à des tests, mais ne doit pas être présenté comme une architecture de production. -->

[`docs/05-rancher.md`](docs/05-rancher.md)

### Étape 6 — Valider le cluster

Depuis le Control Plane, exécuter :

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

> [!TIP]
> Vérifie que les trois nœuds sont `Ready`, que les pods système fonctionnent et qu'un déploiement de test démarre correctement avant de déclarer le cluster opérationnel.

[`docs/06-validation.md`](docs/06-validation.md)

### Étape 7 — Documenter les problèmes

Pour chaque incident, noter le message d'erreur exact, la machine concernée, les commandes de diagnostic, la cause confirmée si elle est connue, la correction et le résultat après correction.

[`docs/07-troubleshooting.md`](docs/07-troubleshooting.md)

## 6. Critères de réussite

- [ ] Les trois VM démarrent correctement.
- [ ] Les noms d'hôtes sont uniques et corrects.
- [ ] Les adresses IP sont configurées sans conflit.
- [ ] Les nœuds communiquent entre eux.
- [ ] Le runtime fonctionne sur chaque nœud.
- [ ] Le Control Plane est initialisé.
- [ ] Les deux Workers ont rejoint le cluster.
- [ ] `kubectl get nodes -o wide` affiche trois nœuds `Ready`.
- [ ] Les pods système sont dans un état attendu.
- [ ] Un déploiement de test fonctionne.
- [ ] Rancher est accessible et permet de consulter le cluster.
- [ ] Les versions, commandes et résultats sont documentés.

> [!IMPORTANT]
> Coche chaque case uniquement après avoir effectué et vérifié le test correspondant.

## 7. Sécurité et bonnes pratiques

> [!CAUTION]
> Ne versionne jamais une clé privée SSH, un mot de passe, un jeton ou un kubeconfig sensible. Vérifie toujours les fichiers à publier avant un commit.

- Vérifier `git status` avant chaque commit.
- Ne pas publier de captures contenant des secrets ou des informations sensibles.
- Noter les versions utilisées et vérifier leur compatibilité.
- Sauvegarder les fichiers de configuration avant de les modifier.
- Indiquer sur quelle machine chaque commande doit être exécutée.
- Distinguer les commandes exécutées des commandes proposées.
- Ne pas déclarer une étape réussie sans vérification.

## 8. Références officielles

- [Documentation Kubernetes](https://kubernetes.io/docs/home/)
- [Installation de kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Documentation Ubuntu Server](https://documentation.ubuntu.com/server/)
- [Documentation Netplan](https://netplan.readthedocs.io/)
- [Documentation Rancher](https://ranchermanager.docs.rancher.com/)
- [Installer Rancher sur Kubernetes avec Helm](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster)
- [Installer Docker Engine sur Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Documentation officielle de containerd](https://containerd.io/docs/?utm_source=chatgpt.com)
- [Documentation officielle de Helm](https://helm.sh/docs/?utm_source=chatgpt.com)

## 9. État actuel

> [!NOTE]
> Le dépôt a été cloné sur le poste Windows. La clé SSH a été créée et ajoutée au compte GitHub, et le test `ssh -T git@github.com` a réussi.

> [!WARNING]
> La documentation des étapes et les captures doivent encore être complétées. L'installation de Kubernetes et de Rancher reste à effectuer et à valider.

**Dernière mise à jour :** 2026-10-26.
