# LAB 9.1 - Ordonnancement & Santé des Pods

Cet atelier vous guidera à travers les mécanismes permettant de contrôler précisément **où** vos Pods s'exécutent (Scheduling) et comment Kubernetes surveille leur **état de santé** (Probes).

**Pré-requis** : Un cluster avec 1 Master et 3 Workers (`worker-node01`, `worker-node02`, `worker-node03`).

**Objectifs**:

* Manipuler les **Labels** pour organiser l'infrastructure.
* Utiliser les **Taints & Tolerations** pour réserver des nœuds.
* Utiliser l'**Affinity** pour attirer des Pods et l'**Anti-Affinity** pour assurer la haute disponibilité.
* Configurer des sondes **Liveness** et **Readiness** pour l'auto-réparation.

---

## Partie 1 : Préparation (Labels)

Les labels sont la base du scheduling. Nous allons simuler une infrastructure hétérogène.

1. Ajoutez les labels suivants à vos nœuds :
* `worker-node01` : `disk=hdd`, `env=prod`
* `worker-node02` : `disk=ssd`, `env=staging`
* `worker-node03` : `disk=ssd`, `env=prod`



<details>
<summary>Correction Commandes </summary>

```bash
kubectl label node worker-node01 disk=hdd env=prod
kubectl label node worker-node02 disk=ssd env=staging
kubectl label node worker-node03 disk=ssd env=prod

# Vérification
kubectl get nodes --show-labels

```

</details>

---

## Partie 2 : Taints et Tolerations

Les **Taints** repoussent les Pods. Elles sont utilisées pour réserver des nœuds (ex: maintenance, hardware spécifique, clients dédiés).

1. **Appliquer un Taint**
Taintez le `worker-node01` pour qu'il n'accepte aucun Pod, sauf ceux qui ont une tolérance spécifique "BlueTeam".
* Clé: `team`, Valeur: `blue`, Effet: `NoSchedule`



<details>
<summary>Correction Commande </summary>

```bash
kubectl taint nodes worker-node01 team=blue:NoSchedule

```

</details>

2. **Test de planification (Échec attendu)**
Créez un pod simple `nginx-standard`. Vérifiez son état. Où est-il planifié ?
* *Note : Comme vous avez d'autres nœuds disponibles (02 et 03), il devrait aller ailleurs. S'il n'y avait que le node01, il resterait "Pending".*


3. **Utilisation de la Tolérance**
Créez un manifest `pod-tolerant.yaml`.
* Image: `nginx`
* Il doit **exiger** d'aller sur `worker-node01` (via `nodeSelector` ou `nodeAffinity` sur le label `disk=hdd` que nous avons mis partie 1).
* Il doit avoir la **Toleration** pour le taint `team=blue`.



<details>
<summary>Correction YAML </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-tolerant
spec:
  containers:
  - name: nginx
    image: nginx
  # 1. On force le pod à aller vers le noeud tainté via ses labels
  nodeSelector:
    disk: hdd
  # 2. On lui donne la clé pour passer la barrière du Taint
  tolerations:
  - key: "team"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"

```

```bash
kubectl apply -f pod-tolerant.yaml
kubectl get pod -o wide
# Le pod doit être Running sur worker-node01

```

</details>

---

## Partie 3 : Node Affinity (Attraction)

Contrairement au `nodeSelector` (qui est strict), l'Affinity offre plus de souplesse et de logique.

1. Créez un manifest `pod-ssd.yaml`.
2. Configurez une **Node Affinity** de type `requiredDuringSchedulingIgnoredDuringExecution`.
3. Le Pod doit être planifié uniquement sur les nœuds ayant le label `disk=ssd`.

<details>
<summary>Correction YAML </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ssd
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd

```

```bash
kubectl apply -f pod-ssd.yaml
kubectl get pod -o wide
# Le pod doit être sur worker-node02 ou worker-node03

```

</details>

---

## Partie 4 : Pod Anti-Affinity (Dispersion)

Pour la Haute Disponibilité (HA), on ne veut pas que tous les réplicas d'une application soient sur le même nœud (si le nœud tombe, tout tombe).

1. Créez un Deployment `web-ha.yaml`.
2. Replicas : `3`
3. Labels du template : `app: web-store`
4. Configurez une **Pod Anti-Affinity** :
* On préfère ne pas (`preferred...`) ou on interdit (`required...`) d'être sur le même nœud que des pods ayant le label `app: web-store`.
* TopologyKey : `kubernetes.io/hostname` (signifie "par nœud physique").



<details>
<summary>Correction YAML </summary>

Utilisons `required` pour forcer la dispersion stricte (1 pod par nœud max).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-ha
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-store
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx

```

```bash
kubectl apply -f web-ha.yaml
kubectl get pods -o wide -l app=web-store

```

**Observation :** Vous devriez voir un pod sur `node01` (s'il a la tolérance, sinon il restera Pending), un sur `node02` et un sur `node03`. Si vous n'avez pas ajouté la tolérance au Deployment, l'un des pods restera en `Pending` car `node01` est tainté et les deux autres nœuds sont déjà occupés par un pod (règle anti-affinity).

</details>