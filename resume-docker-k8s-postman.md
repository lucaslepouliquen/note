# Notes Docker, Kubernetes & Postman

## Docker

### Commandes de base

#### Gestion des conteneurs
```bash
# Lancer un conteneur
docker run <image>

# Lancer en mode interactif
docker run -it <image>

# Lancer avec mapping de ports
docker run -p <host-port>:<container-port> <image>

# Lister les conteneurs actifs
docker ps

# Lister tous les conteneurs
docker ps -a

# Arrêter un conteneur
docker stop <container-id>

# Supprimer un conteneur
docker rm <container-id>
```

#### Gestion des images
```bash
# Lister les images
docker images

# Télécharger une image
docker pull <image>

# Supprimer une image
docker rmi <image>

# Construire une image
docker build -t <name> .
```

### Dockerfile

Structure type d'un Dockerfile:

```dockerfile
# Image de base
FROM ubuntu:latest

# Définir le point d'entrée
ENTRYPOINT ["sleep"]

# Commande par défaut
CMD ["5"]

# Exécuter des commandes
RUN <command>

# Copier des fichiers
COPY <source> <destination>

# Définir des variables d'environnement
ENV APP_COLOR=pink
```

### Docker Compose

```yaml
version: '3'

services:
  webapp:
    image: webapp-color
    ports:
      - "8080:80"
    environment:
      - APP_COLOR=blue
    depends_on:
      - redis
  
  redis:
    image: redis
```

Commandes Docker Compose:
```bash
docker-compose up
docker-compose down
```

---

## Kubernetes

### Commandes kubectl de base

#### Gestion des Pods
```bash
# Créer un pod
kubectl run <pod-name> --image=<image>

# Lister les pods
kubectl get pods

# Voir les détails d'un pod
kubectl describe pod <pod-name>

# Supprimer un pod
kubectl delete pod <pod-name>

# Appliquer un fichier YAML
kubectl apply -f <file.yaml>

# Créer depuis un fichier
kubectl create -f <file.yaml>
```

#### Gestion des Deployments
```bash
# Créer un deployment
kubectl create deployment <name> --image=<image>

# Lister les deployments
kubectl get deployments

# Mettre à l'échelle
kubectl scale deployment <name> --replicas=<count>

# Mise à jour d'image
kubectl set image deployment/<name> <container>=<new-image>

# Voir le statut d'un rollout
kubectl rollout status deployment/<name>

# Annuler un rollout
kubectl rollout undo deployment/<name>
```

#### Gestion des Services
```bash
# Exposer un deployment
kubectl expose deployment <name> --port=<port> --type=<type>

# Lister les services
kubectl get services
```

#### Gestion des Namespaces
```bash
# Lister les namespaces
kubectl get namespaces

# Créer un namespace
kubectl create namespace <name>

# Travailler dans un namespace
kubectl get pods --namespace=<name>
# ou
kubectl get pods -n <name>
```

#### Configuration et contexte
```bash
# Voir la configuration
kubectl config view

# Changer de contexte
kubectl config use-context <context-name>

# Obtenir le contexte actuel
kubectl config current-context
```

### Fichiers de configuration YAML

#### Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
spec:
  containers:
  - name: my-container
    image: nginx
    ports:
    - containerPort: 80
```

#### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: my-container
        image: nginx
```

#### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort  # ou ClusterIP, LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # optionnel pour NodePort
```

#### ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: my-container
        image: nginx
```

### Concepts importants

#### Types de Services
- **ClusterIP**: Service accessible uniquement en interne (défaut)
- **NodePort**: Service accessible via un port sur chaque nœud
- **LoadBalancer**: Service avec un load balancer externe

#### Rolling Updates & Rollbacks
- Strategy: `RollingUpdate` ou `Recreate`
- Les deployments permettent des mises à jour progressives
- Possibilité de revenir en arrière avec `rollout undo`

#### Labels et Selectors
- Les labels permettent d'identifier et de grouper les ressources
- Les selectors permettent de cibler des ressources spécifiques
- Essentiels pour lier Services, Deployments et Pods

---

## Postman

### Concepts de base

#### Collections
- Groupement de requêtes API
- Possibilité d'organiser par dossiers
- Partage et collaboration en équipe

#### Variables d'environnement
```javascript
// Définir une variable
pm.environment.set("variable_name", "value");

// Obtenir une variable
pm.environment.get("variable_name");

// Utiliser dans les requêtes
{{variable_name}}
```

### Scripts de test

#### Pre-request Scripts
Exécutés avant l'envoi de la requête

```javascript
// Exemple: générer un timestamp
pm.environment.set("timestamp", Date.now());
```

#### Tests (Post-response)
Exécutés après réception de la réponse

```javascript
// Vérifier le status code
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

// Vérifier le contenu JSON
pm.test("Response has data", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.name).to.eql("expected_value");
});
```

### Organisation des requêtes

#### Types de requêtes HTTP
- GET: Récupérer des données
- POST: Créer des données
- PUT: Mettre à jour des données
- PATCH: Modification partielle
- DELETE: Supprimer des données

#### Structure d'une requête
1. **URL**: Point d'entrée de l'API
2. **Headers**: Métadonnées (Content-Type, Authorization, etc.)
3. **Body**: Données envoyées (JSON, form-data, etc.)
4. **Query Parameters**: Paramètres dans l'URL

### Bonnes pratiques

- Utiliser des variables pour les URLs de base
- Créer des environnements (dev, staging, prod)
- Organiser les requêtes en collections logiques
- Ajouter des tests pour valider les réponses
- Documenter les collections pour l'équipe

---

### Kubernetes - Concepts avancés

#### ReplicaSet
- **Replicas**: Nombre d'instances souhaitées (ex: 3)
- Commandes principales:
  ```bash
  kubectl scale --replicas=6 -f replicaset-definition.yaml
  kubectl get replicaset
  kubectl get application controller
  kubectl get pods
  ```
- Utilise un sélecteur pour identifier les pods à gérer
- `apiVersion: apps/v1`

#### Node Selector & Affinity
```yaml
# Node Selector (simple)
spec:
  nodeSelector:
    size: Large
```

**Node Affinity** (plus flexible):
- `affinity:` - `nodeAffinity:`

**Types:**
- `requiredDuringSchedulingIgnoredDuringExecution` - Type 1
- `preferredDuringSchedulingIgnoredDuringExecution` - Type 2
- `requiredDuringSchedulingRequiredDuringExecution` - (futur)
- `preferredDuringSchedulingRequiredDuringExecution` - (futur)

**Opérateurs:**
- `In`, `NotIn`, `Exists`

#### Taints et Tolerations
```bash
# Appliquer un taint sur un nœud
kubectl taint nodes <node-name> key=value:taint-effect

# Exemple
kubectl taint nodes node01 app=blue:NoSchedule
```

**Taint Effects:**
- `NoSchedule`: Ne planifie pas de nouveaux pods
- `PreferNoSchedule`: Évite de planifier si possible
- `NoExecute`: Évacue les pods existants non tolérants

**Toleration dans le Pod:**
```yaml
spec:
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

#### Resource Requests & Limits
```yaml
spec:
  containers:
  - name: my-container
    resources:
      requests:
        memory: "1Gi"
        cpu: "1"
      limits:
        memory: "2Gi"
        cpu: "2"
```

**Limit Ranges** (par namespace):
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```

#### DaemonSets
- Déploie un pod sur **chaque nœud** du cluster
- Cas d'usage: monitoring, logs, network, kube-proxy
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring
  template:
    # ... pod template
```

#### Static Pods
- Gérés directement par le kubelet (pas par le control plane)
- Définis dans `/etc/kubernetes/manifests` ou chemin configuré
- Utiles pour les composants du control plane eux-mêmes

#### Multiple Schedulers
- Possibilité d'avoir plusieurs schedulers personnalisés
- Spécifier le scheduler dans le pod:
```yaml
spec:
  schedulerName: my-custom-scheduler
```

---

## Cloud Native & OpenShift

### Construire un Cloud privé OpenShift

**Migration vers le cloud privé:**
- Évaluation des applications existantes
- Stratégie de migration progressive

**Construction Cloud + Migration:**
- Rythme d'application : 50-60 applications migrées par mois
- Plateforme fiable et équipe dédiée (Charwal)

**Ensemble système informatique:**
- Architecture distribuée et scalable

### Cartographie applicative

**Chiffres clés:**
- **2250 applications** (combinaison requête + extractions)
- **8268 applications** au total : **26548 paquets**
- Architecture hors-scope complète

**Types d'architecture:**

1. **Microservices + Architecture clean**
   - Services découplés et indépendants
   
2. **Concurrence/Contenteurisation**
   - Applications en conteneurs
   - Fusion/Rewrite d'applications obsolètes
   - Résolution de problèmes de performance

### Service Accounts & RBAC

#### Création de Service Account
```bash
# Créer un service account
kubectl create serviceaccount <name>

# Voir les service accounts
kubectl get serviceaccount
```

#### Accounts (Comptes utilisateurs)
- **Admin**: Administrateurs système
- **User**: Développeurs et utilisateurs
- **Developer**: Équipe de développement
- **kube-apiserver**: Composant API Kubernetes

**Services et authentification:**
- Static password file
- Static token file  
- Static certificate (certificates, Identity Server)
- `--basic-auth-file=user-details.csv`

**Kube-apiserver.yaml:**
```bash
curl -v -k https://master-node-ip:6443/api/v1/pods -u "user1:password"
```

#### Étude de cas
- Mise en pratique de la configuration
- Exercices sur l'authentification

**Affinity:**
- Contrôle du placement des pods sur les nœuds

---

## Kubernetes - Stockage

### Storage Classes & Dynamic Provisioning

#### Concepts de base
- **PV**: Persistent Volume
- **PVC**: Persistent Volume Claim
- **Access Modes**: ReadWriteOnce, ReadOnlyMany, ReadWriteMany

#### Dynamic Provisioning
Création automatique de volumes à la demande

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

#### Google Cloud Beta
```bash
# Créer un disque compute
gcloud beta compute disks create \
  --size=<GB> \
  --region=us-east1 \
  <pd-disk>
```

#### Kubectl pour webapp avec logs
```bash
kubectl exec webapp -- cat /log/app.log
```

### Persistent Volume (PV)

#### Définition YAML
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

**Options apiVersion:**
- `apiVersion: v1`
- `kind: PersistentVolume`

**Metadata:**
- `name: pv-vol1`

**Spec:**
```yaml
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

**Autres options:**
- `awsElasticBlockStore`
- `volumeId`
- `fsType`

#### Kubectl create -f

```bash
kubectl create -f pv-definition.yaml
kubectl get persistentvolume
```

### Persistent Volume Claim (PVC)

#### Définition YAML
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

**Caractéristiques:**
- (ReadOnlyMany, ReadWriteMany)
- Resources: Requests + Storage

**apiVersion:**
- `apiVersion: v1`
- `kind: PersistentVolumeClaim`

**Metadata:**
- `name: myclaim`

**Spec:**
```yaml
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

**Autres configurations:**
- `volumeName`: Lier à un PV spécifique
- `volumeMode`: `Filesystem` ou `Block`
- `storageClassName`: Nom de la storage class
- Volumes par nom ou par label

#### Kubectl get persistentvolume

```bash
kubectl get persistentvolumeclaim
```

### Kubernetes - Binds

**PV to PVC with Sufficient Capacity, Access Modes, Volume Modes, Storage Class Selection:**

Les bindings sont basés sur :
- Capacité suffisante
- Modes d'accès compatibles
- Volume modes
- Sélection de Storage Class

#### Utilisation de labels et selectors

**One-to-one relationship between Claim and Volume:**
Relation 1:1 entre PVC et PV

**Que se passe-t-il quand un claim est supprimé ?**

**Comportements du persistentVolumeReclaimPolicy:**
- **Retain**: Le volume persiste (défaut)
- **Delete**: Le volume est supprimé
- **Recycle**: Les données sont effacées et le volume réutilisé

### Volumes dans les Pods

#### apiVersion: v1
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
        - mountPath: "/var/www/html"
          name: data-volume
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: myclaim
```

**Volumes:**
- `name`: Nom du volume
- Type: `hostPath` ou `persistentVolumeClaim`

**Sous-options:**
- `path`: `/data`
- `type: Directory`

**awsElasticBlockStore:**
- `volumeId`
- `fsType`

#### Volume Name

**Période (period):**
- Gestion du cycle de vie des volumes

### Network Policies (suite)

#### Kubectl - PVC et espaces critiques

**URL parties:**
- Kubectl get po (pods)

**Deny:**
- kubectl get order
- kubectl get deploy

**Allow (autoriser):**
- `policyTypes:` → `Ingress`
- `Ingress` → `Ingress`
- `Traffic` → `Ingress`
- `From Pod` → `podSelector`
- `From Pod` → selection avec `matchLabels` et `name: api-pod`

**Flux réseau:**
```
Web-port (80) → Pod (5000) → DB (3306)
                  ↓
            Network Policy
```

#### Add: Egress et Policy Types

**Egress:**
- `-/to`
- `- ipBlock:`
  - `cidr: 192.168.5.10/32`
- `ports:` TCP
- `port: 80`

### Ingress (détails avancés)

#### Problème au niveau des routes middleware

**Structure de base:**
```
─ ── ▼ vine
─ ── ─ spec
─ ── ─ Rules
```

**HTTP avec paths:**
- HTTP
- Paths:
  - `path: /wear`
  - `backend:`
    - `serviceName: wear-service`
    - `servicePort: 80`

**Statefulset:**
- Déploiement ordonné et stable
- Host-based deployment

#### Kubectl describe ingress, ingress-wear-watch

```bash
kubectl describe ingress <ingress-name>

# 404 not found (troubleshooting)

# Ingress - wear - watch - yaml:
# apiVersion
# kind
# metadata
# spec
```

**Structure YAML complète:**
```yaml
apiVersion: v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  rules:
  - http:
      paths:
      - path: /wear
        backend:
          serviceName: wear-service
          servicePort: 80
      - path: /watch
        backend:
          serviceName: watch-service
          servicePort: 80
```

#### Service account pour Ingress

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

**Deployment + Service + ConfigMap + Auth:**
- Configuration complète de l'ingress controller

#### Ingress Resource

```yaml
apiVersion: extensions/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  backend:
    serviceName: wear-service
    servicePort: 80
```

**Kubectl pour Ingress:**
```bash
kubectl get ingress

# Rules management:
# Rule 1
# Rule 2
# Rule 3
# Rule 4
```

### Kubectl exec webapp - cat / log / app.log

```bash
# Exécuter une commande dans un conteneur pour voir les logs
kubectl exec webapp -- cat /log/app.log
```

### Google Cloud (gcloud)

#### Créer des disques compute

```bash
# Google Cloud Beta - Compute disks create
gcloud beta compute disks create \
  --size=<GB> \
  --region=us-east1 \
  <pd-disk>
```

**Options:**
- `--size`: Taille en GB
- `--region`: Région (ex: us-east1)
- Nom du disque persistant

### Storage Class - Dynamic Provisioning

#### PVC (Persistent Volume Claim) détails

**Access Modes:**
- `pvc`
- `accessModes`

**Dynamic provisioning:**
- `storageClassName`

**Types disponibles:**
- `type: pd-standard` (disque standard)
- `stateful set` (pour StatefulSets)
- `0`, `1`, `2` (indices)
- `master`

#### Kubectl scale up/down - Reverse order when not used

```bash
# Scale up (augmenter le nombre de replicas)
kubectl scale statefulset <name> --replicas=5

# Scale down (réduire - ordre inverse)
# Effet: Change PoI Many general Policy
```

**Comportement:**
- Les StatefulSets scalent dans l'ordre inverse lors du scale down
- Order preservation pour les pods

### Add grand Cluster

**Opérations:**
- `op: EX18P, EX2338`
- `CHAD`
- `User-ilelectrires/`

**Voucher:**
- `25 October`

#### Créer un header bearer avec ClusterRole & Node

**Add serviceName (deployment):**

```bash
# Créer un service account
kubectl create serviceaccount <name>

# Obtenir le service account
kubectl get serviceaccount
```

**Accounts (Comptes):**
- **Admin** (Administrateurs)
- **User** (Utilisateurs)
- **Developer** (Développeurs)
- **kube-apiserver** (Composant API)

**Services et authentification:**
- Static password file
- Static certificate: certificates, Identity Server
- `--basic-auth-file=user-details.csv`

**Kube-apiserver.yaml:**
```bash
# Commande d'accès avec authentification
curl -v -k

# Mise en pratique de l'init command
# Exercice pratique
# Étude de cas
# APIncii:
```

---

## Kubernetes - Réseau (suite)

### Network Policies

Les Network Policies contrôlent le trafic réseau entre les pods dans un cluster Kubernetes.

#### Commandes de base
```bash
# Obtenir les network policies
kubectl get networkpolicies
kubectl get netpol

# Définition YAML basique
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
      ports:
        - protocol: TCP
          port: 3306
```

#### Types de Policy

**Policy Types:**
- **Ingress**: Trafic entrant vers les pods
- **Egress**: Trafic sortant des pods

**Pod Selector:**
```yaml
spec:
  podSelector:
    matchLabels:
      role: db
```

**Ingress Rules:**
```yaml
ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
      - protocol: TCP
        port: 3306
```

**Solutions supportant Network Policies:**
- Kube-router
- Calico
- Romana
- Weave-net

**Note:** Flannel ne supporte PAS les Network Policies

#### Egress & Ingress avancés

**Egress (sortie):**
```yaml
egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
  - ports:
    - protocol: TCP
      port: 80
```

**CIDR (Classless Inter-Domain Routing):**
- Format: `192.168.5.10/32`
- `cidr: 132.12.10.5.10/32` (exemple)

**Ports:**
- `protocol: TCP`
- `port: 80`

#### Création de Network Policy

```bash
# Créer depuis un fichier YAML
kubectl create -f policy-definition.yaml

# Exemple avec plages IP
# 12.1-186
```

### Ingress

Ingress expose les routes HTTP et HTTPS depuis l'extérieur du cluster vers les services.

#### Concepts de base

**URL pour accès critique + service:**
- Gestion des routes externes
- Load balancing intégré

#### Commandes Ingress

```bash
# Obtenir les pods (avec namespace)
kubectl get po -A

# Deny (refuser le trafic)
kubectl get order
kubectl get deploy
kubectl get service
```

#### Types de trafic

**Allow (autoriser):**
- `Ingress` → `Ingress`
- `Traffic` → `Ingress`

**Deny (refuser):**
- `From Pod`
- `From Pod` → `podSelector`
  - `matchLabels`
  - `name: api-pod`

**Web Port (port externe):**
- Port: 80 → Pod: 5000
- Port: 5000 → DB: 3306
  - Pod: 3306

**Network Policy:**
- Labels et sélecteurs
- `role: db`
- Protection avec `matchLabels`
  - `name: api-pod`
  - `role: db`

#### Kubectl pour Network Policies

```bash
kubectl get networkpolicy
# ou
kubectl get netpol
```

#### Diagramme de flux réseau

```
Web-port (80) -----> Pod (5000) -----> DB (3306)
                      |
                   Network
                   Policy
```

### Problèmes réseau et middleware

#### Problème au niveau des routes middleware

**Schéma:**
```
─ ── ▼
─ ── ─▼e.vine
─ ── ─ spec
```

**Rules (règles):**
- HTTP
- Paths
  - `path: /wear`
  - `backend:`
    - `serviceName: wear-service`
    - `servicePort: 80`

**Statefulset:**
- Déploiement ordonné
- Host-based deployment

#### Kubectl describe ingress

```bash
# Voir les détails d'un ingress
kubectl describe ingress <ingress-name>

# Exemple de vérification
# 404 not found
```

**Ingress - Wear - Watch - Yaml:**
```yaml
apiVersion: extensions/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  rules:
  - http:
      paths:
      - path: /wear
        backend:
          serviceName: wear-service
          servicePort: 80
      - path: /watch
        backend:
          serviceName: watch-service
          servicePort: 80
```

### Service Account pour Ingress

#### Définition

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

**Deployment + Service + ConfigMap + Auth:**

#### Ingress Resource

```yaml
apiVersion: extensions/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  backend:
    serviceName: wear-service
    servicePort: 80
```

#### Kubectl pour Ingress

```bash
# Obtenir les ingress
kubectl get ingress

# Rules:
# Rule 1
# Rule 2
# Rules
# Rule 4
```

---

## Domain-Driven Design (DDD)

### Concepts de base

#### In Memory (données temporaires)
- **SGBD**: Système de Gestion de Base de Données
- Données stockées en mémoire pour accès rapide

#### Elastic Search & Elastic Stack
- Moteur de recherche et d'analyse
- **Datalog**: Journalisation des données
- **Signature**: Identification unique

#### Trois niveaux CVE (Common Vulnerabilities and Exposures)
- **125.11**: Niveau critique
- **252.11**: Niveau important
- Classification des vulnérabilités de sécurité

#### Patterns architecturaux

**Peut CPU (Patronage)**:
- Architecture légère pour optimiser les ressources

**Builder Pattern**:
- Construction d'objets complexes étape par étape
- Sépare la construction de la représentation

**Pipelines (Open Shift)**:
- CI/CD automatisé
- Déploiement continu

**DevOps & Cloud**:
- Microservices architecture
- **EKS** (Elastic Kubernetes Service) - AWS
- **ECS** (Elastic Container Service) - AWS
- **ECR** (Elastic Container Registry) - AWS

**Domain-Driven Design concepts:**
- **Bounded Context**: Limite explicite d'un modèle de domaine
- **Aggregates**: Groupe d'objets traités comme une unité
- **Entities**: Objets avec identité unique
- **Value Objects**: Objets sans identité, définis par leurs attributs

---

## Kubernetes - Monitoring & Health Checks

### Probes (Sondes de santé)

#### Liveness Probe
- Vérifie si le conteneur est vivant
- Redémarre le conteneur si la vérification échoue
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

#### Readiness Probe
- Vérifie si le conteneur est prêt à recevoir du trafic
- Retire le pod du service si la vérification échoue
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

#### Startup Probe
- Pour les applications avec démarrage lent
- Les autres probes sont désactivées jusqu'à la réussite de la startup probe

#### Types de vérifications
```yaml
# HTTP GET
httpGet:
  path: /api/health
  port: 8080

# TCP Socket
tcpSocket:
  port: 3306

# Command exec
exec:
  command:
  - cat
  - /tmp/healthy
```

### Tests de connectivité

#### HTTP Test
- Vérification via requêtes HTTP
- Status codes: 200-399 = succès

#### TCP Test
- Test de connexion au port
- Vérifie si le port écoute

#### Exec Command
- Exécute une commande dans le conteneur
- Exit code 0 = succès

---

## Kubernetes - Services Avancés

### Horizontal Pod Autoscaler (HPA)
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Labels & Selectors avancés
```yaml
# Sélection multiple
selector:
  matchLabels:
    app: myapp
    tier: frontend
  matchExpressions:
  - key: environment
    operator: In
    values: ["prod", "staging"]
```

### Annotations
- Métadonnées supplémentaires (non utilisées pour la sélection)
```yaml
metadata:
  annotations:
    buildVersion: "1.34"
    team: "platform"
```

### Rolling Updates
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Pods supplémentaires pendant la mise à jour
      maxUnavailable: 0  # Pods indisponibles pendant la mise à jour
```

### Ports & Exposition
- **Port**: Port du service (8080)
- **TargetPort**: Port du conteneur (80)
- **NodePort**: Port exposé sur le nœud (30000-32767)

### Région & Efficacité
- Distribution multi-régions pour la haute disponibilité
- Affinité pods/nœuds pour l'optimisation des performances

---

## Ressources utiles

### Documentation officielle
- Docker: https://docs.docker.com/
- Kubernetes: https://kubernetes.io/docs/
- Postman: https://learning.postman.com/
- DDD: https://martinfowler.com/tags/domain%20driven%20design.html

### Commandes de dépannage

#### Docker
```bash
docker logs <container-id>
docker exec -it <container-id> /bin/bash
docker inspect <container-id>
```

#### Kubernetes
```bash
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash
kubectl describe <resource-type> <resource-name>
kubectl get events
kubectl top pods  # Voir utilisation CPU/RAM
kubectl top nodes # Voir utilisation des nœuds
```

### Bonnes pratiques Kubernetes

1. **Toujours définir des resource requests et limits**
2. **Utiliser des health checks (liveness/readiness probes)**
3. **Implémenter le rolling update pour les déploiements**
4. **Utiliser des namespaces pour l'isolation**
5. **Appliquer des labels cohérents pour l'organisation**
6. **Configurer le HPA pour l'autoscaling**
7. **Utiliser des secrets pour les données sensibles**
8. **Mettre en place des NetworkPolicies pour la sécurité**
