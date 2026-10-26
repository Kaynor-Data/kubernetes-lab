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

Document : [`docs/01-vm-setup.md`](docs/01-vm-setup.md)

### Étape 2 — Configurer le réseau et les noms

Vérifier le réseau VMware avant d'attribuer les adresses. Ubuntu Server utilise généralement Netplan pour la configuration réseau. Identifier la passerelle et les DNS réels, sans reprendre des valeurs supposées.

Commandes de diagnostic :

```bash
ip -br address
ip route
resolvectl status
hostnamectl
```

Tester ensuite la connectivité entre les nœuds et vers la passerelle. Consigner les résultats observés.

Document : [`docs/02-network-dns.md`](docs/02-network-dns.md)

### Étape 3 — Installer le runtime de conteneurs

Pour un déploiement `kubeadm`, `containerd` est un choix courant de runtime compatible CRI. Suivre la documentation officielle pour préparer le système, configurer le runtime et installer les outils Kubernetes. Vérifier la compatibilité des versions avant d'exécuter les commandes.

Document : [`docs/03-container-runtime.md`](docs/03-container-runtime.md)

### Étape 4 — Installer Kubernetes

Installer les composants requis sur les trois machines, puis initialiser le Control Plane sur `k8s-server-01`. Installer le plugin réseau CNI retenu et joindre les Workers à l'aide de la commande `kubeadm join` générée par le Control Plane.

> [!CAUTION]
> Ne publie jamais dans le dépôt une commande `kubeadm join` contenant son jeton, une clé privée SSH, un mot de passe ou un kubeconfig sensible.

Document : [`docs/04-kubernetes.md`](docs/04-kubernetes.md)

### Étape 5 — Installer Rancher

Choisir une méthode compatible avec la version de Kubernetes et les prérequis de Rancher. Pour un déploiement destiné à durer, étudier l'installation sur Kubernetes avec Helm. Un conteneur Docker Rancher unique peut servir à des tests, mais ne doit pas être présenté comme une architecture de production.

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

## 9. État actuel

> [!NOTE]
> Le dépôt a été cloné sur le poste Windows. La clé SSH a été créée et ajoutée au compte GitHub, et le test `ssh -T git@github.com` a réussi.

> [!WARNING]
> La documentation des étapes et les captures doivent encore être complétées. L'installation de Kubernetes et de Rancher reste à effectuer et à valider.

**Dernière mise à jour :** 2026-10-26.
