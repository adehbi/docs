# Guide POC WSO2 Integrator — nomClient
## Flux d'intégration Ballerina avec OpenBao, GitLab CI/CD et Kubernetes

---

> **À qui s'adresse ce guide ?**
> Ce guide est destiné à un développeur ou architecte qui souhaite reproduire de zéro le POC WSO2 Integrator réalisé pour nomClient. Il couvre l'implémentation d'un flux d'intégration complet (CU1), la gestion des secrets via OpenBao, le déploiement sur Kubernetes via GitLab CI/CD, et la supervision via le WSO2 Integrator Control Plane (ICP).

---

## Table des matières

1. [Prérequis](#1-prérequis)
2. [Architecture générale](#2-architecture-générale)
3. [Création du projet Ballerina](#3-création-du-projet-ballerina)
4. [Implémentation du flux CU1](#4-implémentation-du-flux-cu1)
5. [Préparation PostgreSQL et RabbitMQ](#5-préparation-postgresql-et-rabbitmq)
6. [Intégration OpenBao en local](#6-intégration-openbao-en-local)
7. [Connexion au WSO2 Integrator Control Plane](#7-connexion-au-wso2-integrator-control-plane)
8. [Packaging Docker et CI/CD GitLab](#8-packaging-docker-et-cicd-gitlab)
9. [Déploiement sur Kubernetes avec OpenBao](#9-déploiement-sur-kubernetes-avec-openbao)
10. [Références documentaires](#10-références-documentaires)

---

## 1. Prérequis

### Outils à installer

| Outil | Version testée | Lien |
|---|---|---|
| Ballerina Swan Lake | 2201.13.4 | https://ballerina.io/downloads/ |
| Docker Desktop | dernière | https://www.docker.com/products/docker-desktop/ |
| kubectl | dernière | https://kubernetes.io/docs/tasks/tools/ |
| Helm | dernière | https://helm.sh/docs/intro/install/ |
| Rancher Desktop (optionnel) | dernière | https://rancherdesktop.io/ |
| OpenBao CLI | 2.5.4 | https://openbao.org/docs/install/ |
| Git | dernière | https://git-scm.com/ |
| curl / Postman | — | — |

### Services locaux nécessaires

- **PostgreSQL** — base de données relationnelle
- **RabbitMQ** — broker de messages (avec le plugin management activé sur le port 15672)
- **OpenBao** — gestionnaire de secrets (fork open-source de HashiCorp Vault)

### Téléchargements WSO2

- **WSO2 Integrator** (profil Ballerina) — depuis https://wso2.com/integrator/ballerina-integrator/
- **WSO2 Integrator Control Plane (ICP)** — image Docker depuis le registry WSO2 :
  ```bash
  docker pull docker.wso2.com/wso2-integration-control-plane:latest
  ```

> **Note sur l'architecture Mac M1/M2** : L'image ICP est uniquement disponible en `linux/amd64`. Sur Mac Apple Silicon, il faut utiliser `--entrypoint /bin/bash` avec le script de démarrage explicite (voir section 7).

---

## 2. Architecture générale

```
┌─────────────────────────────────────────────────────────┐
│                    CI/CD — GitLab                        │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │ GitLab repo  │→ │ Pipeline CI/CD  │→ │  Registry  │ │
│  │ code source  │  │ build · deploy  │  │  Docker    │ │
│  └──────────────┘  └─────────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────────┘
                            │ deploy
                            ▼
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes / Rancher                    │
│                                                          │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │  OpenBao (namespace  │  │  WSO2 Integrator ICP     │ │
│  │  openbao)            │  │  (Control Plane)          │ │
│  │  Vault Agent Injector│  │  port 9445 (Runtime)     │ │
│  └──────────────────────┘  │  port 9446 (Console)     │ │
│            │ inject         └──────────────────────────┘ │
│            ▼ Config.toml                ▲ heartbeat      │
│  ┌──────────────────────────────────────────────────┐    │
│  │         WSO2 Integrator — profil Ballerina        │    │
│  │         namespace: poc-wso2                       │    │
│  │   CU1 : API REST → PostgreSQL → XSLT → RabbitMQ  │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
         PostgreSQL     RabbitMQ       Partenaire
         (CU1 + CU3)   (file CU1)     externe (CU3)
```

**Principes clés :**
- Chaque intégration = 1 image Docker buildée par GitLab
- Aucun credential dans le code source commité
- Les secrets sont injectés par le Vault Agent Injector d'OpenBao au démarrage du pod
- Le WSO2 ICP supervise les runtimes via heartbeat

---

## 3. Création du projet Ballerina

### 3.1 Initialiser le projet

```bash
bal new cu1_integration
cd cu1_integration
```

Deux fichiers sont créés automatiquement : `main.bal` et `Ballerina.toml`.

### 3.2 Configurer Ballerina.toml

```toml
[package]
org = "nomClient"
name = "cu1_integration"
version = "0.1.0"
distribution = "2201.13.4"

[build-options]
observabilityIncluded = true
remoteManagement = true

[[dependency]]
org = "ballerinax"
name = "postgresql"
version = "1.18.0"

[[dependency]]
org = "ballerinax"
name = "postgresql.driver"
version = "1.6.3"

[[dependency]]
org = "ballerinax"
name = "rabbitmq"
version = "3.5.0"

[[dependency]]
org = "ballerina"
name = "xslt"
version = "2.9.1"

[[dependency]]
org = "wso2"
name = "icp.runtime.bridge"
version = "0.1.9"
```

> **Note** : `remoteManagement = true` est requis pour la connexion au WSO2 ICP.

### 3.3 Télécharger les dépendances

```bash
# Créer le dossier libs
mkdir libs

# Télécharger le driver JDBC PostgreSQL
curl -L https://jdbc.postgresql.org/download/postgresql-42.7.5.jar \
  -o libs/postgresql-42.7.5.jar

# Télécharger les packages Ballerina
bal pull ballerinax/postgresql:1.18.0
bal pull ballerinax/postgresql.driver:1.6.3
bal pull ballerinax/rabbitmq:3.5.0
bal pull ballerina/xslt:2.9.1
bal pull wso2/icp.runtime.bridge:0.1.9
```

> **Références** :
> - Packages Ballerina : https://central.ballerina.io
> - Driver JDBC PostgreSQL : https://jdbc.postgresql.org/download/

---

## 4. Implémentation du flux CU1

### 4.1 Flux métier

```
Consommateur → POST /api/commande (XML)
    → Validation XML + champs obligatoires
    → Stockage payload brut en PostgreSQL
    → Transformation XSLT (feuille de style externe)
    → Mise à jour PostgreSQL avec payload transformé
    → Publication sur file RabbitMQ
    → Réponse JSON au consommateur
```

### 4.2 Feuille de style XSLT

Créer le fichier `transform.xsl` à la racine du projet :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
    <xsl:template match="/commande">
        <ordre>
            <reference><xsl:value-of select="id"/></reference>
            <valeur><xsl:value-of select="montant"/></valeur>
            <client><xsl:value-of select="client"/></client>
            <statut>TRANSFORME</statut>
        </ordre>
    </xsl:template>
</xsl:stylesheet>
```

> La feuille de style est externalisée du code pour permettre aux équipes métier de la modifier sans toucher au code d'intégration.

### 4.3 Code Ballerina — main.bal

```ballerina
// =============================================================
// CU1 — Flux d'intégration : API REST → PostgreSQL + RabbitMQ
// POC WSO2 Integrator — nomClient
// =============================================================

import ballerina/http;
import ballerina/sql;
import ballerinax/postgresql;
import ballerinax/rabbitmq;
import ballerina/log;
import ballerina/xslt;
import ballerina/io;
import wso2/icp.runtime.bridge as _;

// -------------------------------------------------------------
// Variables configurables — peuplées depuis Config.toml
// En local : Config.toml manuel
// Sur Kubernetes : Config.toml injecté par Vault Agent OpenBao
// -------------------------------------------------------------
configurable string dbHost     = "";
configurable int    dbPort     = 5432;
configurable string dbUser     = "";
configurable string dbPassword = "";
configurable string dbName     = "";

configurable string mqHost     = "";
configurable int    mqPort     = 5672;
configurable string mqUser     = "";
configurable string mqPassword = "";

// -------------------------------------------------------------
// Clients déclarés sans initialisation.
// Initialisés dans init() après chargement des secrets.
// Pas de placeholder — aucun credential en dur dans le code.
// -------------------------------------------------------------
postgresql:Client? dbClient = ();
rabbitmq:Client? mqClient = ();

// -------------------------------------------------------------
// Initialisation du module — appelée automatiquement par
// Ballerina au démarrage avant tout traitement de requête.
// -------------------------------------------------------------
function init() returns error? {
    log:printInfo("Initialisation — chargement des secrets depuis Config.toml");

    log:printInfo("Connexion PostgreSQL", host = dbHost, port = dbPort, database = dbName);
    dbClient = check new postgresql:Client(
        host = dbHost,
        port = dbPort,
        username = dbUser,
        password = dbPassword,
        database = dbName
    );
    log:printInfo("PostgreSQL connecté avec succès");

    log:printInfo("Connexion RabbitMQ", host = mqHost, port = mqPort);
    mqClient = check new rabbitmq:Client(
        host = mqHost,
        port = mqPort,
        connectionData = {
            username: mqUser,
            password: mqPassword
        }
    );
    log:printInfo("RabbitMQ connecté avec succès");
}

// -------------------------------------------------------------
// Transformation XSLT via feuille de style externe
// -------------------------------------------------------------
function transformerXML(xml payload) returns xml|error {
    xml xslContent = check io:fileReadXml("transform.xsl");
    return check xslt:transform(payload, xslContent);
}

// -------------------------------------------------------------
// Service HTTP — exposé sur le port 8090
// Point d'entrée : POST /api/commande
// -------------------------------------------------------------
service /api on new http:Listener(8090) {

    resource function post commande(http:Caller caller, http:Request req)
            returns error? {

        // Vérification clients initialisés
        if dbClient is () || mqClient is () {
            log:printError("Clients non initialisés");
            check caller->respond({
                "statut": "ERREUR",
                "code": "ERREUR_INIT",
                "message": "Les clients ne sont pas initialisés"
            });
            return;
        }
        postgresql:Client db = <postgresql:Client>dbClient;
        rabbitmq:Client mq  = <rabbitmq:Client>mqClient;

        // Étape 1 — Réception et parsing du payload XML
        xml|error payloadResult = req.getXmlPayload();
        if payloadResult is error {
            log:printError("Payload XML invalide", err = payloadResult.message());
            check caller->respond({
                "statut": "ERREUR",
                "code": "XML_INVALIDE",
                "message": "Le payload reçu n'est pas un XML valide"
            });
            return;
        }
        xml payload = payloadResult;
        string payloadBrut = payload.toString();
        log:printInfo("Payload reçu");

        // Étape 2 — Validation des champs obligatoires
        string id      = (payload/**/<id>/*).toString();
        string montant = (payload/**/<montant>/*).toString();
        if id == "" || montant == "" {
            log:printError("Champs obligatoires manquants", id = id, montant = montant);
            check caller->respond({
                "statut": "ERREUR",
                "code": "CHAMPS_MANQUANTS",
                "message": "Les champs 'id' et 'montant' sont obligatoires"
            });
            return;
        }

        // Étape 3 — Stockage payload brut en PostgreSQL
        sql:ExecutionResult|sql:Error dbResult = db->execute(`
            INSERT INTO commandes (payload_brut, statut)
            VALUES (${payloadBrut}, 'RECU')
        `);
        if dbResult is sql:Error {
            log:printError("Erreur stockage PostgreSQL", err = dbResult.message());
            check caller->respond({
                "statut": "ERREUR",
                "code": "ERREUR_DB",
                "message": "Erreur lors du stockage en base de données"
            });
            return;
        }
        log:printInfo("Stockage PostgreSQL OK — payload brut");

        // Étape 4 — Transformation XSLT
        xml|error transformResult = transformerXML(payload);
        if transformResult is error {
            log:printError("Erreur transformation XSLT", err = transformResult.message());
            check caller->respond({
                "statut": "ERREUR",
                "code": "ERREUR_XSLT",
                "message": "Erreur lors de la transformation XSLT"
            });
            return;
        }
        xml payloadTransforme = transformResult;
        string payloadTransformeStr = payloadTransforme.toString();
        log:printInfo("Transformation XSLT OK", resultat = payloadTransformeStr);

        // Étape 5 — Mise à jour PostgreSQL avec payload transformé
        _ = check db->execute(`
            UPDATE commandes
            SET payload_transforme = ${payloadTransformeStr},
                statut = 'TRANSFORME'
            WHERE payload_brut = ${payloadBrut}
        `);
        log:printInfo("Mise à jour PostgreSQL OK");

        // Étape 6 — Publication RabbitMQ
        rabbitmq:Error|() mqResult = mq->publishMessage({
            content: payloadTransformeStr.toBytes(),
            routingKey: "commande",
            exchange: "poc.exchange"
        });
        if mqResult is rabbitmq:Error {
            log:printError("Erreur publication RabbitMQ", err = mqResult.message());
            check caller->respond({
                "statut": "ERREUR",
                "code": "ERREUR_MQ",
                "message": "Erreur lors de la publication sur RabbitMQ"
            });
            return;
        }
        log:printInfo("Publication RabbitMQ OK");

        // Étape 7 — Réponse au consommateur
        check caller->respond({
            "statut": "OK",
            "message": "Commande traitée avec succès",
            "payload_transforme": payloadTransformeStr
        });
    }
}
```

> **Références** :
> - Module `ballerina/xslt` : https://central.ballerina.io/ballerina/xslt/latest
> - Module `ballerinax/postgresql` : https://central.ballerina.io/ballerinax/postgresql/latest
> - Module `ballerinax/rabbitmq` : https://central.ballerina.io/ballerinax/rabbitmq/latest
> - Configurable variables Ballerina : https://ballerina.io/learn/provide-values-to-configurable-variables/

---

## 5. Préparation PostgreSQL et RabbitMQ

### 5.1 PostgreSQL — Créer la base et la table

```bash
# Créer la base de données
createdb <username>
psql -U <username> -c "CREATE DATABASE poc_wso2;"

# Créer la table
psql -U <username> -d poc_wso2 -c "
CREATE TABLE commandes (
    id SERIAL PRIMARY KEY,
    payload_brut TEXT NOT NULL,
    payload_transforme TEXT,
    statut VARCHAR(50) DEFAULT 'RECU',
    date_reception TIMESTAMP DEFAULT NOW()
);"
```

> Remplacer `<username>` par votre username PostgreSQL local (`whoami` pour le trouver).

### 5.2 RabbitMQ — Créer l'exchange, la queue et le binding

```bash
# Créer la queue
curl -u guest:guest -X PUT http://localhost:15672/api/queues/%2F/poc.commandes \
  -H "Content-Type: application/json" \
  -d '{"durable": true}'

# Créer l'exchange
curl -u guest:guest -X PUT http://localhost:15672/api/exchanges/%2F/poc.exchange \
  -H "Content-Type: application/json" \
  -d '{"type":"direct","durable":true}'

# Créer le binding
curl -u guest:guest -X POST \
  http://localhost:15672/api/bindings/%2F/e/poc.exchange/q/poc.commandes \
  -H "Content-Type: application/json" \
  -d '{"routing_key":"commande"}'
```

### 5.3 Tester le flux en local

Créer le `Config.toml` à la racine du projet :

```toml
# Connexion PostgreSQL
dbHost     = "localhost"
dbPort     = 5432
dbUser     = "<username>"
dbPassword = ""
dbName     = "poc_wso2"

# Connexion RabbitMQ
mqHost     = "localhost"
mqPort     = 5672
mqUser     = "guest"
mqPassword = "<password>"

# WSO2 ICP Control Plane (voir section 7)
[wso2.icp.runtime.bridge]
environment = "dev"
project     = "poc-nomClient"
integration = "cu1-integration"
runtime     = "cu1-runtime-1"
secret      = "<secret-généré-par-icp>"
```

```bash
# Lancer l'intégration
bal run

# Tester — dans un autre terminal
curl -X POST http://localhost:8090/api/commande \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<commande>
  <id>CMD-001</id>
  <client>nomClient</client>
  <montant>1500.00</montant>
</commande>'

# Vérifier PostgreSQL
psql -U <username> -d poc_wso2 \
  -c "SELECT id, statut, date_reception, payload_transforme FROM commandes ORDER BY date_reception;"
```

**Réponse attendue :**
```json
{
  "statut": "OK",
  "message": "Commande traitée avec succès",
  "payload_transforme": "<ordre><reference>CMD-001</reference><valeur>1500.00</valeur><client>nomClient</client><statut>TRANSFORME</statut></ordre>"
}
```

### 5.4 Tester la gestion d'erreurs

```bash
# XML mal formé
curl -X POST http://localhost:8090/api/commande \
  -H "Content-Type: application/xml" \
  -d 'ceci nest pas du xml'

# Champ manquant
curl -X POST http://localhost:8090/api/commande \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<commande>
  <id>CMD-002</id>
  <client>nomClient</client>
</commande>'
```

### 5.5 Configuration des niveaux de log

```toml
# Dans Config.toml — niveau global
[ballerina.log]
level = "DEBUG"   # DEBUG, INFO, WARN, ERROR

# Par module
[[ballerina.log.modules]]
name  = "nomClient/cu1_integration"
level = "INFO"
```

Pour utiliser des configs différentes par environnement :

```bash
# Dev
bal run

# Prod
BAL_CONFIG_FILES=Config.prod.toml bal run
```

> **Référence** :
> - Configuration des logs Ballerina : https://ballerina.io/learn/by-example/logging-configuration/
> - Spécification du module log : https://ballerina.io/spec/log/

---

## 6. Intégration OpenBao en local

> OpenBao est un fork open-source de HashiCorp Vault. Il expose la même API REST.
> Documentation : https://openbao.org/docs/

### 6.1 Démarrer OpenBao en local

```bash
# Configurer l'adresse
export VAULT_ADDR='http://127.0.0.1:8200'

# Vérifier le statut
bao status
```

### 6.2 Stocker les secrets du CU1

```bash
# Activer le moteur de secrets KV
bao secrets enable -path=poc-wso2 kv

# Stocker les secrets
bao kv put poc-wso2/cu1 \
  db_host="localhost" \
  db_port="5432" \
  db_user="<username>" \
  db_password="" \
  db_name="poc_wso2" \
  mq_host="localhost" \
  mq_port="5672" \
  mq_user="guest" \
  mq_password="<password>"

# Vérifier
bao kv get poc-wso2/cu1
```

### 6.3 Pourquoi OpenBao ?

En mode local pour ce POC, OpenBao permet de démontrer le principe de gestion des secrets sans credentials en clair dans le code. Sur Kubernetes, le mode d'intégration est différent et plus propre (voir section 9) — le Vault Agent Injector monte le `Config.toml` directement dans le pod avant son démarrage, sans appel HTTP depuis le code.

> **Note** : Le `Config.toml` est exclu du repo Git via `.gitignore` — c'est lui qui contient les paramètres de connexion locaux. Aucun secret n'est jamais commité.

---

## 7. Connexion au WSO2 Integrator Control Plane

### 7.1 Lancer le Control Plane ICP via Docker

> L'image est disponible uniquement en `linux/amd64`. Sur Mac Apple Silicon (M1/M2), utiliser `--platform linux/amd64` avec l'entrypoint explicite.

```bash
docker run -d \
  --name wso2-icp \
  --platform linux/amd64 \
  -p 9446:9446 \
  -p 9445:9445 \
  -p 9447:9447 \
  -p 9164:9164 \
  --entrypoint /bin/bash \
  docker.wso2.com/wso2-integration-control-plane:latest \
  /home/wso2carbon/wso2-integration-control-plane-2.0.0/bin/icp.sh
```

**Ports exposés :**
- `9446` — Console web (GraphQL + UI)
- `9445` — Runtime service (heartbeat des intégrations)
- `9447` — Authentication service
- `9164` — Observability

### 7.2 Accéder au dashboard

Ouvrir https://localhost:9446 — credentials par défaut : `admin` / `admin`.

### 7.3 Générer un secret ICP

1. Dans le dashboard → **Runtimes** → **Add Runtime** → **Generate Secret**
2. Copier le snippet `Config.toml` affiché (le secret n'est montré qu'une fois)

### 7.4 Configurer l'intégration Ballerina

Dans `Config.toml`, ajouter le snippet copié :

```toml
[wso2.icp.runtime.bridge]
environment = "dev"
project     = "poc-nomClient"
integration = "cu1-integration"
runtime     = "cu1-runtime-1"
secret      = "<secret-généré>"
# serverUrl = "https://localhost:9445"  # décommenter si ICP n'est pas sur localhost
```

Dans `Ballerina.toml`, s'assurer que `remoteManagement = true` est présent :

```toml
[build-options]
observabilityIncluded = true
remoteManagement = true
```

Dans `main.bal`, l'import suivant active automatiquement le bridge :

```ballerina
import wso2/icp.runtime.bridge as _;
```

Au démarrage, les logs suivants confirment la connexion :

```
level=INFO module=wso2/icp.runtime.bridge message="ICP agent initialized with server URL: https://localhost:9445"
level=INFO module=wso2/icp.runtime.bridge message="Full heartbeat acknowledged by ICP server"
```

> **Références** :
> - Connecter une intégration à ICP : https://wso2.com/integration-platform/docs/manage/icp/connect-runtime
> - ICP Runtime Bridge (GitHub) : https://github.com/wso2/icp-runtime-bridge

---

## 8. Packaging Docker et CI/CD GitLab

### 8.1 Structure du projet Git

```
cu1-integration/
├── main.bal
├── vault.bal              # (optionnel, mode local uniquement)
├── transform.xsl
├── Ballerina.toml
├── Config.toml            # exclu du repo via .gitignore
├── libs/
│   └── postgresql-42.7.5.jar
├── target/
│   └── bin/
│       └── cu1_integration.jar   # pré-buildé localement
├── Dockerfile
├── k8s/
│   └── deployment.yaml
└── .gitlab-ci.yml
```

### 8.2 .gitignore

```
Config.toml
.ballerina/
```

> `libs/` et `target/bin/` sont commités pour que le pipeline Docker n'ait pas à recompiler (les shared runners GitLab ont des timeouts réseau qui empêchent le téléchargement des dépendances Ballerina).

### 8.3 Dockerfile

```dockerfile
FROM ballerina/ballerina:2201.13.4
WORKDIR /app
COPY target/bin/cu1_integration.jar .
COPY transform.xsl .
CMD ["java", "-jar", "cu1_integration.jar"]
```

> L'image de base Ballerina inclut la JVM nécessaire. Le jar pré-buildé est copié directement sans recompilation dans le container.

### 8.4 Pipeline GitLab — .gitlab-ci.yml

```yaml
stages:
  - build
  - deploy

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest

deploy:
  stage: deploy
  image: alpine/k8s:1.29.2
  before_script:
    - mkdir -p /root/.kube
    - echo "$KUBE_CONFIG" | base64 -d > /root/.kube/config
  script:
    - kubectl apply -f k8s/deployment.yaml --validate=false
    - kubectl rollout status deployment/cu1-integration -n poc-wso2
  only:
    - main
```

### 8.5 Variables GitLab à configurer

Dans **GitLab → Settings → CI/CD → Variables** :

| Variable | Description | Type | Masked |
|---|---|---|---|
| `KUBE_CONFIG` | Contenu du kubeconfig en base64 | Variable | ✅ |

```bash
# Récupérer le kubeconfig en base64
cat ~/.kube/config | base64
```

### 8.6 Authentification registry GitLab

Si le projet GitLab est privé, créer un Personal Access Token (scope `write_repository`) et configurer le remote :

```bash
git remote set-url origin \
  https://oauth2:<TOKEN>@gitlab.com/<groupe>/<projet>.git
```

### 8.7 Secret pour le registry Kubernetes

```bash
kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
  --docker-username=<username-gitlab> \
  --docker-password=<token-gitlab> \
  --docker-email=<email> \
  -n poc-wso2
```

> **Référence** :
> - Container Registry GitLab : https://docs.gitlab.com/ee/user/packages/container_registry/

---

## 9. Déploiement sur Kubernetes avec OpenBao

> Cette section décrit l'intégration complète OpenBao sur Kubernetes, où les secrets sont injectés automatiquement dans le pod via le Vault Agent Injector — sans aucun appel HTTP depuis le code applicatif.

### 9.1 Installer OpenBao sur Kubernetes

```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update
helm install openbao openbao/openbao \
  --namespace openbao \
  --create-namespace
```

Vérifier que les pods tournent :

```bash
kubectl get pods -n openbao
# NAME                                      READY   STATUS
# openbao-0                                 1/1     Running
# openbao-agent-injector-7b46465dd7-xxxxx   1/1     Running
```

### 9.2 Initialiser et désceller OpenBao

```bash
# Initialiser (à faire une seule fois)
kubectl exec -n openbao openbao-0 -- bao operator init \
  -key-shares=1 \
  -key-threshold=1

# ⚠️ SAUVEGARDER la clé de déscellement et le root token affichés !

# Désceller
kubectl exec -n openbao openbao-0 -- bao operator unseal \
  <UNSEAL_KEY>

# Vérifier
kubectl exec -n openbao openbao-0 -- bao status
```

### 9.3 Configurer l'auth Kubernetes

```bash
# Se connecter avec le root token
kubectl exec -n openbao openbao-0 -- bao login <ROOT_TOKEN>

# Activer l'auth Kubernetes
kubectl exec -n openbao openbao-0 -- bao auth enable kubernetes

# Configurer l'auth avec l'URL interne du cluster
kubectl exec -n openbao openbao-0 -- bao write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local" \
  token_reviewer_jwt="$(kubectl exec -n openbao openbao-0 -- \
    cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_ca_cert="@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
```

### 9.4 Créer la policy et le rôle

```bash
# Créer la policy
kubectl exec -n openbao openbao-0 -- /bin/sh -c \
  'bao policy write cu1-policy - << EOF
path "poc-wso2/cu1" {
  capabilities = ["read"]
}
EOF'

# Créer le rôle liant le ServiceAccount à la policy
kubectl exec -n openbao openbao-0 -- bao write auth/kubernetes/role/cu1-role \
  bound_service_account_names=cu1-integration \
  bound_service_account_namespaces=poc-wso2 \
  policies=cu1-policy \
  ttl=1h
```

### 9.5 Créer le namespace et le ServiceAccount

```bash
kubectl create namespace poc-wso2
kubectl create serviceaccount cu1-integration -n poc-wso2
```

### 9.6 Stocker les secrets dans OpenBao

```bash
# Activer le moteur KV
kubectl exec -n openbao openbao-0 -- bao secrets enable -path=poc-wso2 kv

# Stocker les secrets
# Note : utiliser host.docker.internal pour accéder aux services locaux
# depuis les pods Kubernetes sur Rancher Desktop
kubectl exec -n openbao openbao-0 -- bao kv put poc-wso2/cu1 \
  db_host="host.docker.internal" \
  db_port="5432" \
  db_user="<username>" \
  db_password="" \
  db_name="poc_wso2" \
  mq_host="host.docker.internal" \
  mq_port="5672" \
  mq_user="guest" \
  mq_password="<password>"
```

> **Note** : `host.docker.internal` est le hostname qui permet aux pods Kubernetes (Rancher Desktop) d'accéder aux services tournant sur le Mac hôte. En production, PostgreSQL et RabbitMQ seraient déployés sur Kubernetes.

### 9.7 Manifest Kubernetes — k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cu1-integration
  namespace: poc-wso2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cu1-integration
  template:
    metadata:
      labels:
        app: cu1-integration
      annotations:
        # Activer le Vault Agent Injector
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "cu1-role"
        # Secret à injecter
        vault.hashicorp.com/agent-inject-secret-config: "poc-wso2/cu1"
        # Template du Config.toml injecté
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "poc-wso2/cu1" -}}
          dbHost     = "{{ .Data.db_host }}"
          dbPort     = {{ .Data.db_port }}
          dbUser     = "{{ .Data.db_user }}"
          dbPassword = "{{ .Data.db_password }}"
          dbName     = "{{ .Data.db_name }}"
          mqHost     = "{{ .Data.mq_host }}"
          mqPort     = {{ .Data.mq_port }}
          mqUser     = "{{ .Data.mq_user }}"
          mqPassword = "{{ .Data.mq_password }}"

          [wso2.icp.runtime.bridge]
          environment = "dev"
          project     = "poc-nomClient"
          integration = "cu1-integration"
          runtime     = "cu1-runtime-k8s"
          secret      = "<icp-secret>"
          serverUrl   = "https://host.docker.internal:9445"
          {{- end }}
        vault.hashicorp.com/agent-inject-file-config: "Config.toml"
    spec:
      serviceAccountName: cu1-integration
      imagePullSecrets:
        - name: gitlab-registry
      containers:
        - name: cu1-integration
          image: registry.gitlab.com/<groupe>/<projet>:latest
          ports:
            - containerPort: 8090
          env:
            - name: BAL_CONFIG_FILES
              value: "/vault/secrets/Config.toml"
---
apiVersion: v1
kind: Service
metadata:
  name: cu1-integration
  namespace: poc-wso2
spec:
  selector:
    app: cu1-integration
  ports:
    - port: 8090
      targetPort: 8090
```

**Points clés du manifest :**
- `vault.hashicorp.com/agent-inject: "true"` — active le sidecar Vault Agent
- Le template génère le `Config.toml` à partir des secrets OpenBao
- `BAL_CONFIG_FILES` pointe vers le fichier injecté par le Vault Agent
- `serviceAccountName` doit correspondre au ServiceAccount lié au rôle OpenBao

### 9.8 Déployer et vérifier

```bash
# Déployer
kubectl apply -f k8s/deployment.yaml

# Vérifier l'état du pod (2 containers : vault-agent + cu1-integration)
kubectl get pods -n poc-wso2 -w

# Vérifier les logs du Vault Agent (injection des secrets)
kubectl logs -n poc-wso2 <pod-name> -c vault-agent-init

# Vérifier les logs de l'intégration
kubectl logs -n poc-wso2 <pod-name> -c cu1-integration
```

**Logs attendus au démarrage :**
```
level=INFO module=wso2/icp.runtime.bridge message="Full heartbeat acknowledged by ICP server"
level=INFO module=nomClient/cu1_integration message="Initialisation — chargement des secrets depuis Config.toml"
level=INFO module=nomClient/cu1_integration message="PostgreSQL connecté avec succès"
level=INFO module=nomClient/cu1_integration message="RabbitMQ connecté avec succès"
```

### 9.9 Tester le flux depuis Kubernetes

```bash
# Récupérer l'IP du pod
POD_IP=$(kubectl get pod -l app=cu1-integration -n poc-wso2 \
  -o jsonpath='{.items[0].status.podIP}')

# Tester depuis l'intérieur du cluster
kubectl run test-curl --rm -it --restart=Never \
  --image=curlimages/curl -n poc-wso2 -- \
  curl -X POST http://$POD_IP:8090/api/commande \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><commande><id>CMD-K8S-001</id><client>nomClient</client><montant>2000.00</montant></commande>'
```

**Réponse attendue :**
```json
{
  "statut": "OK",
  "message": "Commande traitée avec succès",
  "payload_transforme": "<ordre><reference>CMD-K8S-001</reference><valeur>2000.00</valeur><client>nomClient</client><statut>TRANSFORME</statut></ordre>"
}
```

### 9.10 Architecture secrets — comparaison local vs Kubernetes

| Aspect | Local (Mac) | Kubernetes |
|---|---|---|
| Source des secrets | `Config.toml` manuel | Vault Agent Injector |
| `Config.toml` | Fichier local exclu du repo | Généré dynamiquement par le sidecar |
| Auth OpenBao | Token en clair dans `Config.toml` | ServiceAccount Kubernetes — zéro credential |
| Credentials dans le code | Aucun | Aucun |
| `Config.toml` commité | Non (`.gitignore`) | N/A — monté dans le pod |

> **Références** :
> - OpenBao Helm chart : https://openbao.org/docs/platform/k8s/helm/
> - Auth Kubernetes OpenBao : https://openbao.org/docs/auth/kubernetes/
> - Vault Agent Injector annotations : https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations
> - ConfigMap Kubernetes : https://kubernetes.io/docs/concepts/configuration/configmap/

---

## 10. Références documentaires

### WSO2 Integrator

| Sujet | Lien |
|---|---|
| Téléchargement WSO2 Integrator (Ballerina) | https://wso2.com/integrator/ballerina-integrator/ |
| Documentation WSO2 Integration Platform | https://wso2.com/integration-platform/docs/ |
| Connecter une intégration à ICP | https://wso2.com/integration-platform/docs/manage/icp/connect-runtime |
| ICP Console Overview | https://wso2.com/integration-platform/docs/manage/icp/icp-console-overview |
| ICP Runtime Bridge (GitHub) | https://github.com/wso2/icp-runtime-bridge |
| Image Docker ICP | https://hub.docker.com/r/wso2/wso2-integration-control-plane |

### Ballerina

| Sujet | Lien |
|---|---|
| Téléchargement Ballerina | https://ballerina.io/downloads/ |
| Ballerina Central (packages) | https://central.ballerina.io |
| Module postgresql | https://central.ballerina.io/ballerinax/postgresql/latest |
| Module rabbitmq | https://central.ballerina.io/ballerinax/rabbitmq/latest |
| Module xslt | https://central.ballerina.io/ballerina/xslt/latest |
| Module log | https://central.ballerina.io/ballerina/log/latest |
| Spécification log | https://ballerina.io/spec/log/ |
| Configuration des logs | https://ballerina.io/learn/by-example/logging-configuration/ |
| Configurable variables | https://ballerina.io/learn/provide-values-to-configurable-variables/ |
| Build GraalVM natif | https://ballerina.io/learn/build-the-executable-in-a-container/ |

### OpenBao

| Sujet | Lien |
|---|---|
| Documentation OpenBao | https://openbao.org/docs/ |
| Installation OpenBao | https://openbao.org/docs/install/ |
| Auth Kubernetes | https://openbao.org/docs/auth/kubernetes/ |
| Helm chart OpenBao | https://openbao.org/docs/platform/k8s/helm/ |

### Kubernetes & Docker

| Sujet | Lien |
|---|---|
| Vault Agent Injector annotations | https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations |
| ConfigMap Kubernetes | https://kubernetes.io/docs/concepts/configuration/configmap/ |
| ServiceAccount Kubernetes | https://kubernetes.io/docs/concepts/security/service-accounts/ |
| Container Registry GitLab | https://docs.gitlab.com/ee/user/packages/container_registry/ |
| GitLab CI/CD Variables | https://docs.gitlab.com/ee/ci/variables/ |

---

## Annexe — Checklist de démarrage rapide

```bash
# 1. Vérifier OpenBao local
export VAULT_ADDR='http://127.0.0.1:8200'
bao status

# 2. Vérifier le Control Plane ICP
docker ps | grep wso2-icp
# Si arrêté :
docker start wso2-icp

# 3. Vider la table PostgreSQL pour une démo propre
psql -U <username> -d poc_wso2 -c "TRUNCATE commandes;"

# 4. Vérifier les queues RabbitMQ
# Ouvrir http://localhost:15672 → Queues → poc.commandes → Purge Messages

# 5. Lancer l'intégration en local
cd cu1_integration
bal run

# 6. Tester
curl -X POST http://localhost:8090/api/commande \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<commande>
  <id>CMD-001</id>
  <client>nomClient</client>
  <montant>1500.00</montant>
</commande>'

# 7. Vérifier PostgreSQL
psql -U <username> -d poc_wso2 \
  -c "SELECT id, statut, date_reception, payload_transforme FROM commandes ORDER BY date_reception;"
```

---

*Guide rédigé dans le cadre du POC WSO2 Integration — nomClient*
*Version 1.0 — Juin 2026*
