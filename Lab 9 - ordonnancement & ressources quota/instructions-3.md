# LAB 9.3 - Gestion des Ressources et Quotas (QoS & Limits)

## Partie 1 : Requests, Limits et QoS (Niveau Pod)

Dans Kubernetes, la gestion des ressources repose sur deux concepts clés définis par conteneur :

  * **Requests (Demandes)** : Ce que le conteneur *garantit* d'avoir. Le Scheduler utilise cette valeur pour placer le Pod sur un nœud.
  * **Limits (Limites)** : Le plafond absolu. Si le CPU dépasse, il est bridé (throttled). Si la RAM dépasse, le Pod est tué (OOMKilled).

L'association de ces valeurs détermine la classe de qualité de service (**QoS**) :

1.  **Guaranteed** : Requests = Limits (pour CPU et RAM).
2.  **Burstable** : Requests < Limits.
3.  **BestEffort** : Aucune request ni limit définie.

### Exercice 1.1 : L'illusion des ressources

Créons un Pod avec des limites strictes pour voir comment il perçoit son environnement.

<details><summary>Afficher le manifeste limited-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      limits:
        cpu: 1
        memory: 20Mi
```

</details>

Créez le pod et, une fois qu'il est *Running*, exécutez la commande `top` à l'intérieur du conteneur.

```bash
kubectl apply -f limited-pod.yaml
kubectl exec -it limited-pod -- top
```

**Observation :**
Vous remarquerez que la mémoire et le CPU affichés ne sont pas ceux de la limite (20Mi / 1 CPU), mais ceux du **Node** (l'hôte physique/virtuel).
*Pourquoi est-ce dangereux ?* Les applications Java ou Node.js anciennes peuvent tenter de consommer toute la RAM visible du Node et se faire tuer par Kubernetes instantanément.

### Exercice 1.2 : Requests et QoS

Créez maintenant un Pod qui définit uniquement des *requests* (réservations).

<details><summary>Afficher le manifeste requested-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: requested-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      requests:
        cpu: 0.5
        memory: 10Mi
```

</details>

Déployez-le et analysez sa QoS :

```bash
kubectl apply -f requested-pod.yaml
kubectl get pod requested-pod -o jsonpath='{.status.qosClass}'
```

**Questions :**

1.  Quelle est la QoS du `limited-pod` ? (Réponse : *Burstable*, car il a des limites mais pas de requests explicites égales aux limites, ou *Guaranteed* si K8s applique la request=limit implicitement. Dans ce cas précis, k8s applique `requests: {cpu: 1, memory: 20Mi}` par défaut si seul limits est mis, donc c'est **Guaranteed**).
2.  Quelle est la QoS du `requested-pod` ? (Réponse : *Burstable*).
3.  Quelle est la QoS d'un pod sans rien ? (Réponse : *BestEffort* - ce sont les premiers à être tués si le nœud sature).

-----

## Partie 2 : LimitRanges (Valeurs par défaut et contraintes)

Les développeurs oublient souvent de mettre des limites. L'administrateur utilise les **LimitRanges** pour :

1.  Injecter des valeurs par défaut.
2.  Imposer des bornes Min/Max par conteneur.

Préparez un namespace de test :

```bash
kubectl create namespace limited-namespace
```

### Exercice 2.1 : Injection de valeurs par défaut

Créez une `LimitRange` qui force une mémoire par défaut à 512Mi si l'utilisateur ne précise rien.

<details><summary>Correction (LimitRange)</summary>

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limit-range
  namespace: limited-namespace
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 1
    defaultRequest:
      memory: 256Mi
      cpu: 0.1
    type: Container
```

</details>

**Test de l'injection :**

1.  Lancez un pod `pod-no-limits` (image busybox) sans aucune section `resources` dans ce namespace.
2.  Vérifiez si les limites ont été injectées :

<!-- end list -->

```bash
kubectl get pod pod-no-limits -n limited-namespace -o yaml | grep resources -A 10
```

Vous constaterez que le Pod a reçu automatiquement les configurations définies dans `default` et `defaultRequest`.

### Exercice 2.2 : Blocage par Min/Max

Nous voulons empêcher la création de "gros" conteneurs. Définissez une limite maximale de 1Gi de RAM et 900m CPU.

<details><summary>Correction (Max LimitRange)</summary>

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: max-limit-range
  namespace: limited-namespace
spec:
  limits:
  - max:
      cpu: "900m"
      memory: 1Gi
    min:
      cpu: "1m"
      memory: 10Mi
    type: Container
```

</details>

Essayez maintenant de créer un Pod qui demande 2 CPU (plus que le max autorisé de 900m) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-too-big
  namespace: limited-namespace
spec:
  containers:
  - image: busybox
    name: main
    resources:
      requests:
        cpu: 2
```

**Résultat :** L'API Server rejette immédiatement la requête : `Forbidden: maximum cpu usage per Container is 900m, but request is 2`.

*Nettoyage avant la suite :*

```bash
kubectl delete namespace limited-namespace
```

-----

## Partie 3 : ResourceQuotas (Budget global)

Les `LimitRange` agissent au niveau du pod/conteneur. Les **ResourceQuotas** agissent au niveau du **Namespace** (le "portefeuille" de l'équipe).

Créez le namespace `quotas-namespace`.

### Exercice 3.1 : Quota de calcul (Compute)

Appliquez un quota strict : l'ensemble des pods du namespace ne doit pas dépasser 2 CPU et 2Gi de RAM au total (limites).

<details><summary>Correction (ResourceQuota Compute)</summary>

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-compute
  namespace: quotas-namespace
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

</details>

Vérifiez l'état du quota (actuellement 0 utilisé) :

```bash
kubectl get resourcequota -n quotas-namespace
```

**Scénario de saturation :**
Déployez un `Deployment` Nginx avec 2 replicas. Chaque replica demande 500m CPU et 128Mi RAM.

<details><summary>Correction</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: quotas-namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            memory: "128Mi"
            cpu: "500m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```

</details>

```bash
kubectl apply -f nginx-deployment.yaml
```

Puis, passez à 5 replicas :

```bash
kubectl scale deployment nginx --replicas=5 -n quotas-namespace
```

**Observation :**
Regardez l'état des Pods et les événements.

```bash
kubectl get pods -n quotas-namespace
kubectl describe rs <nom-du-replicaset> -n quotas-namespace
```

Seuls certains pods démarrent. Les autres sont "Forbidden" ou restent en Pending car le quota global du namespace (2 CPU max) serait dépassé.

### Exercice 3.2 : Quota d'objets (Count)

On peut aussi limiter le nombre d'objets (pour éviter qu'un script ne crée 10 000 configmaps par erreur).
Interdisez la création de tout Deployment (`count/deployments.apps: "0"`) et limitez les Services à 2.

<details><summary>Correction (ResourceQuota Count)</summary>

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-count
  namespace: quotas-namespace
spec:
  hard:
    services: "2"
    count/deployments.apps: "0"
    pods: "3"
```

</details>

Si vous tentez de créer un déploiement maintenant, l'action sera refusée, même s'il vous reste du CPU/RAM disponible.

## Partie 4 : Scénario Bonus - OOMKilled

Ce scénario illustre ce qui se passe quand une application dépasse sa propre limite (et non le quota du namespace).

1.  Créez un Pod "leaky-app" avec une limite de mémoire très faible (5Mi).
2.  L'application va tenter d'écrire en mémoire.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "50Mi"
      requests:
        memory: "20Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
```

**Instruction :**
Appliquez ce pod. Il demande à `stress` d'allouer 100Mo de RAM, mais Kubernetes le limite à 50Mo.

**Observation :**

```bash
kubectl get pod oom-test --watch
```

Vous verrez le statut passer de `Running` à `OOMKilled` (Out Of Memory), puis `CrashLoopBackOff`.
Contrairement au CPU qui est juste ralenti (throttled), la RAM est une ressource incompressible : dépassement = arrêt brutal.

### Nettoyage

```bash
kubectl delete namespace quotas-namespace
```