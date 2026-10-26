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


<!-- Images : [`docs/images/Cluster-Manager-01.png`](docs/images/Cluster-Manager-01.png) -->


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
- Accès administrateur `sudo` sur chaque VM.
- Réseau virtuel permettant la communication entre les nœuds.
- Connectivité Internet pour télécharger les paquets et les images.
- Ressources CPU, mémoire et stockage adaptées au laboratoire.
- VMware, un terminal Windows et un client SSH sur le poste d'administration.
- Vérifier l'état du swap sur les trois VM et appliquer la configuration compatible avec le mode de fonctionnement Kubernetes retenu.
- Vérifier la prise en charge des modules noyau et des paramètres réseau nécessaires au runtime et au plugin CNI.
- Identifier l'état du pare-feu UFW et prévoir les règles nécessaires aux communications entre les nœuds.
- Vérifier la compatibilité des versions de Kubernetes, de containerd, du plugin CNI et de Rancher.
- Prévoir Helm si Rancher est installé sur Kubernetes à l'aide de son chart Helm.

> [!IMPORTANT]
> Kubernetes et Rancher peuvent demander davantage de ressources selon les composants installés. Une configuration minimale ne garantit pas de bonnes performances.

## 4. Structure de la documentation

```text
kubernetes-lab/
├── README.md
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
```

Créer les fichiers référencés avant de considérer la documentation comme complète.

## 5. Parcours d'installation

### Étape 1 — Préparer les VM

Documenter la version d'Ubuntu, le nombre de CPU, la mémoire, le disque, le mode réseau VMware, les noms d'hôtes et la méthode d'accès SSH.

Vérifier également l'état du swap et préparer les paramètres système nécessaires à Kubernetes. Pour le déploiement standard retenu dans ce laboratoire, le swap doit être désactivé, sauf si une configuration compatible avec son utilisation est explicitement mise en place.

Documenter les modifications apportées à la configuration persistante afin qu'elles restent cohérentes après un redémarrage.

Document : [`docs/01-vm-setup.md`](docs/01-vm-setup.md)

### Étape 2 — Configurer le réseau et les noms

Vérifier le réseau VMware avant d'attribuer les adresses. Ubuntu Server utilise généralement Netplan pour la configuration réseau. Identifier la passerelle et les DNS réels, sans reprendre des valeurs supposées.

**Commandes de diagnostic :**

```bash
ip -br address
ip route
resolvectl status
hostnamectl
```

**Vérifier également l'état du pare-feu UFW sur chaque machine :**

```bash
sudo ufw status verbose
```

Si le pare-feu est actif, identifier les règles nécessaires à l'API Kubernetes, aux communications entre les nœuds et au plugin CNI choisi. Ne pas désactiver systématiquement le pare-feu : adapter les règles aux composants effectivement installés et au réseau du laboratoire.



Document : [`docs/02-network-dns.md`](docs/02-network-dns.md)

### Étape 3 — Installer le runtime de conteneurs


Pour ce laboratoire, containerd est le runtime de conteneurs retenu pour Kubernetes. Il doit être configuré avec une interface CRI compatible avec la version de Kubernetes installée.

Avant l'installation, préparer les modules noyau nécessaires, notamment overlay et br_netfilter selon la configuration retenue, puis vérifier les paramètres réseau du noyau, dont net.ipv4.ip_forward et les paramètres de traitement du trafic des ponts réseau.

Vérifier également la configuration de systemd cgroups, le fonctionnement du service containerd et la disponibilité de son interface CRI.

Précision sur Docker : depuis Kubernetes 1.24, Docker Engine n'est plus pris en charge directement comme runtime natif par Kubernetes. Il n'est pas nécessaire d'installer Docker Engine sur les nœuds pour utiliser containerd. Docker peut être utilisé séparément ou avec un adaptateur CRI approprié.


<!-- Pour un déploiement `kubeadm`, `containerd` est un choix courant de runtime compatible CRI. Suivre la documentation officielle pour préparer le système, configurer le runtime et installer les outils Kubernetes. Vérifier la compatibilité des versions avant d'exécuter les commandes. -->

Document : [`docs/03-container-runtime.md`](docs/03-container-runtime.md)

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

Document : [`docs/04-kubernetes.md`](docs/04-kubernetes.md)

### Étape 5 — Installer Rancher


Choisir une méthode compatible avec les versions de Kubernetes et de Rancher retenues. Pour ce laboratoire, étudier l'installation de Rancher sur Kubernetes avec Helm.

Avant le déploiement, vérifier les prérequis officiels, installer Helm sur le poste d'administration ou sur une machine disposant de l'accès nécessaire au cluster, puis préparer les composants requis par la méthode choisie, notamment l'accès HTTPS et le certificat adapté.

Un conteneur Docker Rancher unique peut servir à des tests, mais cette méthode ne doit pas être présentée comme une architecture de production.


<!-- Choisir une méthode compatible avec la version de Kubernetes et les prérequis de Rancher. Pour un déploiement destiné à durer, étudier l'installation sur Kubernetes avec Helm. Un conteneur Docker Rancher unique peut servir à des tests, mais ne doit pas être présenté comme une architecture de production. -->

Document : [`docs/05-rancher.md`](docs/05-rancher.md)

### Étape 6 — Valider le cluster

Depuis le Control Plane, exécuter :

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

> [!TIP]
> Vérifie que les trois nœuds sont `Ready`, que les pods système fonctionnent et qu'un déploiement de test démarre correctement avant de déclarer le cluster opérationnel.

Document : [`docs/06-validation.md`](docs/06-validation.md)

### Étape 7 — Documenter les problèmes

Pour chaque incident, noter le message d'erreur exact, la machine concernée, les commandes de diagnostic, la cause confirmée si elle est connue, la correction et le résultat après correction.

Document : [`docs/07-troubleshooting.md`](docs/07-troubleshooting.md)

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
