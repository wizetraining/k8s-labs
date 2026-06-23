# LAB 8 - Création d'utilisateur sur Kubernetes

## Partie 1 : Gestion des Authentifications

Kubernetes ne gère pas d'objets natifs de type "Utilisateur" et ne dispose pas de son propre registre d'identité interne.
Pour valider l'identité des requêtes, le cluster s'appuie sur des méthodes externes telles que :

  * Fichiers de tokens statiques
  * Jetons "Bearer"
  * Certificats X509
  * OpenID Connect (OIDC)
  * Etc.

Bien que l'intégration avec un fournisseur d'identité (IdP) via OIDC soit recommandée en production, nous utiliserons ici la méthode des certificats clients. Dans ce contexte, le champ "Common Name" (CN) du certificat sera interprété par Kubernetes comme le nom de l'utilisateur.

Le flux de travail se décompose ainsi :

1.  Générer une clé privée RSA.
2.  Créer une demande de signature de certificat (CSR) associée à cette clé.
3.  Transmettre cette CSR à l'API Kubernetes pour signature par l'autorité de certification (CA) du cluster.
4.  Récupérer le certificat final signé.
5.  Configurer le client `kubectl` avec ces nouveaux identifiants.
6.  Valider l'accès (l'authentification fonctionne, mais les actions sont refusées faute de droits).
7.  Attribuer les permissions via les RoleBindings.

### Création de l'utilisateur **devuser**

Commencez par générer une clé privée RSA (format PEM, 2048 bits) avec OpenSSL.

```bash
openssl genrsa -out devuser.pem
```

Générez ensuite la demande de signature (CSR) basée sur cette clé.

  * `CN` (Common Name) : Définit le nom de l'utilisateur.
  * `O` (Organization) : Définit le ou les groupes auxquels l'utilisateur appartient. Kubernetes utilise ces champs pour l'autorisation.

<!-- end list -->

```bash
openssl req -new -key devuser.pem -out devuser.csr -subj "/CN=devuser/O=dev/O=wizetraining"
```

Pour soumettre cette demande à Kubernetes, procédez comme suit :

  * Encodez le fichier CSR en base64 :

<!-- end list -->

```bash
cat devuser.csr | base64
```

  * Déclarez un objet Kubernetes de type `CertificateSigningRequest` en y collant le contenu encodé :

<!-- end list -->

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: devuser
spec:
  request: <##### coller le Contenu Base64#####>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 3600
  usages:
  - client auth
```

Appliquez la configuration et vérifiez l'état de la demande :

```bash
kubectl apply -f devuser-csr.yml

kubectl get csr

kubectl get csr devuser

kubectl describe csr devuser

(La condition est "Pending", l'approbation d'un administrateur est requise)
```

Approuver (signer) la demande de certificat :

```bash
kubectl certificate approve devuser

kubectl get csr devuser

(Condition : Approved,Issued => La demande est validée et le certificat est généré)
```

Récupérez le certificat signé depuis l'objet CSR et décodez-le pour créer le fichier `.crt` :

```bash
kubectl get csr devuser -o jsonpath='{.status.certificate}' | base64 -d > devuser.crt
```

<details><summary>Alternative (Uniquement pour cluster local)</summary>

Si vous avez un accès direct au nœud maître (Master), vous pouvez signer le certificat manuellement avec OpenSSL en utilisant la CA du cluster :

```bash
openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -days 10 -in devuser.csr -out devuser.crt
```

</details>

Il faut maintenant configurer un contexte `kubectl` pour cet utilisateur.
Les étapes consistent à définir le cluster, renseigner les identifiants et créer le contexte.

```bash
kubectl config set-cluster k8s-lab --insecure-skip-tls-verify=true --server=https://<IP_ADVERTISER_API_SERVER>:6443

kubectl config set-credentials devuser --client-certificate=devuser.crt --client-key=devuser.pem --embed-certs=true

kubectl config set-context default --cluster=k8s-lab --user=devuser

kubectl config use-context default

kubectl config view
```

Nettoyage : les fichiers sources ne sont plus nécessaires car les certificats sont désormais intégrés dans la configuration (`embed-certs=true`).

```bash
kubectl delete csr devuser
rm devuser.pem devuser.crt devuser.csr
```

Tentez de lister les Pods avec ce nouvel utilisateur.
Pourquoi cette commande échoue-t-elle ?

```bash
kubectl get pods
```

Le message d'erreur confirme que l'authentification a réussi (Kubernetes sait qui vous êtes), mais l'autorisation a échoué.

```error
Error from server (Forbidden): pods is forbidden: User "devuser" cannot list resource "pods" in API group "" at the cluster scope
```

L'utilisateur est reconnu, mais aucun droit ne lui a été attribué.

*(Nous allons corriger cela dans la partie suivante).*

Commande utile pour vérifier vos propres droits ou ceux d'un autre utilisateur (impersonation) :

```bash
kubectl auth can-i list pods --as devuser
```

### Notes

L'option `expirationSeconds` dans la CSR permet de définir la durée de vie du certificat.

Lors de la création de la configuration via `kubectl`, l'option `--embed-certs` intègre directement les données dans le fichier `config` :

  * **client-key-data** : La clé privée encodée en base64.
  * **client-certificate-data** : Le certificat signé encodé en base64.

**Important** : Il n'existe pas de mécanisme natif simple pour révoquer un certificat signé ; il reste valide jusqu'à son expiration. Cependant, pour bloquer un accès, vous pouvez supprimer les objets RBAC (RoleBinding, ClusterRoleBinding) associés à l'utilisateur. Il pourra toujours s'authentifier, mais ne pourra plus rien faire.

-----

## Partie 2 : Gestion des Autorisations (RBAC)

Quelle distinction faites-vous entre un `Role` et un `ClusterRole` ?

<details><summary>Réponse</summary>

  * **Role** : La portée est limitée à un Namespace spécifique (ex: default, dev, k8s-lab).
  * **ClusterRole** : La portée est globale à tout le cluster (transverse aux namespaces ou pour des ressources non-namespacées comme les Nodes).

</details>

**Exercice :** Pour le namespace `dev-project` (créez-le si nécessaire), configurez un accès en lecture seule sur toutes les ressources pour les membres du groupe `dev`. Créez le Role adéquat.

<details><summary>Correction</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-project
  name: readonly-for-dev
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

</details>

Associez ce Role au groupe via un `RoleBinding` et testez.

<details><summary>Correction</summary>

```bash
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-for-dev
  namespace: dev-project
subjects:
- kind: Group
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: readonly-for-dev
  apiGroup: rbac.authorization.k8s.io
```

</details>

Vérification des droits :

```bash
kubectl auth can-i list pods --namespace dev-project --as devuser
# Résultat attendu : no (Pourquoi ?)
```

<details><summary>Explications</summary>

Parce que Kubernetes n'a pas de mémoire pour comprendre que l'utilisateur devuser est dans le groupe dev
Par contre si vous switchez de context pour vous positionner en tant que devuser et vous réessayez la commande, vous obtenez YES. Car en temps réel Kubernetes vois que le certificat de devuser a bien 0=dev (donc autorisé)

</details>


```bash
kubectl config get-contexts
```

```bash
kubectl config use-context default
```

```bash
kubectl auth can-i list pods --namespace dev-project
# Résultat attendu : yes

kubectl auth can-i list pods --namespace default
# Résultat attendu : no

kubectl auth can-i create pods --namespace dev-project
# Résultat attendu : no
```

**Défi supplémentaire :** Créez un Role et un RoleBinding permettant à l'utilisateur `devuser` (et non au groupe) d'effectuer uniquement les actions suivantes :

  * Créer des ressources : Pods et Deployments (dans le Namespace `default`).
  * Lire des ressources : ConfigMaps et Services (dans le Namespace `default`).

-----

## Partie 3 : ImagePullSecret & ServiceAccount

L'objectif est de déployer l'application `Countvisit` en utilisant une image stockée sur votre registre privé Harbor ([https://harbor.wizetraining.com](https://harbor.wizetraining.com)).

1.  Téléchargez l'image `countvisit` localement.
2.  Créez un projet sur Harbor (donnez-lui un nom explicite ).
3.  Renommez (tag) l'image pour qu'elle pointe vers votre registre privé et poussez-la (push).

<details><summary>Correction</summary>

```bash
docker pull public.ecr.aws/wizetraining/webapp-count-secure:v1
docker login  https://harbor.wizetraining.com 
# Pré-requis : Avoir créé un projet (ex: "test") dans l'interface Harbor
docker tag  public.ecr.aws/wizetraining/webapp-count-secure:v1 harbor.wizetraining.com/test/webapp-count-secure:v1

docker push harbor.wizetraining.com/test/webapp-count-secure:v1
```

</details>

Ensuite, configurez l'accès Kubernetes au registre :

1.  Créez un Secret de type `docker-registry` contenant vos identifiants Harbor.
2.  Créez un `ServiceAccount` nommé `countvisit-sa`. Désactivez le montage automatique du token API et attachez-y le Secret créé précédemment.

<details><summary>Correction</summary>

```bash
DOCKER_USERNAME=labuser13
PASSWORD="Labuser13@"
REGISTRY_URL=harbor.wizetraining.com

kubectl create secret docker-registry registry-secret \
  --docker-username=$DOCKER_USERNAME \
  --docker-password=$PASSWORD \
  --docker-server=$REGISTRY_URL
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  name: countvisit-sa
automountServiceAccountToken: false
imagePullSecrets: 
- name: registry-secret
```

Mise à jour du manifeste de déploiement pour utiliser ce compte de service :

```yaml
(...)
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      serviceAccountName: countvisit-sa             <----------- Champ à ajouter
      containers:
      - name: webapp-secure 
(...)
```

</details>