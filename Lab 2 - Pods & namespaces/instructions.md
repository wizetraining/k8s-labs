# LAB 2 - Manipulation des Ressources Pods

Nous allons travailler sur le cluster Kubernetes déployé (1 Master + 3 Workers).

Dans ce Lab, nous allons manipuler l'atome de Kubernetes : le **Pod**.
C'est la plus petite unité déployable. Contrairement à Docker où l'on manipule des conteneurs, dans Kubernetes, on manipule des Pods qui *contiennent* des conteneurs.

## Partie 1 : Exploration du Cluster

Avant de créer des ressources, inspectons l'état actuel de notre infrastructure.

**1. État des Nœuds**
Listez les nœuds de votre cluster.
*Combien de masters avez-vous et combien de worker nodes ?*

<details><summary>Correction</summary>

```bash
kubectl get nodes
```

*Vous devriez voir 1 Master (Control Plane) et 3 Workers.*

</details>

**2. Les Namespaces**
Kubernetes permet de cloisonner virtuellement un cluster physique en plusieurs clusters virtuels appelés "Namespaces".
*Combien de namespaces avez-vous par défaut ? Listez-les.*

<details><summary>Correction</summary>

```bash
kubectl get namespaces
```

*Vous verrez `default`, `kube-system` (pour les composants internes), `kube-public`, etc.*

</details>

**3. Les Pods existants**
Listez les Pods de **tous** les namespaces pour voir ce qui tourne actuellement (notamment les composants systèmes comme CoreDNS, le Dashboard, etc.).

<details><summary>Correction</summary>

```bash
kubectl get pods -A
# ou la version longue :
kubectl get pods --all-namespaces
```

</details>

-----

## Partie 2 : Opérations sur les Pods (Approche Impérative)

L'approche **impérative** consiste à donner des ordres directs via la ligne de commande (`run`, `create`, `delete`). C'est idéal pour des tests rapides.

**1. Lancer un serveur Web**
Lancez un Pod nommé `webserver` utilisant l'image `nginx:latest`. (Si vous rencontrez des limitations, utilisez l'image `public.ecr.aws/wizetraining/nginx:latest`)
Une fois créé, trouvez les informations suivantes :

  * Sur quel nœud a-t-il été déployé ?
  * Dans quel namespace ?
  * Quel est son état ?

<details><summary>Correction</summary>

```bash
# Création du pod
kubectl run webserver --image nginx

# Visualisation avec détails (-o wide affiche l'IP et le Node)
kubectl get pods -o wide

# Pour voir tous les détails (events, erreurs, config)
kubectl describe pod webserver
```

*Il a été déployé dans le namespace `default` car aucun autre n'a été spécifié.*

</details>

**2. Isolation via Namespace**
Créez un namespace nommé `k8s-lab`.
Ensuite, lancez un Pod nommé `tools` utilisant l'image `busybox` dans ce namespace. Ce pod devra exécuter la commande `sleep 1000` pour rester en vie.
(Si vous rencontrez des limitations, utilisez l'image `public.ecr.aws/wizetraining/busybox:latest`)

<details><summary>Correction</summary>

```bash
# 1. Création du namespace
kubectl create ns k8s-lab

# 2. Création du pod dans ce namespace
# Notez le "--" qui sépare les arguments kubectl de la commande du conteneur
kubectl run tools --image busybox -n k8s-lab -- sleep 1000
```

</details>

**3. Test d'unicité**
Tentez de créer un *second* pod nommé `tools` (avec n'importe quelle image) dans le même namespace `k8s-lab`.
*Que se passe-t-il ? Pourquoi ?*

<details><summary>Correction</summary>

La commande échouera : `Error from server (AlreadyExists): pods "tools" already exists`.
**Pourquoi ?** Dans un Namespace, le nom d'une ressource doit être unique (comme un nom de domaine DNS). Vous pouvez avoir un pod `tools` dans le namespace `default` et un autre dans `k8s-lab`, mais pas deux dans le même.

</details>

**4. Communication Inter-Pods**
Nous allons tester la communication réseau.

1.  Récupérez l'adresse IP du pod `webserver` (dans le namespace `default`).
2.  Entrez dans le pod `tools` (dans le namespace `k8s-lab`) et essayez de "pinger" le webserver.

*Question : Est-ce que le fait d'être dans des namespaces différents bloque le réseau ?*

<details><summary>Correction</summary>

**Récupérer l'IP du webserver :**

```bash
kubectl get pod webserver -o wide
# Notons l'IP, par exemple : 192.168.118.29
```

**Entrer dans le pod tools et pinger :**

```bash
kubectl exec -it -n k8s-lab tools -- sh

# Une fois dans le conteneur :
/ # ping 192.168.118.29
```

**Résultat :** Le ping fonctionne \!
**Explication :** Par défaut, Kubernetes est un "plat de spaghettis" réseau. Tous les pods peuvent communiquer avec tous les pods, peu importe leur Namespace ou le Nœud sur lequel ils se trouvent. Les Namespaces sont une isolation administrative (organisation), pas réseau (sauf si on ajoute des NetworkPolicies).

</details>

**5. Localisation**
Comment vérifier rapidement sur quel serveur physique (Worker node) tourne le pod `tools` ?

<details><summary>Correction</summary>

```bash
# Option 1 : Output wide
kubectl get pods -n k8s-lab -o wide

# Option 2 : Describe
kubectl describe pod tools -n k8s-lab | grep Node:
```

</details>

-----

## Partie 3 : Approche Déclarative (YAML)

C'est la méthode recommandée pour la production ("Infrastructure as Code"). On décrit l'état désiré dans un fichier, et on demande à K8s de l'appliquer.

**Objectif :** Lancer un Pod nommé `tools2`, image `busybox`, commande `sleep 1500` dans le namespace `k8s-lab`.

**1. Générer le YAML**
Au lieu d'écrire le YAML à la main, une astuce de pro est d'utiliser `kubectl` pour le générer pour nous avec l'option `--dry-run=client -o yaml`.

Générez le fichier `pod-tools2.yml` sans créer le pod.

<details><summary>Correction</summary>

```bash
kubectl run tools2 --image busybox -n k8s-lab --dry-run=client -o yaml -- sleep 1500 > pod-tools2.yml
```

</details>

**2. Analyser et Appliquer**
Regardez le contenu du fichier généré :

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: tools2
  name: tools2
  namespace: k8s-lab
spec:
  containers:
  - args:
    - sleep
    - "1500"
    image: busybox
    name: tools2
    resources: {}
```

Appliquez ce fichier pour créer la ressource :

<details><summary>Correction</summary>

```bash
kubectl apply -f pod-tools2.yml
```

</details>

**3. Vérification finale**
Vérifiez que vos deux outils tournent bien dans le namespace dédié.

<details><summary>Correction</summary>

```bash
kubectl get pods -n k8s-lab
```

*Sortie attendue :*

```text
NAME     READY   STATUS    RESTARTS   AGE
tools    1/1     Running   0          10m
tools2   1/1     Running   0          25s
```

</details>

Félicitations \! Vous savez manipuler des Pods de manière impérative et déclarative, et vous avez vérifié la connectivité réseau inter-namespace.