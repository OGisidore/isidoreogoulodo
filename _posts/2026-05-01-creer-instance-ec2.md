---
title: Comment créer une instance EC2 sur AWS
date: 2026-05-01 20:10:00 +0100
categories: [cloud, aws]
author: isidore
tags: [aws, ec2, vpc, ssh, iam, devops]
image:
  path: /assets/img/headers/aws-ec2.png
pin: false
---

Amazon EC2 permet de lancer rapidement des serveurs virtuels dans le cloud. Dans ce guide, nous allons créer une instance EC2 de façon simple, puis nous connecter en SSH et appliquer les bonnes pratiques de base.

### Prérequis

- Un compte AWS actif
- Un utilisateur IAM avec permissions EC2 (éviter d'utiliser le compte root)
- Une paire de clés SSH (ou en créer une pendant le lancement)
- Un VPC disponible (par défaut ou personnalisé)

### Étape 1: Ouvrir le service EC2

Dans la console AWS:

1. Aller sur **EC2**
2. Cliquer sur **Launch instance**
3. Donner un nom à l'instance (ex: `web-ec2-dev`)

### Étape 2: Choisir l'AMI et le type d'instance

- **AMI (Image)**: Ubuntu Server 22.04 LTS (ou Amazon Linux)
- **Instance type**: `t2.micro` (ou `t3.micro`) pour les tests

Ces types sont généralement éligibles au free tier selon ton compte/région.

### Étape 3: Configurer l'accès SSH

Dans **Key pair (login)**:

- Choisir une clé existante, ou
- Créer une nouvelle clé (format `.pem`) et la télécharger

Conserve ce fichier de clé en sécurité, tu en auras besoin pour la connexion SSH.

### Étape 4: Configurer le réseau et la sécurité

Dans **Network settings**:

- Sélectionner ton **VPC**
- Laisser **Auto-assign public IP** activé si tu veux te connecter depuis Internet
- Créer/choisir un **Security Group** avec au minimum:
  - SSH (TCP 22) depuis ton IP uniquement (recommandé)

Optionnel pour un serveur web:

- HTTP (TCP 80)
- HTTPS (TCP 443)

### Étape 5: Stockage et lancement

- Taille disque: 8 à 20 GiB (gp3) selon ton besoin
- Vérifier le récapitulatif
- Cliquer sur **Launch instance**

Une fois lancée, récupère l'IP publique ou le DNS public depuis la page de l'instance.

### Étape 6: Se connecter à l'instance (SSH)

```bash
chmod 400 /path/to/my-key.pem
ssh -i /path/to/my-key.pem ubuntu@<PUBLIC_IP>
```

Notes:

- Utiliser `ubuntu` pour Ubuntu AMI
- Utiliser `ec2-user` pour Amazon Linux

### Création en CLI (optionnel)

Tu peux aussi lancer une instance en ligne de commande:

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxxxxxxxxxx \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-xxxxxxxxxxxxxxxxx \
  --subnet-id subnet-xxxxxxxxxxxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-ec2-dev}]'
```

### Bonnes pratiques après création

- Mettre à jour le système:

```bash
sudo apt update && sudo apt upgrade -y
```

- Désactiver l'accès root SSH
- Restreindre les ports ouverts aux stricts besoins
- Associer un rôle IAM à l'instance au lieu de stocker des clés AWS
- Activer la surveillance (CloudWatch) et des sauvegardes régulières

Avec cette base, tu peux déployer une API, un backend web, ou un environnement de test rapidement.
