# LAB 9.2 - Ordonnancement & Santé des Pods (Suite)

Cet atelier vous guidera à travers la configuration des sondes (Probes) Kubernetes. Ces mécanismes sont essentiels pour créer des applications "auto-réparatrices" (Self-Healing) et garantir que le trafic n'est envoyé qu'aux conteneurs capables de le traiter.

**Objectifs**:

* Configurer une **Liveness Probe** pour redémarrer automatiquement une application plantée.
* Configurer une **Readiness Probe** pour gérer le flux de trafic vers les Pods.
* Configurer une **Startup Probe** pour gérer les applications au démarrage lent.

---

## Partie 1 : Liveness Probe (L'auto-réparation)

La sonde **Liveness** répond à la question : *"Mon application est-elle plantée ?"*. Si la réponse est oui, Kubernetes tue le conteneur et en recrée un nouveau.

**Scénario :** Nous allons simuler une application qui fonctionne pendant 30 secondes, puis rencontre une erreur critique (simulée par la suppression d'un fichier).

1. Créez le manifeste `liveness-pod.yaml` avec les spécifications suivantes :
* **Image** : `busybox`
* **Commande** : Crée le fichier `/tmp/healthy`, dort 30s, supprime le fichier, puis dort 600s.
* **Liveness Probe** :
* Type : Commande (`cat /tmp/healthy`)
* Délai initial : 5 secondes
* Période : 5 secondes





<details>
<summary>Correction YAML </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5

```

```bash
kubectl apply -f liveness-pod.yaml

```

</details>

2. **Observation** :
Surveillez le pod pendant une minute.
* Au début, le pod est `Running`.
* Après 30s, le fichier est supprimé.
* La sonde échoue. Kubernetes incrémente le compteur `RESTARTS`.



<details>
<summary>Commande de vérification </summary>

```bash
# Observez le changement en temps réel
kubectl get pod liveness-pod -w

# Ou regardez les événements pour voir l'erreur exacte
kubectl describe pod liveness-pod

```

*Cherchez la ligne "Liveness probe failed" dans les Events.*

</details>

---

## Partie 2 : Readiness Probe (Le contrôle du trafic)

La sonde **Readiness** répond à la question : *"Mon application est-elle prête à recevoir des clients ?"*. Si la réponse est non, le Pod reste vivant (Running), mais **on lui coupe le réseau** (il est retiré des Endpoints du Service).

**Scénario :** Un serveur web qui possède une page de maintenance. Si un fichier spécifique est absent, il ne doit pas recevoir de trafic.

1. Créez le manifeste `readiness-pod.yaml` :
* **Image** : `nginx`
* **Port** : 80
* **Readiness Probe** :
* Type : `exec` (vérifier si `/tmp/ready` existe)
* InitialDelay: 5s, Period: 5s





<details>
<summary>Correction YAML </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:
    app: my-web
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/ready
      initialDelaySeconds: 5
      periodSeconds: 5

```

```bash
kubectl apply -f readiness-pod.yaml

```

</details>

2. **Manipulation** :
* Le Pod va démarrer en état `Running` mais ne sera pas `Ready` (`0/1` dans la colonne READY) car le fichier `/tmp/ready` n'existe pas par défaut.
* Créez le fichier dans le conteneur pour simuler la fin du chargement de l'application.



<details>
<summary>Commandes </summary>

```bash
# Vérifiez l'état (devrait être 0/1)
kubectl get pod readiness-pod

# Créez le fichier témoin
kubectl exec readiness-pod -- touch /tmp/ready

# Vérifiez à nouveau (devrait passer à 1/1 après quelques secondes)
kubectl get pod readiness-pod -w

```

</details>

3. **Bonus (Impact sur les Services)** :
Si vous créez un Service ciblant ce Pod, et que vous supprimez le fichier `/tmp/ready`, le Pod restera "Running" mais l'IP du Pod disparaîtra de la liste des `Endpoints` du service. C'est idéal pour retirer un nœud d'un LoadBalancer sans le tuer (ex: pour maintenance ou debug).

---

## Partie 3 : Startup Probe (Les démarrages lents)

La sonde **Startup** est utilisée pour les applications Legacy qui mettent du temps à démarrer (ex: migration de base de données, chargement de cache Java...). Tant qu'elle n'a pas réussi, les autres sondes (Liveness/Readiness) sont désactivées.

**Problème à résoudre :** Si une app met 60s à démarrer et que la Liveness check toutes les 10s, l'app sera tuée avant d'avoir pu démarrer (boucle de redémarrage infinie).

**Scénario :** Une application simulant un démarrage lent (30s). Nous voulons une Liveness très réactive (5s) une fois l'app lancée, mais nous devons protéger le démarrage.

1. Créez le manifeste `startup-pod.yaml` :
* **Image** : `busybox`
* **Commande** : `sleep 30; touch /tmp/started; sleep 3600`
* **Liveness Probe** : Check `/tmp/started` toutes les 5s (Va échouer immédiatement sans Startup Probe).
* **Startup Probe** : Check `/tmp/started`. `failureThreshold: 30`, `periodSeconds: 2`. (Laisse 30*2 = 60s pour démarrer).



<details>
<summary>Correction YAML </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
  - name: slow-app
    image: busybox
    args:
    - /bin/sh
    - -c
    - "echo Démarrage...; sleep 30; touch /tmp/started; echo Prêt !; sleep 3600"
    
    # La Liveness est agressive (5s)
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/started
      initialDelaySeconds: 0
      periodSeconds: 5
      
    # La Startup protège la Liveness pendant le boot
    startupProbe:
      exec:
        command:
        - cat
        - /tmp/started
      failureThreshold: 30 # Essaie 30 fois
      periodSeconds: 2     # Toutes les 2 secondes (Total 60s max)

```

```bash
kubectl apply -f startup-pod.yaml

```

</details>

2. **Observation** :
Regardez les événements. Vous ne devriez **pas** voir de redémarrage.
* Pendant les 30 premières secondes, la Startup Probe échoue (normal), mais ne tue pas le conteneur.
* Dès que le fichier est créé, la Startup réussit une fois, puis laisse la main à la Liveness Probe.

---

### Résumé des sondes

| Sonde | Question posée | Action si échec | Cas d'usage |
| --- | --- | --- | --- |
| **Startup** | A-t-elle fini de booter ? | Tue et Redémarre | Appli lourde (Java, grosses DB) |
| **Readiness** | Peut-elle servir du trafic ? | Retire l'IP du Service | Surcharge, dépendance externe HS |
| **Liveness** | Est-elle toujours vivante ? | Tue et Redémarre | Deadlock, crash interne |