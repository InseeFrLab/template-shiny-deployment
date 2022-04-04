# Tutoriel : déployer une application Shiny sur le SSP Cloud

Ce tutoriel documente le processus de déploiement d'une application R Shiny sur un cluster Kubernetes. La configuration présentée est spécifiquement adaptée au [SSP Cloud](https://datalab.sspcloud.fr/home). L'objectif du procesuss est d'automatiser le déploiement au maximum pour favoriser sa reproductibilité : le processus prend en entrée les données nécessaires à l'application Shiny, et renvoie en output une URL qui permet d'accéder à l'application Shiny sur internet.

### Mettre les données d'entrée sur MinIO

La première étape du processus de déploiement consiste à stocker les données d'entrée de manière à les rendre accessibles à partir du cluster. Les données peuvent être mises sur votre bucket personnel, ou bien sur un bucket partagé en cas de projet collaboratif. 

La méthode la plus simple est d'uploader les données à partir d'une interface graphique, que ce soit celle (en cours de développement) du [Datalab](https://datalab.sspcloud.fr/mes-fichiers) ou bien directement la [console MinIO](https://minio-console.lab.sspcloud.fr/buckets). Il est également possible d'interagir avec MinIO en R (RStudio), Python (JupyterLab ou VSCode), ou dans un terminal avec le client [mc](https://docs.min.io/docs/minio-client-complete-guide.html). La [documentation](https://docs.sspcloud.fr/onyxia-guide/stockage-de-donnees) de l'instance SSP Cloud détaille ces différentes méthodes.

### Développement de l'application

#### Phase de développement en self

Le développement d'une application Shiny commence généralement en self, que ce soit sur poste ou sur un environnement de calcul dédié. En général, cet environnement est différent de celui sur lequel l'application va être déployée. Par exemple, il est fréquent de développer son application sur son poste de travail, donc avec Windows, là où les serveurs utilisent généralement Linux.

Ces différences entre les environnements de développement et de production vont nécessiter des ajustements au moment du déploiement, qui peuvent être substantiels. Afin de limiter ce risque, il est conseillé de commencer à développer le plus tôt possible sur un environnement proche de celui de production. Par exemple, sur le SSP Cloud, cela peut consister à développer dans un service RStudio, lancé à partir du [catalogue de services](https://datalab.sspcloud.fr/catalog/inseefrlab-helm-charts-datascience). Ce service ne permet pas en soi de déployer une application Shiny à des utilisateurs, mais il permet de développer dans le même environnement que celui de déploiement, de tester son application Shiny de manière interactive et de procéder aux ajustements nécessaires (résolution de bugs, téléchargement des librairies nécessaires, conflits de version, etc.).

#### Packaging

Une autre bonne pratique de développement qui facilite grandement le déploiement d'une application est de structurer son projet comme un package R. Cela permet notamment de faciliter la gestion des dépendances en les spécifiant explicitement — au lieu de les appeler via les instructions `require` ou `library` — et de permettre un déploiement standardisé en normalisant la structure attendue du package contenant l'application. 

Le repo [shiny-app](https://github.com/InseeFrLab/template-shiny-app) propose un template d'application Shiny sous forme de package R. Voici le détail de sa structure :

```bash
shiny-app
├── Dockerfile
├── .gitlab-ci.yml
├── myshinyapp
│   ├── DESCRIPTION
│   ├── inst
│   │   └── app
│   │       ├── server.R
│   │       └── ui.R
│   ├── man
│   │   └── hello.Rd
│   ├── myshinyapp.Rproj
│   ├── NAMESPACE
│   └── R
│       ├── data.R
│       └── main.R
└── README.md
```

L'application Shiny est intégrée dans un package R appelé `myshinyapp`. Les parties `server` et `ui` de l'application sont contenus dans des scripts du même nom, situés dans un dosiser `inst/app`. Comme tout package R, les fonctions principales sont contenues dans un dossier `R`, séparées en modules (scripts R contenant des fonctions) selon leur domaine. Par exemple, dans ce template, le module `data.R` contient des fonctions permettant de gérer la donnée (stockage MinIO, ingestion dans une table PostgreSQL), et le module `main.R` contient la fonction qui permet de lancer l'application Shiny. 

Le fichier DESCRIPTION doit être rempli avec précaution. Il contient les métadonnées essentielles du package (titre, version, description..). En particulier, il faut spécifier les dépendances de l'application, i.e. les différents packages R tiers nécessaires au bon foncitonnement de l'application.

Afin de faciliter le déploiement, il est recommandé de suivre au maximum la structure proposée via ce template. Pour cela, on peut forker le projet sur GitLab et l'adapter à son application en modifiant les titres, contenus des scripts, documentation, etc.

#### Conteneurisation

Pour pouvoir être déployée sur un cluster Kubernetes, une application doit nécessairement être mise à disposition sous la forme d'une image Docker. Concrètement, cette étape permet de rendre l'application portable : une fois que l'image Docker est construite et fonctionne correctement, elle peut être déployée sur n'importe quel environnement d'éxécution avec la certitude qu'elle fonctionnera de manière attendue, peu importe l'environnement qui a servir à la développer.

Le fichier `Dockerfile` situé à la racine du projet contient une suite d'instructions qui permettent de conteneuriser l'application, sous la forme d'une image Docker. Ce fichier contient 5 parties :
- **appel de l'image Docker de base** : `rocker/shiny`. Il n'est généralement pas nécessaire de changer cette image.
- **installation des librairies système** nécessaires pour installer les packages R utilisés par l'application. Cette liste se construit par un processus itératif : build l'image docker -> regarder les logs -> trouver les packages qui n'ont pas réussi à s'installer -> les logs spécifient généralement les librairies système manquantes -> ajouter les librairies manquantes au Dockerfile -> build l'image docker -> ...
- **installation du package R et de ses dépendances**. Si les dépendances ont été correctement spécifiées dans le fichier DESCRIPTION, il n'est pas nécessaire de changer cette partie.
- **exposer le port utilisé par l'application**. Il n'est généralement pas nécessaire de changer le port exposé.
- **entrypoint**, i.e. la commande de lancement du conteneur. Il n'est pas nécessaire de modifier cette commande si le nom de la fonction dans le fichier `main.R` n'a pas été modifié.

#### Intégration continue (CI)

Le fichier `.github/workflows/ci.yaml` contient une suite d'instructions qui vont s'éxécuter à chaque fois qu'une modification du code sur le repository GitLab est effectuée. C'est l'approche de l'intégration continue : à chaque fois que le code source de l'application est modifié (nouvelles fonctionnalités, correction de bugs, etc.), l'image Docker est automatiquement reconstruite et envoyée sur le registry Docker de votre choix.

### Déploiement de l'application

#### Chart Helm de l'application

Le déploiement de l'application nécessite la création d'un chart Helm. Concrètement, un chart Helm peut être vu comme un package Kubernetes, contenant les ressources nécessaires au déploiement d'une application. 

Ce repository contient un template de chart Helm permettant le déploiement de l'application [shiny-app](https://github.com/InseeFrLab/template-shiny-app). Il convient donc de forker également ce second repository, qui va servir de base pour le chart Helm de votre application. Ce chart contient pour l'essentiel deux fichiers.

Le fichier `Chart.yaml` contient les métadonnées du chart (nom, version) ainsi que ses dépendances, i.e. les potentiels autres charts Helm dont il hérite. Dans notre cas, on voit que le chart hérite du [chart Shiny](https://github.com/InseeFrLab/helm-charts/tree/master/charts/shiny) d'InseeFrLab. Ce chart spécifie généralement les ressources Kubernetes nécessaires au déploiement d'une application Shiny, de sorte à ce que l'on ait qu'à modifier les valeurs d'instanciation pour déployer notre application.

Le fichier `values.yaml` contient précisément les valeurs que l'on modifie par rapport au chart général. Les modifications à apporter dépendent naturellement de ce que réalise en pratique l'application, car cela conditionne les ressources dont elle a besoin. Dans un premier temps, il nous faut modifier : 
- le chemin et nom de l'image (paramètre `shiny.image.repository`)
- le tag de l'image, i.e. sa version (paramètre `shiny.image.tag`)
- l'host, i.e. l'URL à laquelle l'application sera accessible une fois déployée (paramètre shiny.ingress.hosts[0].host)

#### Utilisation du stockage de données S3 avec MinIO

Si l'application Shiny utilise des données en entrée stockées sur MinIO, il faut donner la valeur `true` au paramètre `shiny.s3.enabled`. 

Par ailleurs, il faut fournir à l'application les **informations d'authentification** au service de stockage. Ces informations sont sensibles, et ne doivent donc jamais figurer en clair dans le code source de l'application. Pour éviter ce risque, on va inscrire ces informations dans un objet Kubernetes appelé **Secret**, qui va nous permettre de les passer à l'application sous la forme de **variables d'environnement**.

La première étape est de créer un compte de service sur la [console MinIO](https://minio-console.lab.sspcloud.fr/). Pour ce faire :
- onglet "Service Accounts" -> "Create Service Account" -> "Create"
- comme précédemment, conserver à l'écran les informations de connexion

La seconde étape est de créer un Secret Kubernetes contenant ces informations. Voici un template de Secret à utiliser :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myshinyapp-s3
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: changeme
  AWS_SECRET_ACCESS_KEY: changeme
  AWS_S3_ENDPOINT: minio.lab.sspcloud.fr
  AWS_DEFAULT_REGION: us-east-1
```

Les valeurs de `AWS_ACCESS_KEY_ID` et `AWS_SECRET_ACCESS_KEY` sont à remplacer par les valeurs obtenues à l'étape précédente sur la console MinIO. Les valeurs de `AWS_S3_ENDPOINT` et `AWS_DEFAULT_REGION` n'ont pas besoin d'être modifiées pour une utilisation sur le cluster. Enfin, le nom du Secret (variable `metadata.name`) doit correspondre à la valeur de la variable `shiny.s3.existingSecret` du fichier `values.yaml`.

Pour être accessible dans l'application, ce secret doit être appliqué comme une ressource dans le namespace Kubernetes dans lequel sera déployé l'application. Pour cela :
- mettre le template de secret dans un fichier `sc-s3.yaml` et remplacer les valeurs comme indiqué ci-dessus
- dans un terminal, exécuter `kubectl apply -f sc-s3.yaml`
- si tout a bien fonctionné, un message devrait confirmer la création du secret.

Une fois le secret appliqué, les quatre variables d'environnement définies dans le secret sont accessibles dans l'application. Vu que ces variables sont standards, il est alors possible de se connecter au stockage MinIO via le package R `aws.s3` sans même avoir besoin de les préciser. Le fichier [data.R](https://github.com/InseeFrLab/template-shiny-app/blob/main/myshinyapp/R/data.R) montre comment écrire et lire des données sur MinIO une fois que ces variables d'environnement ont été créées.

#### Utilisation d'une base de données PostgreSQL

Si l'application Shiny utilise une base PostgreSQL, il faut donner la valeur `true` au paramètre `shiny.postgresql.enabled`. Il est par ailleurs possible de changer les paramètres `shiny.postgresql.username` (nom d'utilisateur), `shiny.postgresql.database` (nom de la base de données) et `shiny.postgresql.fullnameOverride` (nom de domaine du service PostgreSQL) à sa guise, sachant que ces paramètres seront de toute manière passés automatiquement à l'application sous forme de variables d'environnement.

Les mots de passe de connexion, donnée sensible, doivent quant à eux être passés à l'application via un Secret Kubernetes. La procédure est la même que précédemment, et le template de Secret à utiliser est : 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myshinyapp-postgresql
type: Opaque
stringData:
  password: changeme
  postgres-password: changeme
  replication-password: changeme
```

Trois passwords sont nécessaires, mais seul le champ `password` (password utilisateur) sera utilisé en pratique dans l'application. Il est donc possible de fixer le même password pour les trois champs sans trop de risque.

Là encore, toutes ces informations (valeurs et secrets) seront passées à l'application sous la forme de variables d'environnement. Le fichier [data.R](https://github.com/InseeFrLab/template-shiny-app/blob/main/myshinyapp/R/data.R) montre comment se connecter à la base PostgreSQL et y écrire de la donnée, et le fichier [server.R](https://github.com/InseeFrLab/template-shiny-app/blob/main/myshinyapp/inst/app/server.R) montre comment se connecter à la base PostgreSQL et y lire de la donnée.

#### Déploiement du chart Helm

Finalement, pour déployer l'application sur le cluster :
- lancer un service VSCode sur le cluster *en mettant des droits Admin sur le namespace Kubernetes* (à l'initialisation du service dans l'IHM : onglet Kubernetes -> "Role" -> sélectionner "admin")
- lancer un terminal
- cloner le repository contenant le chart de **votre** application (pas le template)
- exécuter la commande : `helm install nom_du_repo --generate-name`

Si tout a fonctionné, un message devrait confirmer l'instanciation du chart, est l'application devrait désormais être disponible à l'URL spécifiée dans le fichier `values.yaml`.

### TODO

- industrialiser avec Golem
- activer les extensions pour faire du PostGIS
- LDAP et utilisation concurrente avec ShinyProxy
- GitOps avec Argo CD
- debugging des Pods
