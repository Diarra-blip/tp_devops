# Rapport de TP - Docker & Application 3-Tiers

## 1-1. Justification de l'utilisation du flag `-e` pour les variables d'environnement
L'utilisation du flag `-e` (ou d'un fichier de configuration externe) plutôt que l'écriture directe dans le `Dockerfile` répond à un impératif de sécurité. Cela évite d'inscrire des informations sensibles (telles que le mot de passe de la base de données `POSTGRES_PASSWORD`) en clair dans le code source et dans l'historique des couches de l'image Docker.

## 1-2. Nécessité d'un volume pour le conteneur PostgreSQL
Les conteneurs étant par nature éphémères, la suppression ou le redémarrage du conteneur de la base de données entraînerait la perte irrémédiable des données. L'attachement d'un volume Docker permet de garantir la persistance des données directement sur le disque de la machine hôte.

## 1-3. Documentation du conteneur de base de données
* **Dockerfile** : Basé sur l'image minimale `postgres:17.2-alpine`.
* **Mise en œuvre** : Création préalable d'un réseau dédié (`app-network`), injection des variables d'environnement à l'exécution, et montage d'un volume pour la persistance des données stockées dans `/var/lib/postgresql/data`. Les scripts SQL d'initialisation sont placés dans le répertoire `/docker-entrypoint-initdb.d` pour s'exécuter automatiquement au premier lancement.

## 1-4. Utilité du build multi-stage et description du Dockerfile
Le build multi-stage permet de dissocier l'environnement de compilation (lourd, incluant le JDK et Maven) de l'environnement d'exécution (léger, ne contenant que le JRE).
* **Étape de build (`myapp-build`)** : Utilise une image JDK, récupère le code source et le fichier `pom.xml`, puis exécute la commande Maven pour compiler l'application et générer le fichier `.jar`.
* **Étape d'exécution (Run stage)** : Utilise une image JRE minimale, y copie uniquement le fichier `.jar` compilé depuis l'étape précédente, et définit l'instruction de démarrage. Cela réduit considérablement la taille de l'image finale et renforce la sécurité.

## 1-5. Rôle du reverse proxy
Le serveur HTTP (configuré ici avec Apache) agit comme un point d'entrée unique sur le port `80`. Il permet de masquer l'architecture interne des conteneurs aux utilisateurs, de centraliser les flux web et de rediriger de manière transparente les requêtes HTTP vers l'API backend.

## 1-6. Importance de `docker-compose`
L'outil `docker-compose` simplifie l'orchestration de l'application multi-conteneurs. Il permet de définir, lier, démarrer et arrêter l'ensemble des services de l'application 3-tiers à l'aide d'un unique fichier de configuration déclaratif, évitant la gestion manuelle et fastidieuse de chaque conteneur.

## 1-7. Commandes essentielles de `docker-compose`
* `docker-compose up -d --build` : Construit les images si nécessaire et lance tous les services en arrière-plan.
* `docker-compose ps` : Affiche l'état des conteneurs du projet en cours.
* `docker-compose logs` : Permet de consulter les journaux de bord de l'ensemble des services.
* `docker-compose down` : Arrête et supprime les conteneurs ainsi que le réseau associé.

## 1-8. Structure du fichier `docker-compose.yml`
Le fichier de configuration structure l'application autour de trois services interconnectés :
1. `database` : conteneur PostgreSQL associé à un volume de persistance.
2. `backend` : API Spring Boot construite via son Dockerfile.
3. `httpd` : serveur web Apache configuré en reverse proxy et exposé sur le port `80`.
Les services communiquent via un réseau bridge personnalisé (`app-network`) et l'ordre de démarrage est géré par la directive `depends_on`.

## 1-9. Commandes de publication sur Docker Hub
* **Connexion** : `docker login`
* **Étiquetage de l'image** : `docker tag mon-image nom_utilisateur/mon-image:1.0`
* **Publication** : `docker push nom_utilisateur/mon-image:1.0`

## 1-10. Intérêt d'un dépôt en ligne pour les images
Stocker les images sur un registre en ligne (comme Docker Hub) facilite le partage au sein d'une équipe, permet un déploiement rapide sur d'autres environnements (serveurs de test, cloud ou production) et assure le suivi versionné des différentes versions de l'application.

This project is now mine

---

# TD2 / TP2 : GitHub Actions (CI/CD)

## TD2 : prise en main de Git/GitHub

- Création d'une clé SSH (`ssh-keygen -t rsa -b 4096`) et ajout de la clé publique dans GitHub (Settings > SSH and GPG keys), pour cloner et pousser sans mot de passe.
- Clone du dépôt en SSH : `git clone git@github.com:Diarra-blip/tp_devops.git`.
- Cycle de base : `git add .`, `git commit -m "..."`, `git push origin main`. La branche par défaut s'appelle `main` (et non `master` comme dans le sujet).
- Règle de protection de branche sur `main` (Settings > Branches), pour empêcher la suppression de la branche et les `push --force`.

## TP2 : pipelines CI/CD

Les pipelines sont dans `.github/workflows/` et séparés en deux fichiers.

### `ci.yml` : intégration continue (CI)

Déclenché sur `push` (branches `main` et `develop`) et sur `pull_request`. Job `test-backend` :

1. `actions/checkout@v4` : récupère le code (avec `fetch-depth: 0`, historique complet requis par Sonar).
2. `actions/setup-java@v4` : installe le JDK 21 (Temurin).
3. `mvn clean verify --file ./simple-api/pom.xml` : nettoie, compile, lance les tests unitaires et d'intégration.
4. Analyse SonarCloud avec `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` (clés de projet et d'organisation passées en paramètres, token via le secret `SONAR_TOKEN`).

Le raccourci `sonar:sonar` du sujet ne fonctionnait pas sur ce projet (erreur `No plugin found for prefix 'sonar'`) : j'ai utilisé le nom complet du plugin.

### `cd.yml` : livraison continue (CD)

Déclenché avec `on: workflow_run` quand le workflow `CI devops 2025` se termine sur `main`. Le job `build-and-push-docker-image` ne s'exécute que si le CI a réussi (`conclusion == 'success'`) :

1. Checkout du commit exact qui a passé les tests (`head_sha`).
2. Connexion à Docker Hub avec les secrets `DOCKERHUB_USERNAME` et `DOCKERHUB_TOKEN`.
3. Build et push des 3 images : `tp-devops-simple-api`, `tp-devops-database`, `tp-devops-httpd`.

### Secrets utilisés (Settings > Secrets and variables > Actions)

- `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`
- `SONAR_TOKEN`

## Réponses aux questions

**2-1 Que sont les testcontainers ?**
Ce sont des bibliothèques Java qui lancent des conteneurs Docker temporaires pendant les tests. Ici, un conteneur PostgreSQL est démarré pour les tests d'intégration, puis supprimé à la fin. Les tests tournent ainsi sur une vraie base de données, sans installation manuelle.

**2-2 Pourquoi utiliser des variables sécurisées (secrets) ?**
Pour ne jamais écrire d'identifiants (Docker Hub, SonarCloud) en clair dans le code, surtout sur un dépôt public. Les secrets sont chiffrés par GitHub et masqués dans les logs (affichés en `***`).

**2-3 Pourquoi `needs: test-backend` sur le job de build/push ?**
Sans `needs`, les jobs s'exécutent en parallèle : on pourrait publier une image alors que les tests échouent. Avec `needs` (puis `workflow_run` après la séparation des pipelines), l'image n'est construite et publiée que si le code compile et que les tests passent.

**2-4 Pourquoi pousser les images Docker ?**
Pour les stocker et les versionner dans un registre (Docker Hub), afin que les autres membres de l'équipe, d'autres machines ou des serveurs de déploiement puissent les récupérer avec `docker pull`. C'est la base de la livraison continue.

## Résultats

- CI : le workflow `CI devops 2025` passe au vert (onglet Actions).
- Quality Gate SonarCloud : analyse effectuée à chaque push (projet `Diarra-blip_tp_devops`).
- CD : le workflow `CD devops 2025` se déclenche après le succès du CI et publie les 3 images sur Docker Hub.
