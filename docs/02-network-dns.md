
# Étape 2 — Configuration réseau et DNS avec Netplan

## 1. Objectif

Configurer les adresses IP statiques des trois machines Ubuntu Server 24.04, vérifier la passerelle et les DNS, puis valider la communication entre les nœuds avant l'installation de Kubernetes.

## 2. Vérifier les paramètres réseau dans VMware

Avant de modifier la configuration réseau d'Ubuntu, vérifier le mode réseau des trois machines virtuelles dans VMware Workstation.

Les captures suivantes servent à documenter la configuration réellement utilisée. Elles doivent provenir de ton propre environnement VMware.

### 2.1 — Paramètres réseau de la machine virtuelle

Dans les paramètres de chaque VM, ouvrir **Network Adapter** et vérifier :

- que l'adaptateur réseau est activé ;
- que l'option de connexion au démarrage est activée ;
- le mode réseau sélectionné : NAT, Bridged ou Host-only ;
- que les trois VM utilisent un réseau permettant leur communication.

**Capture 1 — Paramètres de la machine virtuelle**

![Paramètres réseau de la machine virtuelle](images/network-dns/vm-settings/paramettre-de-la-machine-virtuelle.png)

Fichier attendu :

`docs/images/network-dns/vm-settings/paramettre-de-la-machine-virtuelle.png`

**Capture 2 — Adaptateur réseau et mode de connexion**

![Adaptateur réseau VMware](images/network-dns/vm-settings/Virtual-Network-Adaptateur.png)

Fichier attendu :

`docs/images/network-dns/vm-settings/Virtual-Network-Adaptateur.png`

Le mode NAT est souvent pratique pour un laboratoire nécessitant un accès Internet et des communications entre les machines virtuelles. Le choix doit cependant correspondre à la configuration réelle de VMware.

### 2.2 — Virtual Network Editor

Dans VMware Workstation, ouvrir :

**Edit → Virtual Network Editor**

Identifier le réseau virtuel utilisé par les trois VM.

**Capture 3 — Vue d'ensemble du Virtual Network Editor**

![Virtual Network Editor](images/network-dns/virtual-network-editor/Virtual-Network-Editor.png)

Fichier attendu :

`docs/images/network-dns/virtual-network-editor/Virtual-Network-Editor.png`

Si les VM utilisent le réseau NAT, généralement nommé `VMnet8`, ouvrir ses paramètres et relever :

- le sous-réseau et le masque ;
- l'adresse de la passerelle NAT ;
- l'état du service DHCP ;
- la plage d'adresses distribuées par DHCP.

**Capture 4 — Paramètres NAT de VMnet8**

![Paramètres NAT de VMnet8](images/network-dns/virtual-network-editor/VMnet8-NAT-Settings.png)

Fichier attendu :

`docs/images/network-dns/virtual-network-editor/VMnet8-NAT-Settings.png`

**Capture 5 — Paramètres DHCP de VMnet8**

![Paramètres DHCP de VMnet8](images/network-dns/virtual-network-editor/VMnet8-DHCP-Settings.png)

Fichier attendu :

`docs/images/network-dns/virtual-network-editor/VMnet8-DHCP-Settings.png`

Si ton environnement VMware utilise d'autres noms ou une autre interface, adapte les noms des fichiers et les chemins des images à tes captures réelles.

> [!WARNING]
> Ne configure pas les adresses IP statiques avant d'avoir confirmé le sous-réseau, la passerelle et la plage DHCP. Les captures ne doivent pas être des illustrations génériques : elles doivent correspondre à ta configuration réelle.

## 3. Plan d'adressage prévisionnel

| Machine | Nom d'hôte | Interface | Adresse IP prévue |
|---|---|---|---|
| Control Plane | `k8s-server-01` | `ens33` | `192.168.189.10/24` |
| Worker 01 | `k8s-server-02` | `ens33` | `192.168.189.11/24` |
| Worker 02 | `k8s-server-03` | `ens33` | `192.168.189.12/24` |

Paramètres envisagés, à confirmer :

- Passerelle : `192.168.189.2`
- DNS primaire : `1.1.1.1`
- DNS secondaire : `8.8.8.8`
- Sous-réseau : `192.168.189.0/24`

Ces valeurs sont provisoires. Elles doivent être confirmées dans VMware et sur Ubuntu avant toute modification.

## 4. Diagnostic initial dans Ubuntu

Sur chaque machine, exécuter :

```bash
ip -br address
ip route
resolvectl status
hostnamectl
ls -la /etc/netplan/
sudo netplan get
sudo ufw status verbose
```

Consigner les informations suivantes :

- adresse IP actuelle ;
- nom de l'interface réseau ;
- passerelle par défaut ;
- serveurs DNS ;
- fichiers Netplan présents ;
- état du pare-feu UFW.

Si `ens33` n'affiche pas d'adresse IP, vérifier que l'adaptateur est connecté dans VMware et que la VM utilise le bon réseau virtuel. Ne pas modifier Netplan avant d'avoir compris l'état actuel.

### 4.1 — Captures de diagnostic Ubuntu

Après avoir pris les captures VMware, compléter cette section avec les captures réelles de ton terminal Ubuntu.

**Capture 6 — État du réseau avant modification**

![Diagnostic réseau Ubuntu avant modification](images/network-dns/ubuntu-network-before.png)

**Capture 7 — Fichiers Netplan existants**

![Fichiers Netplan](images/network-dns/ubuntu-netplan-files.png)

Ces deux captures sont à ajouter après avoir exécuté les commandes de diagnostic. Si les images ne sont pas encore disponibles, conserver les emplacements comme éléments à compléter.

## 5. Sauvegarde de la configuration Netplan

Avant toute modification, identifier le fichier Netplan réellement utilisé.

Créer un dossier de sauvegarde :

```bash
sudo mkdir -p /root/netplan-backup
```

Lister les fichiers YAML existants :

```bash
find /etc/netplan -maxdepth 1 -type f -name '*.yaml' -print
```

Si des fichiers YAML sont présents, les sauvegarder :

```bash
sudo cp -a /etc/netplan/*.yaml /root/netplan-backup/
```

Si aucun fichier YAML n'existe, ne pas poursuivre avec cette commande : examiner la configuration avant de créer un fichier.

Ne pas supprimer les fichiers existants sans avoir déterminé leur rôle. Si cloud-init gère la configuration réseau, vérifier comment il intervient avant de modifier durablement Netplan.

## 6. Configuration Netplan

Après confirmation du sous-réseau, de la passerelle et de l'interface réseau, adapter la configuration à chaque machine.

Exemple pour le Control Plane, uniquement si les paramètres prévus sont confirmés :

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
```

Pour les Workers, remplacer l'adresse IP :

- Worker 01 : `192.168.189.11/24`
- Worker 02 : `192.168.189.12/24`

La passerelle et les DNS doivent rester cohérents avec le réseau VMware réel.

Ne remplace pas aveuglément le contenu d'un fichier existant. Examine sa configuration, sauvegarde-le, puis adapte les paramètres nécessaires en conservant les autres réglages utiles.

**Capture 8 — Exemple de configuration Netplan**

![Configuration Netplan avec adresse IP statique](images/network-dns/ubuntu-netplan-static-ip.png)

Cette capture est à ajouter après avoir préparé et vérifié la configuration dans ton environnement. Masquer toute information sensible avant de la publier.

## 7. Permissions et application

Sur chaque machine, appliquer les permissions recommandées au fichier YAML concerné, en adaptant son nom.

Exemple, uniquement si ce fichier existe et contient la configuration utilisée :

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
```

Valider la syntaxe :

```bash
sudo netplan generate
```

Si la validation réussit et que la console VMware reste disponible pour récupérer la machine en cas de perte du réseau, appliquer les changements :

```bash
sudo netplan apply
```

Vérifier ensuite les adresses et la route par défaut :

```bash
ip -br address
ip route
```

Si la machine perd sa connectivité, utiliser la console VMware pour diagnostiquer et corriger la configuration.

## 8. Tests de validation

### 8.1 — Vérifier l'adresse IP et la route

```bash
ip -br address
ip route
```

Vérifier que l'adresse IP appartient au sous-réseau prévu et que la route par défaut utilise la passerelle correcte.

### 8.2 — Tester la communication entre les nœuds

Depuis le Control Plane, une fois les adresses réellement configurées :

```bash
ping -c 3 192.168.189.11
ping -c 3 192.168.189.12
```

Si le sous-réseau réel est différent, remplacer ces adresses par celles attribuées aux Workers.

Un échec de `ping` ne prouve pas à lui seul que le réseau est défaillant : certains équipements bloquent les requêtes ICMP. Vérifier également les adresses IP, les routes et les règles du pare-feu.

### 8.3 — Tester la résolution DNS

```bash
getent hosts archive.ubuntu.com
```

Une adresse IP renvoyée indique que la résolution du nom a fonctionné.

### 8.4 — Vérifier l'accès réseau HTTPS

Si la résolution DNS fonctionne, tester l'accès à une ressource HTTPS :

```bash
curl -I --max-time 10 https://archive.ubuntu.com/
```

Si `curl` n'est pas installé, noter le résultat et installer l'outil lorsque l'accès aux dépôts sera disponible.

### 8.5 — Captures après validation

**Capture 9 — Adresses IP et route après configuration**

![Validation du réseau Ubuntu](images/network-dns/ubuntu-network-validation.png)

**Capture 10 — Test DNS**

![Test de résolution DNS](images/network-dns/ubuntu-dns-test.png)

**Capture 11 — Connectivité entre les nœuds**

![Test de connectivité entre les nœuds Kubernetes](images/network-dns/kubernetes-nodes-connectivity.png)

Ces captures sont à prendre après les tests réels. Ne pas ajouter d'image simulant un résultat qui n'a pas été obtenu.

## 9. Critères de validation

- [ ] Chaque machine possède le nom d'hôte attendu.
- [ ] Chaque machine possède une adresse IP unique.
- [ ] Le sous-réseau correspond à la configuration VMware.
- [ ] La passerelle correspond au réseau VMware.
- [ ] La route par défaut est présente.
- [ ] La résolution DNS fonctionne.
- [ ] Les nœuds peuvent communiquer entre eux.
- [ ] L'état du pare-feu UFW est documenté.
- [ ] La configuration reste valide après redémarrage.
- [ ] Les captures correspondent à la configuration réellement utilisée.

Ne déclarer l'étape réseau terminée qu'après avoir vérifié les résultats sur les trois machines.

## 10. Organisation des captures

Les captures réseau doivent être placées dans cette structure :

```text
docs/
└── images/
    └── network-dns/
        ├── vm-settings/
        │   ├── paramettre-de-la-machine-virtuelle.png
        │   └── Virtual-Network-Adaptateur.png
        ├── virtual-network-editor/
        │   ├── Virtual-Network-Editor.png
        │   ├── VMnet8-NAT-Settings.png
        │   └── VMnet8-DHCP-Settings.png
        ├── ubuntu-network-before.png
        ├── ubuntu-netplan-files.png
        ├── ubuntu-netplan-static-ip.png
        ├── ubuntu-network-validation.png
        ├── ubuntu-dns-test.png
        └── kubernetes-nodes-connectivity.png
```

Les fichiers d'images doivent être ajoutés au dépôt pour que les liens Markdown s'affichent dans GitHub. Les chemins et noms de fichiers sont sensibles à la casse : ils doivent correspondre exactement aux noms enregistrés.

Avant de publier les captures, vérifier qu'elles ne contiennent pas de mots de passe, de jetons, de clés privées ou d'autres informations sensibles.





<!-- # Étape 2 — Configuration réseau et DNS avec Netplan

## 1. Objectif

Configurer les adresses IP statiques des trois machines Ubuntu Server 24.04, vérifier la passerelle et les DNS, puis valider la communication entre les nœuds avant l'installation de Kubernetes.

## 2. Plan d'adressage

| Machine | Nom d'hôte | Interface | Adresse IP prévue |
|---|---|---|---|
| Control Plane | `k8s-server-01` | `ens33` | `192.168.189.10/24` |
| Worker 01 | `k8s-server-02` | `ens33` | `192.168.189.11/24` |
| Worker 02 | `k8s-server-03` | `ens33` | `192.168.189.12/24` |

Paramètres envisagés, à confirmer :

- Passerelle : `192.168.189.2`
- DNS primaire : `1.1.1.1`
- DNS secondaire : `8.8.8.8`
- Sous-réseau : `192.168.189.0/24`

**Ne pas appliquer ce plan avant d'avoir confirmé le réseau VMware.**

## 3. Diagnostic initial

Sur chaque machine, exécuter :

```bash
ip -br address
ip route
resolvectl status
hostnamectl
ls -la /etc/netplan/
sudo netplan get
sudo ufw status verbose
```

Consigner les adresses existantes, l'interface réseau, la passerelle, les DNS, les fichiers Netplan et l'état du pare-feu.

## 4. Sauvegarde de la configuration

Avant toute modification, identifier le fichier Netplan réellement utilisé.

Créer un dossier de sauvegarde :

```bash
sudo mkdir -p /root/netplan-backup
```

Copier les fichiers YAML existants dans ce dossier :

```bash
sudo cp -a /etc/netplan/*.yaml /root/netplan-backup/
```

Si aucun fichier YAML n'existe, ne pas poursuivre avec cette commande : examiner la configuration avant de créer un fichier.

Ne pas supprimer les fichiers existants sans avoir déterminé leur rôle.

## 5. Configuration Netplan

Après confirmation du sous-réseau et de la passerelle, adapter la configuration à l'interface réelle.

Exemple pour le Control Plane, si les paramètres prévus sont confirmés :

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
```

Pour les Workers, remplacer uniquement l'adresse IP par celle attribuée à chaque machine :

- Worker 01 : `192.168.189.11/24`
- Worker 02 : `192.168.189.12/24`

La passerelle et les DNS doivent rester cohérents avec le réseau VMware réel.

Si le fichier est géré par cloud-init, vérifier la manière dont cloud-init gère le réseau avant de modifier durablement la configuration.

## 6. Permissions et application

Sur chaque machine, appliquer les permissions recommandées au fichier YAML concerné, en adaptant son nom :

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
```

Cette commande n'est valable que si ce fichier existe et contient la configuration utilisée.

Valider la syntaxe :

```bash
sudo netplan generate
```

Si la validation réussit et que la console VMware reste disponible pour récupérer la machine en cas de problème :

```bash
sudo netplan apply
```

## 7. Tests de validation

### 7.1 Vérifier l'adresse IP et la route

```bash
ip -br address
ip route
```

### 7.2 Tester la communication entre les nœuds

Depuis le Control Plane :

```bash
ping -c 3 192.168.189.11
ping -c 3 192.168.189.12
```

### 7.3 Tester la résolution DNS

```bash
getent hosts archive.ubuntu.com
```

### 7.4 Vérifier l'accès réseau

Si la résolution DNS fonctionne, tester ensuite l'accès à une ressource HTTPS :

```bash
curl -I --max-time 10 https://archive.ubuntu.com/
```

Si `curl` n'est pas installé, ne pas interrompre la configuration réseau pour autant : noter le résultat et installer l'outil lorsque l'accès aux dépôts sera disponible.

## 8. Critères de validation

- [ ] Chaque machine possède le nom d'hôte attendu.
- [ ] Chaque machine possède une adresse IP unique.
- [ ] La passerelle correspond au réseau VMware.
- [ ] La route par défaut est présente.
- [ ] La résolution DNS fonctionne.
- [ ] Les nœuds peuvent communiquer entre eux.
- [ ] L'état du pare-feu UFW est documenté.
- [ ] La configuration reste valide après redémarrage.

Ne déclarer l'étape réseau terminée qu'après avoir vérifié les résultats sur les trois machines. -->