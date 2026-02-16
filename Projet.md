![](media/image1.jpeg){width="3.52742125984252in"
height="1.2708333333333333in"}

Année :2025-2026 Prof: M. Azeddine KHIAT

Etudiant : TOE Lawanabienko Ange Gael

4eme Année Cybersécurité

# Projet :

# Mise en œuvre d'une infrastructure cloud de supervision centralisée sous AWS avec Zabbix

------------------------------------------------------------------------

![](media/image2.png){width="2.321428258967629in"
height="2.629861111111111in"}
![](media/image3.png){width="2.370138888888889in"
height="2.5970417760279965in"}

# PLAN

[Projet : [1](#projet)](#projet)

[Mise en œuvre d'une infrastructure cloud de supervision centralisée
sous AWS avec Zabbix
[1](#mise-en-œuvre-dune-infrastructure-cloud-de-supervision-centralisée-sous-aws-avec-zabbix)](#mise-en-œuvre-dune-infrastructure-cloud-de-supervision-centralisée-sous-aws-avec-zabbix)

[Introduction [3](#introduction)](#introduction)

[Part 1. Architecture et Configuration Réseau
[3](#architecture-et-configuration-réseau)](#architecture-et-configuration-réseau)

[Part 2. Instances EC2 [8](#instances-ec2)](#instances-ec2)

[Part 3. Déploiement du Serveur Zabbix via Docker
[9](#déploiement-du-serveur-zabbix-via-docker)](#déploiement-du-serveur-zabbix-via-docker)

[Step 1. Installer Docker et Compose :
[9](#installer-docker-et-compose)](#installer-docker-et-compose)

[Step 2. Vérifier l'accès [12](#vérifier-laccès)](#vérifier-laccès)

[Part 4. Configuration des Agents
[12](#configuration-des-agents)](#configuration-des-agents)

[Step 1. Agent Linux [12](#agent-linux)](#agent-linux)

[Step 2. Configuration du service
[14](#configuration-du-service)](#configuration-du-service)

[Step 3. Procédure d'installation de l'agent Zabbix sur Windows
[16](#procédure-dinstallation-de-lagent-zabbix-sur-windows)](#procédure-dinstallation-de-lagent-zabbix-sur-windows)

[Part 5. Monitoring et Validation
[19](#monitoring-et-validation)](#monitoring-et-validation)

[Part 6. Conclusion [20](#conclusion)](#conclusion)

# Introduction

Dans le cadre de la gestion moderne des infrastructures informatiques,
nous avons choisi de mettre en place une solution de supervision
centralisée afin de garantir la disponibilité et la performance des
services.\
Notre projet consiste à déployer un serveur **Zabbix** conteneurisé sur
**AWS**, supervisant un parc hybride composé de clients Linux et
Windows. L'utilisation de Docker nous offre une flexibilité accrue et
une simplification de la gestion des dépendances.

> ***Lien github : <https://github.com/Tag95/projet-zabbix>***

![](media/image4.png){width="6.3in" height="6.3in"}

------------------------------------------------------------------------

## Architecture et Configuration Réseau

Nous avons commencé par créer un **VPC** dédié au projet, avec un bloc
CIDR 10.0.0.0/16.

![](media/image5.png){width="6.3in" height="3.5430555555555556in"}\
Ensuite, nous avons ajouté un **subnet** (10.0.0.0/24) afin d'isoler les
ressources.

![](media/image6.png){width="6.3in" height="2.8868055555555556in"}

![](media/image7.png){width="6.3in" height="2.2527777777777778in"}

![](media/image8.png){width="6.3in" height="1.1708333333333334in"}

Pour sécuriser l'infrastructure, nous avons défini deux **Security
Groups** :

> **Étape 1 : Accéder à la console**

Nous nous connectons à la **console AWS**, puis nous ouvrons le service
**EC2**. Dans le menu de gauche, nous sélectionnons **Security Groups**
afin de créer un nouveau groupe de sécurité.

> **Étape 2 : Créer un Security Group**

1.  Nous cliquons sur **Create security group**.

2.  Nous renseignons les champs :

    - **Name** : Zabbix-Server-SG (pour le serveur) ou Agents-SG (pour
      les clients).

    - **Description** : autorisation du trafic pour Zabbix.

    - **VPC** : nous choisissons le VPC que nous avons créé (ex.
      *VPC_Projet_Zabbix*).

![](media/image9.png){width="6.3in" height="2.1631944444444446in"}

> **Étape 3 : Ajouter les règles entrantes**

Nous configurons les règles de trafic entrant en fonction du rôle de
l'instance :

- **Pour le serveur Zabbix (Zabbix-Server-SG)** :

  - Port 22 (SSH) → accès administrateur.

  - Ports 80 et 443 (HTTP/HTTPS) → accès à l'interface web Zabbix.

  - Port 10051 (TCP) → communication avec les agents Zabbix.

![](media/image10.png){width="6.3in" height="3.1284722222222223in"}

- **Pour les clients (Agents-SG)** :

  - Port 22 (SSH) → administration des machines Linux.

  - Port 3389 (RDP) → accès distant aux machines Windows.

  - Port 10050 (TCP) → écoute de l'agent Zabbix.

![](media/image11.png){width="6.3in" height="1.7520833333333334in"}

![](media/image12.png){width="6.3in" height="2.915277777777778in"}

**Étape 4 : Ajouter les règles sortantes**

Par défaut, AWS autorise tout le trafic sortant. Nous avons conservé
cette configuration afin de permettre aux instances de communiquer
librement avec Internet (mises à jour, téléchargement de paquets, etc.).

**Mise en place de la Passerelle Internet (IGW)**

1.  **VPC → Internet Gateways → Create internet gateway**

    - Nom : IGW_Zabbix

2.  **Attach to VPC** → rattacher au VPC du projet.

3.  **Route Tables → Edit routes → Add route**

    - Destination : 0.0.0.0/0

    - Target : IGW_Zabbix

![](media/image13.png){width="6.3in" height="1.4222222222222223in"}

![](media/image14.png){width="6.3in" height="1.74375in"}

Enfin, nous avons configuré une **Internet Gateway** et une **table de
routage** avec une route par défaut (0.0.0.0/0) vers l'IGW, garantissant
la connectivité Internet des instances.

![](media/image15.png){width="6.3in" height="1.63125in"}

![](media/image16.png){width="6.3in" height="2.1284722222222223in"}

![](media/image17.png){width="6.3in" height="2.2270833333333333in"}

------------------------------------------------------------------------

## Instances EC2

Nous avons déployé trois instances EC2 :

- Un **serveur Zabbix** (Ubuntu 22.04, type t3.large).

- Un **client Linux** (Ubuntu 22.04, type t3.medium).

- Un **client Windows** (Windows Server 2022, type t3.large).

Chaque instance a été associée au Security Group approprié et validée
dans la console AWS avec l'état **Running**.

![](media/image18.png){width="6.3in" height="1.4715277777777778in"}

------------------------------------------------------------------------

## Déploiement du Serveur Zabbix via Docker

Nous nous sommes connectés en SSH au serveur Ubuntu et avons installé
Docker et Docker Compose. Nous avons vérifié l'état des conteneurs avec
docker ps, puis accédé à l'interface Zabbix via l'adresse IP publique
sur le port 80.

### Installer Docker et Compose :

sudo apt update && sudo apt install docker.io docker-compose -y

![](media/image19.png){width="6.3in" height="3.0208333333333335in"}

![](media/image20.png){width="6.3in" height="0.61875in"}

Créer un fichier docker-compose.yml avec services : MySQL,
Zabbix-server, Zabbix-web.

Lancer : **sudo docker-compose up -d**

Nous avons structuré notre infrastructure dans un fichier
docker-compose.yml. Ce fichier définit trois services essentiels : le
serveur Zabbix, l'interface Web (Nginx) et une base de données MySQL
persistante. Pour des raisons de sécurité, les informations sensibles
(mots de passe, utilisateurs) ne sont pas inscrites en dur mais stockées
dans un fichier de variables d'environnement nommé .env. 

![](media/image21.png){width="5.925513998250219in"
height="0.2583552055993001in"}

services:

zabbix-db:

image: mysql:8.0

environment:

\- MYSQL_USER=\${MYSQL_USER}

\- MYSQL_PASSWORD=\${MYSQL_PASSWORD}

\- MYSQL_ROOT_PASSWORD=\${MYSQL_ROOT_PASSWORD}

\- MYSQL_DATABASE=\${MYSQL_DATABASE}

volumes:

\- ./zabbix-db-data:/var/lib/mysql

zabbix-server:

image: zabbix/zabbix-server-mysql:ubuntu-6.4-latest

ports:

\- \"10051:10051\"

environment:

\- DB_SERVER_HOST=zabbix-db

\- MYSQL_USER=\${MYSQL_USER}

\- MYSQL_PASSWORD=\${MYSQL_PASSWORD}

\- MYSQL_DATABASE=\${MYSQL_DATABASE}

depends_on:

\- zabbix-db

zabbix-web:

image: zabbix/zabbix-web-nginx-mysql:ubuntu-6.4-latest

ports:

\- \"80:8080\"

environment:

\- ZBX_SERVER_HOST=zabbix-server

\- DB_SERVER_HOST=zabbix-db

\- MYSQL_USER=\${MYSQL_USER}

\- MYSQL_PASSWORD=\${MYSQL_PASSWORD}

\- MYSQL_DATABASE=\${MYSQL_DATABASE}

\- PHP_TZ=Europe/Paris

depends_on:

\- zabbix-db

\- zabbix-server

![](media/image22.png){width="6.404166666666667in"
height="2.7708333333333335in"}

Puis on fait : **nano .env**

![](media/image23.png){width="5.600485564304462in"
height="0.1833497375328084in"}

![](media/image24.png){width="5.692159886264217in"
height="1.8834962817147856in"}

Lancement et Vérification 

Le déploiement est initié via la commande suivante : **docker - compose
up -d**

![](media/image25.png){width="6.3in" height="2.654166666666667in"}

![](media/image26.png){width="6.3in" height="0.5576388888888889in"}

### Vérifier l'accès

- Ouvre ton navigateur et tape : http://\<IP_PUBLIQUE\>

- Tu devrais voir l'interface de connexion Zabbix

**Utilisateur** : Admin (avec un A majuscule)

**Mot de passe** : zabbix

![](media/image27.png){width="6.3in" height="3.3784722222222223in"}

------------------------------------------------------------------------

## Configuration des Agents

Sur le **client Linux**, nous avons ajouté le dépôt officiel Zabbix,
installé l'agent et modifié le fichier /etc/zabbix/zabbix_agentd.conf
afin de pointer vers l'IP du serveur.\
Sur le **client Windows**, nous avons téléchargé le MSI depuis le site
officiel, installé l'agent via RDP et configuré le fichier
zabbix_agent2.conf.

### Agent Linux

Installer dépôt Zabbix :

**wget
https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb**

**sudo dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb**

**sudo apt update**

![](media/image28.png){width="6.3in" height="2.0444444444444443in"}

![](media/image29.png){width="6.3in" height="2.7430555555555554in"}

Une fois le dépôt configuré, nous installons le paquet zabbix-agent : 

**sudo apt install zabbix - agent -y**

**Modifier /etc/zabbix/zabbix_agentd.conf :**

**Code : Server=\<IP_Zabbix\>**

> **ServerActive=\<IP_Zabbix\>**
>
> **Hostname=Client-Linux**

![](media/image30.png){width="6.3in" height="6.041666666666667in"}

Configuration du service 

### Configuration du service 

La communication entre l'agent et le serveur repose sur la modification
du fichier de configuration principal situé dans /etc/zabbix/. 

Après on fait la change le fichier de configuration :

sudo nano /etc/zabbix/zabbix_agentd.conf

on modifie les variable suivante :

**Server=Ip du serveur**

**ServerActive=ip du serveur**

**Hostname=Nom_De_Cette_Instance**

Et on redemarre le service :

**sudo systemctl restart zabbix-agent**

**sudo systemctl enable zabbix-agent**

![](media/image31.png){width="6.3in" height="0.19194772528433945in"}

![](media/image31.png){width="6.270833333333333in"
height="1.3895833333333334in"}

#### Étape A : Accéder à la configuration des hôtes

- Dans l'interface web Zabbix, nous allons dans le menu :
  **Configuration → Hosts**. 👉 Cette section permet de gérer tous les
  hôtes supervisés.

> ![](media/image32.png){width="6.3in" height="3.029861111111111in"}

#### Étape B: Créer un nouvel hôte

- Nous cliquons sur **Create host** (bouton en haut à droite).

- Une fenêtre s'ouvre avec plusieurs champs à renseigner.

> **Étape C : Paramétrer l'hôte Linux**

1.  **Hostname** : Client Linux

2.  **Groups** : nous ajoutons l'hôte au groupe *Linux servers*.

3.  **Interfaces** :

    - Type : Agent

    - IP : adresse privée ou publique du client Linux

    - Port : 10050

4.  **Templates** : nous associons le template *Linux by Zabbix agent
    active*. 👉 Cela permet de collecter automatiquement les métriques
    CPU, RAM, disque et réseau.

![](media/image33.png){width="6.3in" height="4.465277777777778in"}

![](media/image34.png){width="6.3in" height="2.951388888888889in"}

### Procédure d'installation de l'agent Zabbix sur Windows

Très bien Gaël 👌, détaillons la **procédure complète d'installation de
l'agent Zabbix sur Windows**, étape par étape, en style académique avec
le "nous".

#### Étape 1 : Télécharger l'agent

1.  Nous nous rendons sur le site officiel de Zabbix :
    <https://www.zabbix.com/download_agents>.

2.  Nous choisissons la version correspondant à notre serveur (ex.
    Zabbix 6.4 ou 7.0).

3.  Nous téléchargeons le fichier MSI adapté à notre architecture
    (64-bit pour Windows Server 2016/2019/2022).

👉 Le fichier téléchargé est par exemple :
zabbix_agent2-6.4.10-windows-amd64-openssl.msi.
[Zabbix](https://www.zabbix.com/documentation/current/en/manual/installation/install_from_packages/win_msi)
[Serverspace](https://serverspace.io/support/help/installing-zabbix-agent-on-windows/)

------------------------------------------------------------------------

#### Étape 2 : Installation via l'assistant graphique

1.  Nous exécutons le fichier MSI téléchargé.

2.  L'assistant d'installation nous guide :

    - Choix du répertoire d'installation (par défaut C:\\Program
      Files\\Zabbix Agent 2).

    - Configuration initiale (adresse du serveur Zabbix, port, nom
      d'hôte).

3.  Nous validons et lançons l'installation.

![](media/image35.png){width="5.29212489063867in"
height="4.258701881014873in"}

![](media/image36.png){width="5.400468066491689in"
height="4.650402449693789in"}

#### Étape 3 : vérification du service

vérifions son état :

net Start \"Zabbix Agent 2\"

![](media/image37.png){width="6.333333333333333in"
height="2.0972222222222223in"}

#### Étape 4 : Enregistrement dans Zabbix

- Dans l'interface graphique Zabbix : **Configuration → Hosts → Create
  host**.

- Hostname : Client-Windows.

- Interface : IP du client Windows, port 10050.

- Template : *Windows by Zabbix agent active*.

![](media/image38.png){width="6.3in" height="3.5791666666666666in"}

![](media/image39.png){width="6.3in" height="2.05625in"}

------------------------------------------------------------------------

## Monitoring et Validation

Nous avons créé un **Dashboard Global** via **Monitoring → Dashboards →
Create dashboard**, nommé *Global Infrastructure Monitoring*.\
Nous y avons ajouté deux widgets :

- Un graphique CPU (item CPU utilization, hôtes Linux et Windows).

- Un graphique RAM (item Memory utilization, hôtes Linux et Windows).

Nous avons ensuite mis en place un **Trigger proactif** :

- Menu **Data collection → Hosts → Client Linux → Triggers → Create
  trigger**.

- Nom : *High CPU utilization on {HOST.NAME}*.

- Sévérité : High (rouge).

- Expression : last(/Linux:CPU utilization)\>10.

![](media/image40.png){width="6.3in" height="3.4097222222222223in"}

Ce trigger nous a permis de simuler une alerte de surcharge CPU et de
valider la capacité d'auto-surveillance du système.

------------------------------------------------------------------------

## Conclusion

Nous avons atteint nos objectifs techniques : déploiement d'un serveur
Zabbix conteneurisé sur AWS, configuration des agents Linux et Windows,
création d'un dashboard global et mise en place de triggers proactifs.

Nous avons rencontré certaines difficultés, notamment le téléchargement
du MSI Windows marqué par des avertissements, le transfert via RDP,
ainsi que des erreurs de configuration liées à l'utilisation de
127.0.0.1 au lieu de l'IP réelle. Nous avons résolu ces problèmes en
téléchargeant les fichiers depuis la source officielle, en utilisant RDP
pour le transfert et en corrigeant les paramètres Server et
ServerActive.

Enfin, nous envisageons d'étendre la supervision à d'autres services
(bases de données, applications métiers), d'ajouter des notifications
par e-mail, et de mettre en place une haute disponibilité pour Zabbix.
