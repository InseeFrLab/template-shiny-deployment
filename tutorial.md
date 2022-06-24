# Un template de déploiement d'une application Shiny sur le SSP Cloud

Ce tutoriel documente le processus de déploiement d'une [application R Shiny](https://github.com/InseeFrLab/template-shiny-app) sur un cluster Kubernetes. La configuration présentée est spécifiquement adaptée au [SSP Cloud](https://datalab.sspcloud.fr/home). L'objectif du procesuss est d'automatiser le déploiement au maximum pour favoriser sa reproductibilité : le processus prend en entrée les données nécessaires à l'application Shiny, et renvoie en output une URL qui permet d'accéder à l'application Shiny sur internet.

### Mettre les données d'entrée sur MinIO

La première étape du processus de déploiement consiste à stocker les données d'entrée de manière à les rendre accessibles à partir du cluster. Les données peuvent être mises sur votre bucket personnel, ou bien sur un bucket partagé en cas de projet collaboratif. 

La méthode la plus simple est d'uploader les données à partir d'une interface graphique, que ce soit celle (en cours de développement) du [Datalab](https://datalab.sspcloud.fr/mes-fichiers) ou bien directement la [console MinIO](https://minio-console.lab.sspcloud.fr/buckets). Il est également possible d'interagir avec MinIO en R (RStudio), Python (JupyterLab ou VSCode), ou dans un terminal avec le client [mc](https://docs.min.io/docs/minio-client-complete-guide.html). La [documentation](https://docs.sspcloud.fr/onyxia-guide/stockage-de-donnees) du projet Onyxia détaille ces différentes méthodes.

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

L'application Shiny est intégrée dans un package R appelé `myshinyapp`. Les parties `server` et `ui` de l'application sont contenues dans des scripts du même nom, situés dans un dossier `inst/app`. Comme tout package R, les fonctions principales sont contenues dans un dossier `R`, réparties en modules (fichiers `.R` contenant des fonctions) selon leur domaine. Par exemple, dans ce template, le module `data.R` contient des fonctions permettant de gérer la donnée (stockage MinIO, ingestion dans une table PostgreSQL), et le module `main.R` contient la fonction qui permet de lancer l'application Shiny. 

Le fichier `DESCRIPTION` doit être rempli avec précaution. Il contient les métadonnées essentielles du package (titre, version, description..). En particulier, il faut spécifier les dépendances de l'application, i.e. les différents packages R tiers nécessaires au bon foncitonnement de l'application.

Afin de faciliter le déploiement, il est recommandé de suivre au maximum la structure proposée via ce template. Pour cela, on peut forker le projet sur GitLab et l'adapter à son application en modifiant les titres, contenus des scripts, documentation, etc.

#### Conteneurisation

Pour pouvoir être déployée sur un cluster Kubernetes, une application doit nécessairement être mise à disposition sous la forme d'une image Docker. Concrètement, cette étape permet de rendre l'application portable : une fois que l'image Docker est construite et fonctionne correctement, elle peut être déployée sur n'importe quel environnement d'éxécution avec la certitude qu'elle fonctionnera de manière attendue, peu importe l'environnement qui a servir à la développer.

Le fichier `Dockerfile` situé à la racine du projet contient une suite d'instructions qui permettent de conteneuriser l'application, sous la forme d'une image Docker. Ce fichier contient 5 parties :
- **appel de l'image Docker de base** : `rocker/shiny`. Il n'est généralement pas nécessaire de changer cette image.
- **installation des librairies système** nécessaires pour installer les packages R utilisés par l'application. Cette liste se construit par un processus itératif :
```mermaid
flowchart TB;
  A[build l'image docker];
  B[les logs spécifient généralement les librairies système manquantes];
  C[regarder les logs];
  D[trouver les packages qui n'ont pas réussi à s'installer];
  E[ajouter les librairies manquantes au Dockerfile];
  
  A --> B
  B --> C
  C --> D
  D --> E
  E --> A
```
- **installation du package R et de ses dépendances**. Si les dépendances ont été correctement spécifiées dans le fichier DESCRIPTION, il n'est pas nécessaire de changer cette partie.
- **exposer le port utilisé par l'application**. Il n'est généralement pas nécessaire de changer le port exposé.
- **entrypoint**, i.e. la commande de lancement du conteneur. Il n'est pas nécessaire de modifier cette commande si le nom de la fonction dans le fichier `main.R` n'a pas été modifié.

#### Intégration continue (CI)

Le fichier `.github/workflows/ci.yaml` contient une suite d'instructions qui vont s'éxécuter à chaque fois qu'une modification du code sur le dépôt Git est effectuée. C'est l'approche de l'intégration continue : à chaque fois que le code source de l'application est modifié (nouvelles fonctionnalités, correction de bugs, etc.), l'image `Docker` est automatiquement reconstruite et envoyée sur le registry `Docker` de votre choix.

Dans ce tuto, on utilise la forge [DockerHub](https://hub.docker.com/), idéale pour les projets open-source. Une création de compte est nécessaire pour pouvoir l'utiliser, ainsi qu'un dépôt associé au projet. Une fois ces étapes effectuées, il faut modifier le nom de l'image dans le fichier [ci.yaml](https://github.com/InseeFrLab/template-shiny-app/blob/main/.github/workflows/ci.yaml#L19) avec vos informations : `<username_docker_hub>/<project_name>`.

### Déploiement de l'application

#### Chart Helm de l'application

Le déploiement de l'application nécessite la création d'un chart Helm. Concrètement, un chart Helm peut être vu comme un package Kubernetes, contenant les ressources nécessaires au déploiement d'une application. 

Ce dépôt contient un [chart Helm](https://helm.sh/) permettant le déploiement de l'[application template](https://github.com/InseeFrLab/template-shiny-app). Il convient donc de *forker* également ce second repository, qui va servir de base pour le chart `Helm` de votre application. Ce chart contient pour l'essentiel deux fichiers.

**Le fichier `Chart.yaml`** contient les métadonnées du chart ([nom](https://github.com/InseeFrLab/helm-charts-shiny-apps/blob/main/charts/quakes/Chart.yaml#L2), [version](https://github.com/InseeFrLab/helm-charts-shiny-apps/blob/main/charts/quakes/Chart.yaml#L6)) ainsi que ses dépendances, i.e. les potentiels autres charts `Helm` dont il hérite. Dans notre cas, [on voit](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/Chart.yaml#L5) que le chart hérite du chart [Shiny](https://github.com/InseeFrLab/helm-charts/tree/master/charts/shiny) d'[InseeFrLab](https://github.com/InseeFrLab). Ce chart spécifie généralement les ressources Kubernetes nécessaires au déploiement d'une application Shiny, de sorte à ce que l'on ait qu'à modifier les valeurs d'instanciation pour déployer notre application.

**Le fichier `values.yaml`** contient précisément les valeurs que l'on modifie par rapport au chart général. Les modifications à apporter dépendent naturellement de ce que réalise en pratique l'application, car cela conditionne les ressources dont elle a besoin. Dans un premier temps, il nous faut modifier : 
- [le chemin et nom de l'image](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L3) (paramètre `shiny.image.repository`)
- [le tag de l'image](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L4), i.e. sa version (paramètre `shiny.image.tag`)
- [l'hostname de l'Ingress](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L7), l'URL à laquelle l'application sera accessible une fois déployée (paramètre `shiny.ingress.hostname` avec comme nom de domaine obligatoire `lab.sspcloud.fr`); par exemple dans notre cas :`myshinyapp.lab.sspcloud.fr`.

#### Utilisation du stockage de données S3 avec MinIO

Si l'application Shiny utilise des données en entrée stockées sur MinIO, il faut donner la valeur `true` au [paramètre shiny.s3.enabled](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L9).

Par ailleurs, il faut fournir à l'application les **informations d'authentification** au service de stockage. Ces informations sont sensibles, et ne doivent donc jamais figurer en clair dans le code source de l'application. Pour éviter ce risque, on va inscrire ces informations dans un objet Kubernetes appelé **Secret**, qui va nous permettre de les passer à l'application sous la forme de **variables d'environnement**.

La première étape est de créer un compte de service sur la [console MinIO](https://minio-console.lab.sspcloud.fr/). Pour ce faire :
- menu "Identity" -> "Service Accounts" -> "Create Service Account" -> "Create"
- comme précédemment, conserver à l'écran les informations de connexion

La seconde étape est de créer un Secret Kubernetes contenant ces informations. Pour être accessible dans l'application, ce secret doit être appliqué comme une ressource dans le namespace Kubernetes dans lequel sera déployé l'application. Pour cela :
- Écrire le template suivant dans un fichier `quelconque.yaml` :

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

- Les valeurs de `AWS_ACCESS_KEY_ID` et `AWS_SECRET_ACCESS_KEY` sont à remplacer par les valeurs obtenues à l'étape précédente sur la console MinIO. Les valeurs de `AWS_S3_ENDPOINT` et `AWS_DEFAULT_REGION` n'ont pas besoin d'être modifiées pour une utilisation sur le cluster. Enfin, le nom du Secret (variable `metadata.name`) doit porter la même valeur que la variable [`shiny.s3.existingSecret`](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L10)

Pour être accessible dans l'application, ce secret doit être appliqué comme une ressource dans le namespace Kubernetes dans lequel sera déployé l'application. Pour cela :
- mettre le template de secret dans un fichier `quelconque.yaml` et remplacer les valeurs comme indiqué ci-dessus
- dans un terminal, exécuter `kubectl apply -f quelconque.yaml`
- si tout a bien fonctionné, un message devrait confirmer la création du secret. Du style `secret/nom_de_secret created` où `nom_de_secret` est ce que vous avez renseigné dans `metadata.name` du fichier `quelconque.yaml`.

Une fois le secret appliqué, les quatre variables d'environnement définies dans le secret sont accessibles dans l'application. Vu que ces variables sont standards, il est alors possible de se connecter au stockage MinIO via le package R `aws.s3` sans même avoir besoin de les préciser. Le fichier [data.R](https://github.com/InseeFrLab/template-shiny-app/blob/main/myshinyapp/R/data.R) montre comment écrire et lire des données sur MinIO une fois que ces variables d'environnement ont été créées.

#### Utilisation d'une base de données PostgreSQL

Si l'application Shiny utilise une base PostgreSQL, il faut donner la valeur `true` au paramètre [`shiny.postgresql.enabled`](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L12). Il est par ailleurs possible de changer les paramètres [`shiny.postgresql.username`](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L14) (nom d'utilisateur), [`shiny.postgresql.database`](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L15) (nom de la base de données) et [`shiny.postgresql.fullnameOverride`](https://github.com/InseeFrLab/template-shiny-deployment/blob/master/values.yaml#L19) (nom de domaine du service PostgreSQL) à sa guise, sachant que ces paramètres seront de toute manière passés automatiquement à l'application sous forme de variables d'environnement.

Les mots de passe de connexion, données sensibles, doivent quant à eux être passés à l'application via un Secret Kubernetes. La procédure est la même que précédemment, et le template de Secret à utiliser est : 

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

Trois passwords sont nécessaires, mais seul le champ `stringData.password` (password utilisateur) sera utilisé en pratique dans l'application. Il est donc possible de fixer le même password pour les trois champs de `stringData` sans trop de risque. Là encore, toutes ces informations (valeurs du chart et secrets) seront passées à l'application sous la forme de variables d'environnement, dont voici la liste :

|      **Variable**      |          **Description**          |
|:----------------------:|:---------------------------------:|
| POSTGRESQL_DB_NAME     | Nom de la BDD à créer             |
| POSTGRESQL_DB_HOST     | Nom d'hôte du service             |
| POSTGRESQL_DB_PORT     | Port utilisé par le service       |
| POSTGRESQL_DB_USER     | Nom de l'utilisateur à créer      |
| POSTGRESQL_DB_PASSWORD | Password de l'utilisateur à créer |

Le fichier [data.R](https://github.com/InseeFrLab/template-shiny-app/blob/main/myshinyapp/R/data.R#L24-L31) montre comment se connecter à la base PostgreSQL et y écrire de la donnée, et ces [lignes](https://github.com/InseeFrLab/template-shiny-app/blob/main/myshinyapp/inst/app/server.R#L6-L13)) du fichier `server.R` montre comment se connecter à la base PostgreSQL et y lire de la donnée.

#### Déploiement du chart Helm

Finalement, pour déployer l'application sur le cluster :
- lancer un service VSCode sur le cluster *en mettant des droits Admin sur le namespace Kubernetes* (à l'initialisation du service dans l'IHM : onglet Kubernetes -> "Role" -> sélectionner "admin")
- lancer un terminal
- cloner le dépôt contenant le chart de **votre** application (pas le template)
- importer les dépendances (en l'occurence, le chart [Shiny](https://github.com/InseeFrLab/helm-charts/tree/master/charts/shiny)) : `helm dependency update nom_du_repo`
- exécuter la commande : `helm install nom_du_repo --generate-name`

Si tout a fonctionné, un message devrait confirmer l'instanciation du chart, et l'application devrait désormais être disponible à l'URL spécifiée [dans le fichier `values.yaml`](https://github.com/InseeFrLab/helm-charts/blob/master/charts/shiny/values.yaml#L46).

### TODO

- debugging pods/helm
- industrialiser avec Golem
- LDAP et utilisation concurrente avec ShinyProxy
- GitOps avec Argo CD
