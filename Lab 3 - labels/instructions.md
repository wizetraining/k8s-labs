# LAB 3 - Création et Manipulation des Labels et Selectors

Ce TP fait suite au Lab 3. Nous allons utiliser le même cluster.

Dans une architecture microservices, le nombre de pods peut exploser (centaines de répliques, versions canary vs stable, backend vs frontend) . Sans un système de classement efficace, l'administration devient un cauchemar.

**Les Labels** sont des paires clé/valeur attachées aux objets (comme les Pods) pour les organiser.
**Les Selectors** permettent de filtrer ces objets pour effectuer des opérations ciblées.

## Partie 1 : Ajout de Labels (Namespaces et Pods)

Commencez par créer un dossier pour ce labo si nécessaire.

**1. Labelliser un Namespace**
Créez un namespace nommé `dev`. Ensuite, sans le recréer, ajoutez-lui le label `formation=k8s` de manière impérative.

<details><summary>Correction</summary>

```bash
# Création du namespace
kubectl create ns dev

# Ajout du label
kubectl label namespaces dev formation=k8s

# Vérification
kubectl get namespaces --show-labels
```


</details>

**2. Création de Pods avec Labels**
Créez un fichier YAML nommé `nginx-labels.yaml` pour déployer un pod avec les caractéristiques suivantes :

  * Nom: `nginx1`
  * Namespace: `dev`
  * Image: `nginx`
  * Labels: `app: frontend`

(Si vous rencontrez des limitations, utilisez l'image `public.ecr.aws/w1t1o1o1/nginx:latest`)

<details><summary>Correction</summary>

Fichier `nginx-labels.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
  namespace: dev
  labels:
    app: frontend
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

Appliquez le fichier :

```bash
kubectl apply -f nginx-labels.yaml
```


</details>

**3. Modifier des Labels existants**
Nous avons oublié la version \! Ajoutez le label `rel=beta` au pod `nginx1` existant via la ligne de commande.
Ensuite, créez un second pod nommé `nginx2` (impérativement ou via YAML) dans le même namespace avec les labels : `app=frontend` et `rel=stable`.

<details><summary>Correction</summary>

```bash
# Ajout du label manquant sur nginx1
kubectl label -n dev pod nginx1 rel=beta

# Création de nginx2 (méthode rapide)
kubectl run nginx2 --image=nginx -n dev --labels="app=frontend,rel=stable"
```


</details>

## Partie 2 : Filtres et Selectors

Maintenant que nous avons des pods étiquetés, apprenons à les filtrer.

**1. Affichage des Labels**
Listez les pods du namespace `dev`.

  * Affichez tous les labels.
  * Affichez uniquement les colonnes pour les labels `app` et `rel`.

<details><summary>Correction</summary>

```bash
# Afficher tous les labels
kubectl get po -n dev --show-labels

# Afficher des labels spécifiques en colonnes
kubectl get po -n dev -L app,rel
```


</details>

**2. Filtrage (Selectors)**
Effectuez les recherches suivantes dans le namespace `dev` :

  * Listez uniquement les pods en version `stable` (`rel=stable`).
  * Listez tous les pods qui *possèdent* le label `rel` (peu importe la valeur).
  * Listez tous les pods qui *ne possèdent pas* le label `rel`.

<details><summary>Correction</summary>

```bash
# Filtrer par valeur exacte
kubectl get po -n dev -l rel=stable

# Filtrer par présence de la clé
kubectl get po -n dev -l rel

# Filtrer par absence de la clé (Note : il faut des quotes pour le point d'exclamation)
kubectl get po -n dev -l '!rel'
```


</details>

## Partie 3 : Ordonnancement avec NodeSelector

Les labels ne servent pas qu'à filtrer, ils servent aussi à décider **où** un pod doit tourner (par exemple, uniquement sur des serveurs avec disque SSD).

**1. Labelliser un Nœud**
Listez vos nœuds. Choisissez un worker (ex: `worker-1`) et ajoutez-lui le label `disk=ssd`. Vérifiez ensuite que le label est bien appliqué.

<details><summary>Correction</summary>

```bash
# Lister les noeuds pour avoir le nom exact
kubectl get nodes

# Appliquer le label (remplacez <nom-du-worker> par le vrai nom)
kubectl label node <nom-du-worker> disk=ssd

# Vérifier avec un selector
kubectl get nodes -l disk=ssd
```


</details>

**2. Contraindre un Pod**
Créez un pod nommé `pod-ssd` dans le namespace `dev` (image `nginx`) qui **doit** obligatoirement tourner sur un nœud ayant le label `disk=ssd`. Utilisez le champ `nodeSelector`.

<details><summary>Correction</summary>

Fichier `pod-ssd.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ssd
  namespace: dev
  labels:
    app: backend
    rel: beta
spec:
  nodeSelector:       # C'est ici que la magie opère
    disk: "ssd"
  containers:
  - name: nginx
    image: nginx
```

Appliquez et vérifiez sur quel noeud il tourne :

```bash
kubectl apply -f pod-ssd.yaml
kubectl get pod pod-ssd -n dev -o wide
```


</details>

## Partie 4 : Annotations

Contrairement aux labels, les annotations ne servent pas à sélectionner des pods, mais à ajouter des métadonnées (description, contact, configuration technique, etc.).

Ajoutez l'annotation `wizetraining.com/equipe-responsable="les K8S dev"` au pod `pod-ssd`. Vérifiez ensuite en inspectant le pod.

<details><summary>Correction</summary>

```bash
# Annoter le pod
kubectl annotate pod -n dev pod-ssd wizetraining.com/equipe-responsable="les K8S dev"

# Vérifier (Section Annotations)
kubectl describe pod -n dev pod-ssd
```

</details>

## Partie 5 : Suppression via Selectors

Les selectors sont très puissants pour le nettoyage.

**1. Suppression par Nom**
Supprimez le pod `pod-ssd` en utilisant son nom.

<details><summary>Correction</summary>

```bash
kubectl delete pod -n dev pod-ssd
```

</details>

**2. Suppression de masse par Label**
Supprimez tous les pods restants qui ont le label `app=frontend` en une seule commande.

<details><summary>Correction</summary>

```bash
kubectl delete po -n dev -l app=frontend
```

</details>