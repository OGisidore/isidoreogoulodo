---
title: Cluster K3s haute disponibilité
date: 2026-05-01 14:00:00 -0000
categories: [kubernetes, k3s]
author: Wolo leonardo
tags: [homelab, proxmox, kubernetes, k3s, kube-vip, metallb, nginx]
image:
  path: /assets/img/headers/k8s-blocks.webp
pin: true
page_id: post-k3s-ha
permalink: /posts/k3s-ha-install/
lang-exclusive: ['fr-FR']
---

k3s est une distribution Kubernetes légère, pensée pour les environnements aux ressources limitées (edge, IoT, homelab). Ce guide décrit comment monter un cluster k3s **hautement disponible** sur trois nœuds avec une base embarquée, **kube-vip** pour exposer l’API du plan de contrôle sur une IP virtuelle, et **MetalLB** pour équilibrer les services de type LoadBalancer.

### Prérequis

Avant de commencer, vérifie les points suivants :

- Trois machines Linux (Ubuntu 22.04 ou plus récent), 2 vCPU et 2 Go de RAM minimum
- Accès SSH à toutes les machines
- Un utilisateur avec sudo sur chaque nœud
- Les adresses IP de chaque nœud

### Étape 1 : Configurer le nœud master

On utilise **k3sup** pour installer k3s et initialiser le cluster.

<aside>
⚠️ Adapte les valeurs suivantes à ton environnement :

- IP du nœud master : `--ip = 192.168.1.151`
- Utilisateur sudo : `--user leonardo`
- IP virtuelle kube-vip (TLS SAN) : `--tls-san = 192.168.1.150`

La liste complète des options est sur le dépôt [k3sup](https://github.com/alexellis/k3sup#do-you-love-k3sup).

</aside>

```bash
k3sup install \
    --cluster \
    --ip 192.168.1.151 \
    --user leonardo \
    --tls-san 192.168.1.150 \
    --context k3s-cluster-1 \
    --local-path ~/.kube/config \
    --k3s-extra-args '--disable servicelb,traefik' \
    --k3s-channel stable
```

### Étape 2 : Déployer kube-vip en **DaemonSet**

On configure kube-vip pour la HA du plan de contrôle k3s. Après l’installation sur le premier control plane, les autres nœuds rejoignent le cluster via l’**IP virtuelle**.

**Appliquer le RBAC**

Applique d’abord le manifeste RBAC officiel : 

```bash
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
```

****

**Télécharger l’image et créer un alias**

Connecte-toi au premier master en SSH et exécute ces commandes en root.

```bash
ssh 192.168.1.151
```

```bash
sudo -i
```

```bash
export KVVERSION=v0.6.0
```

```bash
# Pull image - Check latest version on github or refer to docs.
ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
```

```bash
# Create alias for the Kube-VIP command
alias kube-vip="ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
```

<aside>
⚠️ Adapte notamment :

- IP virtuelle kube-vip : `-address 192.168.1.150`
- Interface réseau (`ip a` / `ifconfig`) : `-interface eth0`
</aside>

```bash
# Generate and deploy the manifest
kube-vip manifest daemonset \
    --interface eth0 \
    --address 192.168.1.150 \
    --inCluster \
    --taint \
    --controlplane \
    --arp \
    --leaderElection | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml
```

Le manifeste est déposé sous `/var/lib/rancher/k3s/server/manifests/` : k3s l’applique automatiquement. Vérifie que tout tourne avant de continuer.

```bash
# Start pinging the virtual ip
ping 192.168.1.150 -c 100
```

```bash
# Check if the daemonset is running
kubectl get daemonset -A
```

```bash
# Check the logs of kube-vip
kubectl -n kube-system logs <podname>
```

**Mettre à jour `~/.kube/config`**

Remplace l’adresse du serveur par l’IP virtuelle kube-vip : `192.168.1.150`.

### Étape 3 : Ajouter d’autres nœuds master

Toujours avec **k3sup**, on joint les masters supplémentaires au cluster existant.

<aside>
⚠️ Exemple de valeurs :

- IP des nouveaux masters : `192.168.1.152`, `192.168.1.153`
- IP du serveur (kube-vip) : `-server-ip 192.168.1.150`
- Utilisateur sudo : `-user leonardo`
</aside>

**Installer k3s et rejoindre le cluster (masters)**

```bash
# Join the second master node
k3sup join \
    --ip 192.168.1.152 \
    --server-ip 192.168.1.150 \
    --server \
    --sudo \
    --user leonardo \
    --k3s-extra-args '--disable servicelb,traefik' \
    --k3s-channel stable
```

```bash
# Join the third master node
k3sup join \
    --ip 192.168.1.153 \
    --server-ip 192.168.1.150 \
    --server \
    --sudo \
    --user leonardo \
    --k3s-extra-args '--disable servicelb,traefik' \
    --k3s-channel stable
```

**Vérifier que les masters sont prêts**

```bash
kubectl get nodes
```

### Étape 4 : Tainter les nœuds master

Le *tainting* des masters pour éviter d’y planifier des charges applicatives peut poser problème selon ton modèle (addons système, etc.). Si tu suis cette approche :

```bash
# Tainter chaque nœud master
kubectl taint nodes <master-node-name> CriticalAddonsOnly=true:NoExecute
```

### Étape 5 : Ajouter les nœuds worker

Les workers rejoignent le cluster via l’IP virtuelle du control plane.

<aside>
⚠️ Exemple :

- Workers : `192.168.1.161` … `192.168.1.164`
- Serveur : `-server-ip 192.168.1.150`
- Utilisateur : `-user leonardo`
- Les agents héritent souvent de `--k3s-extra-args` adaptés (pas de servicelb/traefik sur les masters, etc.).
</aside>

**Rejoindre le cluster (workers)**

```bash
# Join the first worker node
k3sup join \
    --ip 192.168.1.161 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

```bash
# Join the second worker node
k3sup join \
    --ip 192.168.1.162 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

```bash
# Join the third worker node
k3sup join \
    --ip 192.168.1.163 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

```bash
# Join the fourth worker node
k3sup join \
    --ip 192.168.1.164 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

**Vérifier les workers**

```bash
kubectl get nodes
```

### Étape 6 : Déployer MetalLB

MetalLB fournit des services **LoadBalancer** en couche 2 (ou 3) dans un cluster bare-metal / homelab.

**Installer les manifests**

```bash
# Apply the metallb configurations
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```

**Définir le pool d’adresses**

Pour attribuer des IP aux services, déclare un `IPAddressPool` : MetalLB puisera dans cette plage.

```yaml
# metallb-addrespool.yml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: loadbalancer-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.245-192.168.1.250
```

Tu peux combiner plusieurs pools, plages ou CIDR (IPv4/IPv6).

**Annoncer les IP (mode L2)**

Une fois une IP assignée à un service, elle doit être annoncée sur le réseau. Ici on utilise la **couche 2** : souvent aucun protocole routage supplémentaire, MetalLB répond aux ARP avec le bon MAC.

Associe un `L2Advertisement` à ton `IPAddressPool`.

```yaml
# metallb-l2advertisement.yml
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: loadbalancer-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - loadbalancer-pool
```

Sans sélecteur, l’`L2Advertisement` s’applique à tous les pools ; sinon, précise les noms ou des labels.

**Appliquer les CR MetalLB**

```bash
kubectl apply -f metallb-addrespool.yml
```

```bash
kubectl apply -f metallb-l2advertisement.yml
```

**Contrôler MetalLB**

```bash
kubectl -n metallb-system get daemonsets
```

```bash
kubectl -n metallb-system logs <podname>
```

### Étape 7 : **Test avec Nginx**

On valide le parcours LoadBalancer avec un déploiement Nginx simple.

**Manifests**

```yaml
# deployment.yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

```yaml
# service.yml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

**Déployer**

```bash
kubectl apply -f deployment.yml
```

```bash
kubectl apply -f service.yml
```

**Lire l’IP MetalLB**

```bash
kubectl describe service nginx
```

**Tester depuis le réseau**

```bash
# Remplace par l’IP externe affichée par MetalLB
curl 192.168.0.XXX
```

**Nettoyage**

```bash
kubectl delete deployment,service nginx
```

## Conclusion

Un cluster **k3s** HA avec **kube-vip** et **MetalLB** donne une base légère mais solide pour des charges proches de la prod : basculement de l’API, services exposés proprement, sans l’empreinte d’une distribution « full » Kubernetes.

Idéal pour homelab, edge et apprentissage. Pense à la surveillance, aux sauvegardes et éventuellement au GitOps pour industrialiser.

**Bon cluster !**
