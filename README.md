# Pipeline CI/CD avec Jenkins, SonarQube et Docker

## 🎯 Objectifs pédagogiques

- Mettre en place Jenkins et configurer les outils (JDK, Maven, SonarScanner)
- Déployer SonarQube via Docker Compose et créer des projets + tokens par microservice
- Exposer Jenkins avec Ngrok et brancher GitHub via webhooks
- Créer un job Pipeline Jenkins et écrire un script de pipeline multi-stages
- Lancer/valider l'exécution (Jenkins, SonarQube, Docker) et vérifier le déclenchement par push

---

## 📋 Prérequis

### Prérequis techniques (outils)

| Outil | Description |
|-------|-------------|
| **JDK 17** | Version compatible avec le projet + variable `JAVA_HOME` configurée |
| **Maven** | Installé localement ou géré par Jenkins |
| **Git** | Ligne de commande |
| **Docker + Docker Compose** | Pour déployer SonarQube et les microservices |
| **Jenkins** | Installation locale |
| **SonarQube** | Déployé via Docker Compose avec PostgreSQL |
| **Ngrok** | Compte + authtoken pour exposer Jenkins |
| **GitHub** | Compte avec accès au dépôt du projet |

> ⚠️ **Remarque** : Le pipeline fourni utilise `bat` (agents Windows) et des chemins Windows. Sur Linux, remplacer `bat` par `sh` et adapter les chemins.

### Prérequis de connaissances

- **Git** : clone, commit, push, notion de webhook
- **Java/Spring Boot** : structure d'un projet, build Maven
- **Notions CI/CD** : stages (build, analyse, déploiement), exécution automatique

---

## 🏗️ Contexte et Architecture

L'application est composée de **4 microservices** :

```
┌─────────────────────────────────────────────────────────────┐
│                    Architecture Microservices                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│   │   Car    │   │  Client  │   │ Gateway  │               │
│   │ Service  │   │ Service  │   │ Service  │               │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘               │
│        │              │              │                      │
│        └──────────────┼──────────────┘                      │
│                       │                                      │
│              ┌────────▼────────┐                            │
│              │  Eureka Server  │                            │
│              │ (server_eureka) │                            │
│              └─────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### Le pipeline CI/CD assure :

1. **CI** : Compilation Maven + Analyse SonarQube (qualité, code smells, bugs, vulnérabilités)
2. **CD** : Déploiement des services dans des conteneurs via Docker Compose
3. **Automatisation** : Jenkins exécute le pipeline à chaque push/pull request via webhook GitHub exposé par Ngrok

---

## 📚 Table des matières

1. [Étape 1 : Récupération du projet GitHub](#étape-1--récupération-du-projet-github)
2. [Étape 2 : Installation et configuration de Jenkins](#étape-2--installation-et-configuration-de-jenkins)
3. [Étape 3 : Installation et configuration de SonarQube](#étape-3--installation-et-configuration-de-sonarqube)
4. [Étape 4 : Exposition de Jenkins via Ngrok et Webhooks GitHub](#étape-4--exposition-de-jenkins-via-ngrok-et-webhooks-github)
5. [Étape 5 : Création du Job Pipeline Jenkins](#étape-5--création-du-job-pipeline-jenkins)
6. [Étape 6 : Détail du script de pipeline Jenkins](#étape-6--détail-du-script-de-pipeline-jenkins)
7. [Étape 7 : Exécution du pipeline et vérifications](#étape-7--exécution-du-pipeline-et-vérifications)

---

## Étape 1 : Récupération du projet GitHub

### Introduction

Récupérer le code source et repérer la structure multi-services (car, client, gateway, server_eureka, deploy).

### 1.1 Cloner le dépôt

```bash
git clone https://github.com/lachgar/jenkins2.git
cd jenkins2
```

> 📌 Le dépôt indiqué est : `https://github.com/lachgar/jenkins2.git`

**Résultat attendu** : un dossier local contenant les répertoires microservices.

### 1.2 Vérifier la structure du dépôt

```powershell
# PowerShell
dir

# ou Bash
ls -la
```

**Structure attendue :**

```
jenkins2/
├── car/                 # Microservice voiture
│   └── pom.xml
├── client/              # Microservice client
│   └── pom.xml
├── gateway/             # API Gateway
│   └── pom.xml
├── server_eureka/       # Serveur Eureka (Discovery)
│   └── pom.xml
└── deploy/              # Configuration Docker Compose
    └── docker-compose.yml
```

> 💡 **Astuce** : Repérer dès maintenant le dossier `deploy/` : c'est lui qui sera utilisé par Jenkins au stage Docker Compose.

---

## Étape 2 : Installation et configuration de Jenkins

### Introduction

Installer Jenkins localement, puis préparer l'environnement d'exécution (JDK + Maven) côté Jenkins.

### 2.1 Installer Jenkins

1. Télécharger Jenkins depuis [jenkins.io](https://www.jenkins.io/download/)
2. Lancer l'installateur Windows
3. Suivre l'assistant de configuration

![Installation Jenkins](images/jenkins-install.png)

### 2.2 Choisir le type de service "LocalSystem"

Pendant l'installation, sélectionner **LocalSystem** comme type de connexion initiale.

![Service LocalSystem](images/jenkins-localsystem.png)

> ⚠️ Ce choix lance Jenkins en tant que service Windows. Si un compte local/domaine est sélectionné par erreur, des problèmes de permissions peuvent apparaître.

### 2.3 Choisir et tester le port Jenkins

1. Spécifier un port (par défaut : **8080**)
2. Cliquer sur **Tester le port**
3. Cliquer sur **Next**

![Test du port](images/jenkins-port.png)

> 💡 Si le port 8080 est déjà utilisé, choisir un autre port (ex. 8081) et noter la nouvelle URL Jenkins.

### 2.4 Indiquer le chemin du JDK

Renseigner le chemin local du JDK (ex. JDK 17) :

```
C:\Program Files\Java\jdk-17
```

![Configuration JDK](images/jenkins-jdk.png)

> ⚠️ Jenkins doit connaître un JDK valide pour exécuter Maven/compilation. Un chemin incorrect provoque des erreurs `Java not found` ou `JAVA_HOME`.

### 2.5 Configurer Maven dans Jenkins

1. Aller dans **Administrer Jenkins** → **Tools**
2. Section **Maven installations**
3. Ajouter une installation Maven :
   - **Nom** : `maven` (IMPORTANT : ce nom exact)
   - **Chemin** : `C:\Program Files\Apache\maven` (ou installation automatique)

![Configuration Maven](images/jenkins-maven.png)

> ⚠️ **Important** : Donner exactement le nom `maven` à l'installation Maven pour correspondre au script du pipeline (`tools { maven 'maven' }`).

---

## Étape 3 : Installation et configuration de SonarQube

### Introduction

Déployer SonarQube avec Docker Compose, puis créer un projet + token pour chaque microservice.

### 3.1 Créer le fichier docker-compose.yml pour SonarQube

Créer le fichier `sonarqube-compose.yml` :

```yaml
version: '3.9'
services:
  sonarqube:
    image: sonarqube:latest
    ports:
      - "9999:9000"
    environment:
      - SONARQUBE_JDBC_URL=jdbc:postgresql://sonarqube-db:5432/sonarqube
      - SONARQUBE_JDBC_USERNAME=sonar
      - SONARQUBE_JDBC_PASSWORD=sonar_pass

  sonarqube-db:
    image: postgres:latest
    environment:
      - POSTGRES_DB=sonarqube
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  postgres_data:
```

> 💡 `9999:9000` signifie : accès SonarQube via `http://localhost:9999`

### 3.2 Démarrer SonarQube

```bash
docker compose -f sonarqube-compose.yml up -d
docker ps
```

**Résultat attendu** : Deux conteneurs démarrés (SonarQube + PostgreSQL).

### 3.3 Configurer SonarScanner dans Jenkins

1. Aller dans **Administrer Jenkins** → **Tools**
2. Section **SonarQube Scanner installations**
3. Ajouter SonarQube Scanner

![SonarScanner Jenkins](images/sonarscanner-jenkins.png)

### 3.4 Créer les projets SonarQube et générer les tokens

Dans SonarQube (`http://localhost:9999`) :

1. **Connexion** : admin / admin (changer le mot de passe)
2. **Créer projet "car"** :
   - Projects → Create Project → Manually
   - Project key : `car`
   - Display name : `car`
3. **Créer projet "client"** :
   - Project key : `client`
   - Display name : `client`
4. **Générer les tokens** :
   - My Account → Security → Generate Tokens
   - Token pour `car` : noter la valeur
   - Token pour `client` : noter la valeur

### 3.5 Déclarer les serveurs SonarQube dans Jenkins

1. Aller dans **Administrer Jenkins** → **System**
2. Section **SonarQube servers**
3. Ajouter deux serveurs :

| Nom | URL | Token |
|-----|-----|-------|
| `SonarQube-Car` | `http://localhost:9999` | Token du projet car |
| `SonarQube-Client` | `http://localhost:9999` | Token du projet client |

![SonarQube Servers](images/sonarqube-servers.png)

> ⚠️ Les noms `SonarQube-Car` et `SonarQube-Client` doivent correspondre exactement au pipeline (`withSonarQubeEnv('SonarQube-Car')`).

---

## Étape 4 : Exposition de Jenkins via Ngrok et Webhooks GitHub

### Introduction

Jenkins local doit recevoir des notifications GitHub. Ngrok fournit une URL publique temporaire vers le port Jenkins.

### 4.1 Installer Ngrok et associer un authtoken

1. Créer un compte sur [ngrok.com](https://ngrok.com/)
2. Récupérer votre authtoken
3. Configurer Ngrok :

```bash
ngrok config add-authtoken <votre_token>
```

![Ngrok Config](images/ngrok-config.png)

### 4.2 Lancer un tunnel HTTP vers Jenkins

```bash
ngrok http http://localhost:8080
```

![Ngrok Tunnel](images/ngrok-tunnel.png)

### 4.3 Copier l'URL publique Ngrok

Repérer l'URL du type :
```
https://xxxx.ngrok-free.app
```

![Ngrok URL](images/ngrok-url.png)

> ⚠️ Cette URL change si Ngrok est relancé (plan gratuit). Mettre à jour le webhook GitHub si l'URL change.

### 4.4 Configurer GitHub dans Jenkins

1. Aller dans **Administrer Jenkins** → **System**
2. Section **GitHub**
3. Ajouter l'URL du dépôt GitHub

![GitHub Jenkins](images/github-jenkins.png)

### 4.5 Créer un Webhook GitHub

Dans GitHub :

1. Ouvrir le dépôt → **Settings** → **Webhooks**
2. Cliquer **Add webhook**
3. Configurer :

| Champ | Valeur |
|-------|--------|
| Payload URL | `https://<URL_NGROK>/github-webhook/` |
| Content type | `application/json` |
| Which events | Just the push event |

4. Activer le webhook

![GitHub Webhook](images/github-webhook.png)

> 💡 **Astuce** : Dans GitHub Webhooks → onglet "Recent Deliveries", vérifier un code **200** après un push.

---

## Étape 5 : Création du Job Pipeline Jenkins

### Introduction

Créer un job de type Pipeline, le relier au dépôt GitHub et activer le trigger webhook.

### 5.1 Créer un nouveau job "Pipeline"

1. Dans Jenkins : **Tableau de bord** → **Nouveau Item**
2. Nom : `cicd-microservices`
3. Type : **Pipeline**
4. Cliquer **OK**

![Nouveau Job](images/jenkins-new-job.png)

### 5.2 Configurer le trigger GitHub

Dans la configuration du job :

1. Cocher **GitHub project**
2. Coller l'URL du dépôt : `https://github.com/lachgar/jenkins2`
3. Dans **Build Triggers**, cocher **GitHub hook trigger for GITScm polling**

![Trigger GitHub](images/jenkins-trigger.png)

---

## Étape 6 : Détail du script de pipeline Jenkins

### Introduction

Saisir le pipeline qui enchaîne : clonage → build + analyse SonarQube (en parallèle) → déploiement Docker Compose.

### 6.1 Ajouter le pipeline script

Dans la configuration du job :
1. Section **Pipeline** → **Definition** = `Pipeline script`
2. Coller le script ci-dessous

### 6.2 Script de pipeline complet

```groovy
pipeline {
    agent any

    tools {
        maven 'maven'
    }

    stages {

        stage('Cloner le dépôt') {
            steps {
                echo 'Clonage du dépôt GitHub...'
                git branch: 'main', url: 'https://github.com/lachgar/jenkins2.git'
            }
        }

        stage('Build and SonarQube Analysis') {
            parallel {

                stage('Car Service') {
                    stages {

                        stage('Build Car Service') {
                            steps {
                                dir('car') {
                                    echo 'Compilation et génération du service Car...'
                                    script {
                                        bat 'mvn clean install -DskipTests'
                                    }
                                }
                            }
                        }

                        stage('SonarQube Analysis Car Service') {
                            steps {
                                dir('car') {
                                    script {
                                        def mvn = tool 'maven';
                                        withSonarQubeEnv('SonarQube-Car') {
                                            bat "${mvn}\\bin\\mvn clean verify ^ " +
                                                "sonar:sonar ^ " +
                                                "-Dsonar.projectKey=car ^ " +
                                                "-Dsonar.projectName='car' ^ " +
                                                "-DskipTests"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }

                stage('Client Service') {
                    stages {

                        stage('Build Client Service') {
                            steps {
                                dir('client') {
                                    echo 'Compilation et génération du service Client...'
                                    script {
                                        bat 'mvn clean install -DskipTests'
                                    }
                                }
                            }
                        }

                        stage('SonarQube Analysis Client Service') {
                            steps {
                                dir('client') {
                                    script {
                                        def mvn = tool 'maven';
                                        withSonarQubeEnv('SonarQube-Client') {
                                            bat "${mvn}\\bin\\mvn clean verify ^ " +
                                                "sonar:sonar ^ " +
                                                "-Dsonar.projectKey=client ^ " +
                                                "-Dsonar.projectName='client' ^ " +
                                                "-DskipTests"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }

                stage('Gateway Service') {
                    steps {
                        dir('gateway') {
                            echo 'Compilation et génération du service Gateway...'
                            script {
                                bat 'mvn clean install -DskipTests'
                            }
                        }
                    }
                }

                stage('Eureka Server') {
                    steps {
                        dir('server_eureka') {
                            echo 'Compilation et génération du serveur Eureka...'
                            script {
                                bat 'mvn clean install -DskipTests'
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Compose') {
            steps {
                dir('deploy') {
                    echo 'Création et déploiement des conteneurs Docker...'
                    script {
                        bat 'docker-compose up -d --build'
                    }
                }
            }
        }
    }
}
```

### Explication pédagogique du script

| Élément | Description |
|---------|-------------|
| `tools { maven 'maven' }` | Utilise l'installation Maven déclarée dans Jenkins (Étape 2.5) |
| `stage('Cloner le dépôt')` | Récupère la branche `main` du dépôt GitHub |
| `parallel { ... }` | Exécute plusieurs builds/analyses en même temps pour gagner du temps |
| `withSonarQubeEnv('SonarQube-Car')` | Injecte l'URL + token SonarQube configurés (Étape 3.5) |
| `docker-compose up -d --build` | Rebuild et redémarre les services conteneurisés |

> 📝 **Note** : Le pipeline compile `gateway` et `eureka` mais n'exécute pas d'analyse SonarQube pour ces services. Pour ajouter l'analyse, dupliquer le modèle "car/client".

---

## Étape 7 : Exécution du pipeline et vérifications

### 7.1 Lancer un build manuel

1. Dans Jenkins : ouvrir le job
2. Cliquer **Build Now**

**Résultat attendu** : Une exécution apparaît dans l'historique.

### 7.2 Vérifier le résultat dans Jenkins

Ouvrir **Console Output** et contrôler :

- ✅ Stage clonage : checkout main
- ✅ Builds Maven : succès sur car/client/gateway/server_eureka
- ✅ SonarQube : exécution `sonar:sonar` sur car et client
- ✅ Docker Compose : `up -d --build` exécuté

### 7.3 Vérifier les tableaux de bord SonarQube

1. Aller sur SonarQube : `http://localhost:9999`
2. Ouvrir projet **car** → vérifier qu'une analyse récente existe
3. Ouvrir projet **client** → vérifier idem

**Métriques attendues** : bugs, vulnérabilités, code smells, "Last analysis" récent

### 7.4 Vérifier le déploiement Docker Compose

```bash
docker ps
```

**Résultat attendu** : Conteneurs démarrés pour les microservices.

```bash
# Test optionnel des endpoints
curl http://localhost:<port_gateway>/actuator/health
curl http://localhost:<port_car>/actuator/health
```

### 7.5 Tester le déclenchement automatique

```bash
git add README.md
git commit -m "test: déclenchement webhook"
git push
```

**Résultat attendu** : Jenkins démarre automatiquement une nouvelle exécution après le push.

---

## 🔧 Dépannage rapide

| Problème | Solution |
|----------|----------|
| Jenkins ne se lance pas | Port occupé → changer le port (Étape 2.3) |
| SonarQube inaccessible | Vérifier `docker ps` et le port 9999 |
| Analyse SonarQube échoue | Nom `withSonarQubeEnv('...')` ≠ nom déclaré dans Jenkins System |
| Webhook GitHub "failed" | URL Ngrok changée → mettre à jour Payload URL |
| Docker Compose échoue | Jenkins n'a pas accès au daemon Docker (droits/service) |

---

## 📊 Schéma récapitulatif du flux CI/CD

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            FLUX CI/CD COMPLET                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   [Developer]                                                            │
│       │                                                                  │
│       │ git push                                                         │
│       ▼                                                                  │
│   [GitHub]                                                               │
│       │                                                                  │
│       │ Webhook POST                                                     │
│       ▼                                                                  │
│   [Ngrok] ──────────────────► [Jenkins]                                 │
│                                   │                                      │
│                    ┌──────────────┼──────────────┐                      │
│                    │              │              │                      │
│                    ▼              ▼              ▼                      │
│              [Clone Repo]   [Build Maven]  [SonarQube]                  │
│                                   │              │                      │
│                                   └──────┬───────┘                      │
│                                          │                              │
│                                          ▼                              │
│                                 [Docker Compose]                        │
│                                          │                              │
│                    ┌─────────┬───────┬───┴───┬─────────┐               │
│                    ▼         ▼       ▼       ▼         ▼               │
│                 [Car]   [Client] [Gateway] [Eureka]                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Checklist finale

- [ ] Jenkins installé et accessible
- [ ] JDK 17 configuré dans Jenkins
- [ ] Maven configuré avec le nom `maven`
- [ ] SonarQube déployé via Docker Compose
- [ ] Projets `car` et `client` créés dans SonarQube
- [ ] Tokens générés et configurés dans Jenkins
- [ ] Ngrok tunnel actif
- [ ] Webhook GitHub configuré et fonctionnel
- [ ] Job Pipeline créé avec le script complet
- [ ] Build manuel réussi
- [ ] Déclenchement automatique via push testé

---

