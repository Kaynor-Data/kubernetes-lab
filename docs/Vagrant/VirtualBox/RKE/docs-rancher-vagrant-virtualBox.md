# Configuration locale de Rancher avec Vagrant, VirtualBox et RKE

## 1. Présentation

Ce document présente une méthode alternative de déploiement d'un cluster Kubernetes local à partir du projet communautaire suivant :

**Dépôt source :** [SUSE-Rancher-Community/local-setup-of-rancher-with-rke](https://github.com/SUSE-Rancher-Community/local-setup-of-rancher-with-rke)

Ce projet automatise la création de machines virtuelles avec **Vagrant et VirtualBox**, déploie un cluster Kubernetes avec **Rancher Kubernetes Engine (RKE)** et décrit l'installation de **Rancher** pour administrer le cluster depuis une interface Web.

Cette documentation présente l'architecture et les principaux fichiers du dépôt original. Elle constitue une référence distincte du laboratoire principal, qui utilise VMware Workstation, Ubuntu Server et `kubeadm`.

## 2. Architecture du projet original

Le dépôt configure trois machines virtuelles :

| Machine virtuelle | Rôle Kubernetes | Adresse IP | Mémoire configurée | CPU configurés |
|---|---|---|---|---|
| `master` | Control Plane et etcd | `172.16.129.21` | Environ 4 Go | 2 |
| `worker1` | Worker | `172.16.129.22` | Environ 4 Go | 2 |
| `worker2` | Worker | `172.16.129.23` | Environ 4 Go | 2 |

### Technologies utilisées

| Technologie | Fonction |
|---|---|
| Vagrant | Création et configuration automatisées des machines virtuelles |
| VirtualBox | Exécution des machines virtuelles |
| openSUSE Leap 15.2 | Système d'exploitation défini dans le `Vagrantfile` original |
| Docker Engine | Runtime utilisé par la configuration RKE d'origine |
| RKE | Déploiement du cluster Kubernetes |
| Helm | Installation des applications Kubernetes, notamment Rancher |
| cert-manager | Gestion des certificats TLS utilisés par Rancher |

> [!NOTE]
> Les versions et configurations mentionnées correspondent aux fichiers récupérés dans le dépôt original. Elles ne constituent pas nécessairement des recommandations pour un nouveau déploiement.

## 3. Organisation des fichiers

Les principaux fichiers du dépôt sont les suivants :

| Fichier | Fonction |
|---|---|
| `.gitignore` | Exclut certains fichiers générés, notamment l'état RKE et le fichier kubeconfig du cluster |
| `Vagrantfile` | Définit les machines virtuelles, leurs adresses IP, leur mémoire, leurs CPU et leur provisionnement |
| `node_script.sh` | Automatise la préparation initiale des machines |
| `cluster.yml` | Décrit les nœuds, les rôles et plusieurs paramètres du cluster Kubernetes pour RKE |
| `README.md` | Présente les prérequis et les étapes d'installation |
| `rancher-logo.png` | Illustration du projet |
| `rancher-landing-page.png` | Illustration de la page d'accueil de Rancher |

## 4. Création des machines virtuelles

Le fichier `Vagrantfile` définit trois machines virtuelles :

- `master` : nœud de contrôle et membre etcd ;
- `worker1` : premier nœud de travail ;
- `worker2` : deuxième nœud de travail.

Chaque machine reçoit une adresse IP privée et des ressources CPU et mémoire définies dans la configuration.

Le fichier utilise également un script de provisionnement nommé `node_script.sh`, exécuté lors de la configuration des machines.

### Commandes utilisées

À la racine du projet, l'auteur utilise :

```bash
vagrant up
```

Pour vérifier l'état des machines :

```bash
vagrant status
```

Pour ouvrir une session sur une machine :

```bash
vagrant ssh master
```

Le nom `master` peut être remplacé par `worker1` ou `worker2`.

> [!NOTE]
> Ces commandes appartiennent à la méthode Vagrant/VirtualBox. Elles ne sont pas nécessaires pour créer les machines virtuelles déjà installées manuellement dans le laboratoire VMware.

## 5. Préparation des nœuds

Le fichier `node_script.sh` automatise plusieurs opérations sur les machines :

- configuration de l'accès SSH ;
- installation et démarrage de Docker Engine ;
- désactivation du swap ;
- modification de certains paramètres réseau du noyau ;
- activation de l'acheminement IPv4 ;
- désactivation du pare-feu UFW.

Ces opérations préparent les machines à la procédure de déploiement prévue par le projet original.

> [!WARNING]
> Le script original modifie également les paramètres d'authentification SSH et configure un mot de passe administrateur commun. Il désactive le pare-feu sans mettre en place de règles restrictives. Ces choix ne doivent pas être reproduits tels quels sur un système exposé ou utilisé en production.
>
> Le système openSUSE Leap 15.2 indiqué dans le `Vagrantfile` est ancien. Il faut vérifier la disponibilité du système, sa maintenance et sa compatibilité avec les composants envisagés avant de reproduire cette installation.

## 6. Configuration du cluster avec RKE

Le fichier `cluster.yml` décrit les paramètres du cluster Kubernetes.

Il définit notamment :

- les adresses IP des trois nœuds ;
- les utilisateurs utilisés pour les connexions SSH ;
- les rôles `controlplane`, `etcd` et `worker` ;
- le nom du cluster : `rancher-cluster` ;
- les plages d'adresses des services et des pods ;
- le plugin réseau Calico ;
- le fournisseur DNS CoreDNS ;
- le mode d'autorisation RBAC ;
- le fournisseur de métriques `metrics-server`.

### Déploiement

Le dépôt décrit la génération ou la préparation du fichier `cluster.yml`, puis le lancement du déploiement avec RKE :

```bash
rke config
rke up
```

La commande `rke config` permet de répondre à une série de questions pour générer la configuration du cluster. Si le fichier est déjà préparé, cette étape peut être remplacée par sa vérification et son adaptation.

Après le déploiement, RKE génère notamment :

- `cluster.rkestate` : fichier décrivant l'état du cluster ;
- `kube_config_cluster.yml` : fichier permettant à `kubectl` de se connecter au cluster.

### Vérification

Le kubeconfig doit être placé dans le répertoire attendu par `kubectl`, après avoir sauvegardé tout fichier de configuration existant.

Les commandes de vérification indiquées par le projet sont :

```bash
kubectl cluster-info
kubectl config current-context
```

> [!IMPORTANT]
> Le dépôt utilise RKE1. Cette procédure ne doit pas être confondue avec une installation Kubernetes utilisant `kubeadm`. RKE1 a atteint sa fin de vie : il convient de vérifier les possibilités de maintenance et d'étudier une solution adaptée avant tout nouveau déploiement.

## 7. Installation de Rancher

Une fois le cluster disponible, le projet décrit l'installation de Rancher avec Helm.

Les grandes étapes sont les suivantes :

1. Installer cert-manager pour gérer les certificats.
2. Créer le namespace `cattle-system`.
3. Ajouter le dépôt Helm de Rancher.
4. Installer Rancher avec un nom d'hôte adapté à l'environnement.
5. Configurer la résolution de ce nom d'hôte depuis le poste client.
6. Accéder à Rancher depuis un navigateur Web.

Le dépôt original utilise notamment un manifeste cert-manager versionné et le dépôt Helm `rancher-stable`.

> [!WARNING]
> Les commandes originales font référence à des versions anciennes, notamment cert-manager `v1.4.0`. Elles sont présentées ici pour décrire le projet source, et non comme une procédure d'installation actuelle. Pour une nouvelle installation, vérifier les versions compatibles de Kubernetes, Rancher, Helm et cert-manager dans leurs documentations officielles.

## 8. Différences avec le laboratoire principal

Les deux méthodes répondent à un objectif similaire, mais leurs outils et leurs procédures diffèrent.

| Élément | Projet original | Laboratoire principal |
|---|---|---|
| Hyperviseur | VirtualBox | VMware Workstation |
| Automatisation des VM | Vagrant | Création manuelle des VM |
| Système d'exploitation | openSUSE Leap 15.2 dans la configuration d'origine | Ubuntu Server |
| Déploiement Kubernetes | RKE1 | `kubeadm` |
| Runtime de conteneurs | Docker Engine dans la configuration RKE d'origine | containerd |
| Réseau des VM | `172.16.129.0/24` | `192.168.189.0/24` |
| Administration | Rancher | Rancher, après déploiement du cluster |

Les commandes, les adresses IP, les fichiers de configuration et les étapes d'installation ne sont donc pas interchangeables sans adaptation.

## 9. Références

- [Dépôt original : Local Setup of Rancher with RKE](https://github.com/SUSE-Rancher-Community/local-setup-of-rancher-with-rke)
- [Documentation de RKE](https://rke.docs.rancher.com/)
- [Documentation de Rancher](https://ranchermanager.docs.rancher.com/)
- [Documentation de Helm](https://helm.sh/docs/)

---

**Conclusion :** ce projet constitue une référence pour comprendre une ancienne méthode de déploiement local de Kubernetes et d'installation de Rancher. Il est documenté séparément afin de conserver une distinction claire entre cette approche et celle retenue pour le laboratoire principal.