# LAB 01 - Exploration des Fondements de Kubernetes et de son API

Ce lab a pour objectif de : 
* Vous familiariser avec l'outil principal d'administration `kubectl`
* D'explorer les composants internes du cluster
* De comprendre comment l'API Kubernetes est structurée, et enfin d'apprendre à l'étendre selon vos propres besoins.

## Partie 1 : L'Outil Client et les Contextes (`kubectl`)

**Rappel :**
`kubectl` est l'outil en ligne de commande fourni par Kubernetes pour communiquer avec le plan de contrôle (API Server). Il se réfère par défaut à un fichier nommé `config` situé dans `$HOME/.kube/config`. Ce fichier contient les accès aux clusters, les identifiants des utilisateurs, et les contextes (la liaison entre un utilisateur, un cluster et un namespace).

> **Astuce :** Vous pouvez spécifier d'autres fichiers de configuration en définissant la variable d'environnement `$KUBECONFIG` ou en utilisant l'option `--kubeconfig`.

**1. Lister et observer les contextes**
Lister les contextes de votre environnement Kubernetes. Combien de contextes avez-vous et quel est celui par défaut ?

<details><summary>Correction</summary>

```bash
kubectl config get-contexts
```
*L'étoile (`*`) indique le contexte actuellement utilisé.*
</details>

**2. Visualiser la configuration**
Comment est structurée cette configuration ? Affichez la version complète, puis la version courte (minifiée).

<details><summary>Correction</summary>

```bash
kubectl config view 
```
On reconnaît la structure générale des fichiers YAML de Kubernetes :
* `apiVersion: v1`
* `kind: Config`
* `clusters` : Liste des clusters et les IP/URL des API-Servers.
* `users` : Liste des utilisateurs avec leurs clés d'authentification.
* `contexts` : Fait le lien entre un user, un cluster, et optionnellement un namespace.

Pour voir uniquement la configuration du contexte actuel (version courte) :
```bash
 kubectl config view --minify
```
</details>

**3. Changer de namespace par défaut**
Dans votre organisation, vous travaillez sur le namespace `dev`. Pour vous faciliter la tâche et éviter de taper `-n dev` à chaque commande, modifiez votre contexte actuel pour utiliser `dev` par défaut.

<details><summary>Correction</summary>

```bash
kubectl config set-context --current --namespace=dev
kubectl config view --minify
```
</details>

---

## Partie 2 : Exploration du Control Plane (Architecture)

Maintenant que nous savons utiliser notre client, regardons ce qui tourne "sous le capot" de Kubernetes. Les composants essentiels (le Control Plane) tournent généralement sous forme de Pods dans un namespace dédié.

**1. Identifier les composants de base**
Lister les Pods contenus dans le namespace `kube-system`.

<details><summary>Correction</summary>

```bash
# On force le namespace kube-system car notre contexte est maintenant sur "dev"
kubectl get pods -n kube-system 
```
</details>

Quels sont les composants que vous avez identifiés et quels sont leurs rôles ?

<details><summary>Correction</summary>

* **`kube-apiserver` :** Le point d'entrée unique du cluster. Tous les autres composants communiquent avec lui.
* **`etcd` :** La base de données clé-valeur, cohérente et hautement disponible, stockant l'intégralité de l'état du cluster.
* **`kube-controller-manager` :** Boucle de contrôle qui surveille l'état actuel et tente de converger vers l'état désiré.
* **`kube-scheduler` :** Détermine sur quel nœud un nouveau Pod doit être placé en fonction des ressources disponibles et des contraintes.
* **`coredns` :** Serveur DNS interne permettant la découverte de services au sein du cluster.
* **`kube-proxy` :** Tourne sur chaque nœud pour gérer les règles réseaux iptables/IPVS nécessaires aux Services Kubernetes.
* **`cilium` (ou autre CNI) :** Le plugin réseau permettant aux Pods de communiquer entre eux à travers les différents nœuds.
</details>

**2. Inspecter la configuration d'un composant**
À l'aide de la commande `kubectl describe pods -n kube-system <NOM_DU_POD>`, visualisez les options de lancement de chacun des Pods du Control Place : etcd, kube-scheduler, api-server...

<details><summary>Correction</summary>

```bash
# Remplacer <NOM_DU_POD> par le nom exact affiché précédemment
kubectl describe pods -n kube-system etcd-master-node
```
*Dans la section `Command:`, vous verrez tous les arguments passés au binaire etcd (ex: `--data-dir`, `--cert-file`, etc.).*
</details>

**3. Inspectez la configuration de l'Agent Kubelet**

Placez-vous sur le serveur master et visualisez les options avec lesquelles Kubelet est en cours d'exécution

_**Note:** Si vous explorez avec `Minikube` étant données que tout cela tourne à l'intérieur d'un noeud virtual, pour y accéder, exécutez la commande suivante_

```bash
minikube ssh 
```

<details><summary>Correction</summary>

```bash
ps aux | grep -i kubelet
```
ou 
```bash
systemctl status kubelet
```

</details>

---

## Partie 3 : L'API Kubernetes (Groupes et Ressources)



L'API Kubernetes est RESTful et fortement structurée. Elle est divisée en groupes d'API pour faciliter son extension.
* **Le groupe Core (legacy) :** Se trouve dans le chemin REST `/api/v1` (ex: Pods, Services, ConfigMaps).
* **Les groupes nommés (named groups) :** Se trouvent dans le chemin REST `/apis/$GROUP_NAME/$VERSION` (ex: `apps/v1` pour les Deployments).

**1. Explorer les APIs disponibles**
Listez les groupes d'API Kubernetes, puis listez toutes les ressources disponibles.

<details><summary>Correction</summary>

```bash
# Lister les groupes
kubectl api-versions

# Lister les ressources et leurs groupes
kubectl api-resources
```
</details>

**2. Trouver l'apiVersion d'une ressource**
Quels sont les `apiVersion` des ressources `StorageClass` et `RoleBinding` ?

<details><summary>Correction</summary>

Elles sont respectivement `storage.k8s.io/v1` et `rbac.authorization.k8s.io/v1`.

```bash
kubectl api-resources | grep -i storageclass
kubectl api-resources | grep -i rolebinding
```
</details>

**3. Interroger l'API en direct (REST)**
Affichez les informations du Cluster pour identifier l'IP de l'API Server. Faites ensuite une requête `curl` basique : `curl -k https://<IP_API_Server>:6443`. Quel est le résultat et pourquoi ?

<details><summary>Correction</summary>

```bash
kubectl cluster-info 
curl -k https://192.168.56.10:6443 # Remplacer par l'IP de votre API
```
**Résultat :** Un message `403 Forbidden` (`"User system:anonymous cannot get path /"`).
**Pourquoi ?** L'API Server est sécurisé et exige une authentification (certificat, token, etc.).
</details>

**4. Contourner l'authentification avec un Proxy**
Démarrez un Proxy de l'API Serveur avec kubectl, puis refaites un `curl` sur le port local pour lire la version du cluster.

<details><summary>Correction</summary>

```bash
# Lance le proxy en tâche de fond
kubectl proxy --port=8064 &
```

```bash
# Accès autorisé sans certificat car kubectl gère l'authentification en arrière-plan
curl http://localhost:8064/version

# Lister les pods du kube-system via l'API REST brute
curl http://localhost:8064/api/v1/namespaces/kube-system/pods
```
</details>

---

## Partie 4 : Étendre l'API avec les CRD (Custom Resource Definitions)

Kubernetes est conçu pour être extensible. Si les ressources natives (Pods, Deployments...) ne suffisent pas, vous pouvez créer vos propres objets grâce aux **CRD (Custom Resource Definitions)**. L'API Server les traitera exactement comme des objets natifs.

**Scénario :** Nous développons une plateforme de formation en ligne. Nous voulons que Kubernetes gère un nouvel objet appelé `Course` (Cours).
* **Groupe nommé :** `stable.k8s.wizetraining.io`
* **Version :** `v1`
* **Portée (Scope) :** Namespaced

**1. Déclarer la CRD à l'API Server**

* Avant de déclarer la CRD, faites `kubectl get course`; quel message d'erreur obtenez vous ?

* Créez un fichier `course-crd.yaml` pour définir la structure de notre nouvel objet (son "schéma").

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # Le nom doit être au format pluriel.groupe
  name: courses.stable.k8s.wizetraining.io
spec:
  group: stable.k8s.wizetraining.io
  versions:
    - name: v1
      served: true
      storage: true
      # Définition de la structure de notre objet "Course"
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                title:
                  type: string
                durationHours:
                  type: integer
  scope: Namespaced
  names:
    plural: courses
    singular: course
    kind: Course
    shortNames:
    - crs
```

* Appliquez le fichier et vérifiez que l'API l'a bien pris en compte :

<details><summary>Correction CRD</summary>

```bash
kubectl apply -f course-crd.yaml
kubectl api-resources | grep course
```
</details>

**2. Créer, Mettre à jour et Supprimer notre ressource personnalisée**
Maintenant que Kubernetes "connaît" le type `Course`, créez un fichier `my-course.yaml` pour instancier un cours sur Kubernetes. Appliquez-le, récupérez-le, puis mettez-le à jour.

<details><summary>Correction YAML et Commandes</summary>

**Création du fichier `my-course.yaml` :**
```yaml
apiVersion: stable.k8s.wizetraining.io/v1
kind: Course
metadata:
  name: kubernetes-fundamentals
spec:
  title: "Maîtriser les Fondamentaux de Kubernetes"
  durationHours: 3
```

**Manipulation avec `kubectl` :**
```bash
# 1. Création
kubectl apply -f my-course.yaml

# 2. Récupération (Notez qu'on peut utiliser le shortName "crs")
kubectl get courses
kubectl get crs kubernetes-fundamentals -o yaml

# 3. Mise à jour (Imperative pour tester)
# Modifions la durée ou le titre à la volée
kubectl patch course kubernetes-fundamentals --type='merge' -p '{"spec":{"durationHours": 5}}'

# Vérification
kubectl get crs kubernetes-fundamentals -o jsonpath='{.spec.durationHours}'

# 4. Suppression
kubectl delete course kubernetes-fundamentals
```
</details>