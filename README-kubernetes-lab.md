# Kubernetes Lab — Ubuntu Server 24.04

Projet de laboratoire pour déployer un cluster Kubernetes (1 Control Plane, 2 Workers) sur trois machines virtuelles Ubuntu Server 24.04 LTS sous VMware Workstation avec le réseau NAT (`VMnet8`), puis intégrer Rancher pour l'administration du cluster.

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


---


## 1. Architecture cible et plan d'adressage (Validé)

| Rôle | Nom d'hôte (`hostname`) | Interface | Adresse IP Statique | Passerelle (Gateway) | DNS |
|---|---|---|---|---|---|
| **Control Plane** | `k8s-control-plane` | `ens33` | `192.168.189.10/24` | `192.168.189.2` | `1.1.1.1`, `8.8.8.8` |
| **Worker 01** | `k8s-worker-01` | `ens33` | `192.168.189.11/24` | `192.168.189.2` | `1.1.1.1`, `8.8.8.8` |
| **Worker 02** | `k8s-worker-02` | `ens33` | `192.168.189.12/24` | `192.168.189.2` | `1.1.1.1`, `8.8.8.8` |

### Spécifications de l'environnement
- **Hyperviseur :** VMware Workstation (Réseau NAT `VMnet8`)
- **Système d'exploitation :** Ubuntu Server 24.04 LTS (sans interface graphique)
- **Sous-réseau NAT :** `192.168.189.0/24` (Plage DHCP VMware : `192.168.189.128` à `192.168.189.254`)
- **Passerelle NAT :** `192.168.189.2`
- **Gestionnaire réseau :** Netplan (`/etc/netplan/50-cloud-init.yaml`)
- **Runtime de conteneurs :** `containerd` (avec `SystemdCgroup = true`)
- **Outils Kubernetes :** `kubeadm`, `kubelet`, `kubectl` (v1.30)

---

## 2. Résolution de noms locale (`/etc/hosts`)

Sur les 3 nœuds du cluster, le fichier `/etc/hosts` a été configuré comme suit :

- 192.168.189.10 k8s-control-plane
- 192.168.189.11 k8s-worker-01
- 192.168.189.12 k8s-worker-02

### 2.1 - Commandes de validation :

- getent hosts k8s-control-plane
- getent hosts k8s-worker-01
- getent hosts k8s-worker-02


### 2.2 - Commandes de validation validées :


---

### 2.3 - Mise à jour de `Configuration du réseau et déploiement d'un cluster Kubernetes sur VMware Workstation - Ubuntu Server 24.04.md`

Remplacez les sections **2**, **7**, **8**, **9**, **10** et **18** par le contenu réel :

```markdown
# Configuration du réseau et déploiement d'un cluster Kubernetes sur VMware Workstation — Ubuntu Server 24.04

## 2. Architecture du cluster (Réelle & Validée)

| Machine virtuelle | Nom d'hôte | Rôle | Adresse IP Statique | Passerelle |
|---|---|---|---|---|
| `Ubuntu-server-01` | `k8s-control-plane` | Control Plane | `192.168.189.10/24` | `192.168.189.2` |
| `Ubuntu-server-02` | `k8s-worker-01` | Worker 1 | `192.168.189.11/24` | `192.168.189.2` |
| `Ubuntu-server-03` | `k8s-worker-02` | Worker 2 | `192.168.189.12/24` | `192.168.189.2` |


---


## 3. Objectifs du projet

1. Créer trois VM Ubuntu Server sous VMware Workstation en mode NAT (VMnet8).

2. Configurer un nom d'hôte unique sur chaque machine (hostnamectl).

3. Configurer les adresses IP statiques via Netplan et la passerelle 192.168.189.2.

4. Configurer la résolution des noms locale (/etc/hosts).

5. Préparer les prérequis système Linux (désactivation du swap, modules noyau overlay et br_netfilter, paramètres sysctl).

6. Installer et configurer le runtime de conteneurs containerd avec SystemdCgroup = true.

7. Installer les outils Kubernetes (kubeadm, kubelet, kubectl).

8. Initialiser le Control Plane et joindre les Workers.

9. Installer un plugin réseau CNI et valider l'état du cluster (kubectl get nodes -o wide).

10. Installer et configurer Rancher pour l'administration.




## 4. Structure de la documentation


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




## 5. Parcours d'installation et procédures

### Étape 1 - Création et configuration des VM
* Création de trois machines virtuelles personnalisées (Custom) sous VMware Workstation.
* Utilisation de l'ISO Ubuntu Server 24.04 LTS.
* Sélection du mode réseau NAT (VMnet8).

#### 1.1 — Créer une machine virtuelle personnalisée (Custom)


- Typical : Permet de créer une machine virtuelle simplement en quelques étapes sans entrer dans les détails de configuration avancés.

- Custom : Permet une configuration personnalisée et précise (choix du matériel, des contrôleurs, etc.). C'est l'option recommandée pour ce laboratoire.



#### 1.2 - Configuration initiale de l'assistant

- Choisir "I will install the operating system later" pour garder la maîtrise de l'installation de l'ISO.

- Sélectionner Linux et choisir la version correspondante pour Ubuntu.

- Spécifier le nom de la machine virtuelle (k8s-control-plane, k8s-worker-01, k8s-worker-02).

- Attribuer les ressources processeur et la taille de la mémoire/disque dur.

- Finaliser la création de la structure de la VM.


#### 1.3 - Association de l'image ISO Ubuntu Server 24.04

- Effectuer un clic droit sur la machine virtuelle dans VMware, puis cliquer sur Settings.

- Sélectionner CD/DVD (SATA), cocher Use ISO image file et parcourir pour sélectionner votre fichier ISO Ubuntu Server 24.04.

- Valider en cliquant sur OK.


### Étape 2 - Configuration du réseau (Netplan)

#### 2.1 - Configuration appliquée sur k8s-control-plane (192.168.189.10) :


network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.189.10/24
      routes:
        - to: default
          via: 192.168.189.2
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8


#### 2.2 - Configuration appliquée sur k8s-worker-01 (192.168.189.11) :

network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.189.11/24
      routes:
        - to: default
          via: 192.168.189.2
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8


#### 2.3 - Configuration appliquée sur k8s-worker-02 (192.168.189.12) :

network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.189.12/24
      routes:
        - to: default
          via: 192.168.189.2
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8


> [!IMPORTRANT]
> Application et sécurisation des permissions :

sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply



### Étape 3 - Préparation système (Prérequis Kubernetes)

> [!NOTE]
> Réalisé sur les trois nœuds :

1. Désactivation du swap :

sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

2. Modules noyau (overlay et br_netfilter) :

sudo tee /etc/modules-load.d/k8s.conf <<EOF ## (Swap, (`.10`, (`VMnet8`) (`kubeadm (`kubeadm`, (`sysctl`) (les **En **Paramètres **Passerelle **Plage **Serveurs **Sous-réseau **Terminé** **À - --- --system / /etc/sysctl.d/k8s.conf 24.04 3. 6. 7. :** <<EOF Attribution CNI Configuration Control Création DHCP DNS EOF IP Initialisation Installation Kubernetes NAT Plan Plane Préparation Server Statut Ubuntu VM VMware Workers `.10`, `.11` `.11`, `.127` `.128-.254`). `.12` `.12`) `.3` `/etc/hosts` `1.1.1.1`, `192.168.189.0/24` `192.168.189.2` `8.8.8.8` ``` ```bash `containerd` `kubectl`) `kubelet`, adresses aux br_netfilter configuration cours d'adressage d'avancement d'hôtes de des du dynamique et fichier finaliser** hors init`) jointure la mode modprobe modules net.bridge.bridge-nf-call-ip6tables="1" net.bridge.bridge-nf-call-iptables="1" net.ipv4.ip_forward="1" noms noyau noyau, outils overlay parfaitement plage plugin projet réaliser** réseau réservée s'inscrivent statiques sudo sysctl sysctl) système tee trois | |---|---| À Étape État à>

<!-- 2. Modules noyau :

sudo tee /etc/modules-load.d/k8s.conf <<EOF ## ### 'deb 's/SystemdCgroup="false/SystemdCgroup" (Swap, (`.10`, (`/etc/hosts`) (`VMnet8`) (`hostnamectl`) (`kubeadm (`kubeadm`, (v1.30) **Paramètres **Terminé** - --- --dearmor --system -fsSL -i -m -o -p -y /' /etc/apt/keyrings /etc/apt/keyrings/kubernetes-apt-keyring.gpg /etc/apt/sources.list.d/kubernetes.list /etc/containerd /etc/containerd/config.toml /etc/sysctl.d/k8s.conf 24.04 3 3. 4 5 6. 7. 755 :** <<EOF CNI Configuration Control Création EOF IP Initialisation Installation Jointure Kubernetes Kubernetes]([https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)) Manager]([https://ranchermanager.docs.rancher.com/](https://ranchermanager.docs.rancher.com/)) Modules NAT Netplan]([https://netplan.readthedocs.io/](https://netplan.readthedocs.io/)) Plane Prérequis Rancher Références Server Server]([https://documentation.ubuntu.com/server/](https://documentation.ubuntu.com/server/)) Statut Sysctl) Ubuntu VMware Workers [Documentation [Installation [signed-by="/etc/apt/keyrings/kubernetes-apt-keyring.gpg]" `.11`, `.12`) ``` ```bash `containerd` `kubectl`) `kubelet`, adresses apt-get apt-mark apt-transport-https br_netfilter ca-certificates cluster config configuration containerd containerd]([https://containerd.io/docs/](https://containerd.io/docs/)) curl d'avancement d'hôte de default des du echo enable et gpg hold [https://pkgs.k8s.io/core:/stable:/v1.30/deb/](https://pkgs.k8s.io/core:/stable:/v1.30/deb/) [https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key) init`) install join`) kubeadm kubeadm]([https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)) kubectl kubelet la locale machines mkdir modprobe net.bridge.bridge-nf-call-ip6tables="1" net.bridge.bridge-nf-call-iptables="1" net.ipv4.ip_forward="1" noms noyau, officielles overlay paquets plugin projet restart runtime réaliser réseau résolution sed statiques sudo sysctl systemctl système tee true/g' update validation virtuelles | |---|---| À Étape État —> -->




## 7. Plan d'adressage et configuration VMware NAT (`VMnet8`)

- **Sous-réseau :** `192.168.189.0/24`
- **Passerelle NAT :** `192.168.189.2`
- **Serveurs DNS :** `1.1.1.1`, `8.8.8.8`
- **Plage réservée aux adresses statiques :** `.3` à `.127` (les adresses IP `.10`, `.11`, et `.12` s'inscrivent parfaitement hors de la plage DHCP `.128-.254`).

---

## 8. Fichier Netplan type (`/etc/netplan/50-cloud-init.yaml`)

Exemple appliqué sur `k8s-control-plane` (à adapter avec `.11` pour `k8s-worker-01` et `.12` pour `k8s-worker-02`) :

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.189.10/24
      routes:
        - to: default
          via: 192.168.189.2
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8

          