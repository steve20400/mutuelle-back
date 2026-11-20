# Déploiement du backend Mutuelle Mobile sur Render

Procédure de déploiement de l'API sur Render, en complément du guide GANDAL
(`Guide_Deploiement_MutueleMobile_GANDAL.docx`) qui couvre le datacenter de l'ENSPY.

## 1. Rappel de la pile technique

| Élément | Valeur |
|---|---|
| Langage | Java |
| Version du code (`pom.xml`) | Java 17 |
| Version de compilation et d'exécution (`Dockerfile`) | Java 21 (Temurin) |
| Framework | Spring Boot 3.3.4 |
| Build | Maven (`mvnw` fourni) |
| Base de données | PostgreSQL (driver 42.7.4) |
| Sécurité | Spring Security + JWT (jjwt 0.12.6) |
| Conteneurisation | Dockerfile multi-étapes à la racine du dépôt |

Le code cible Java 17 alors que l'image Docker fournit Java 21. C'est compatible et
Render ne s'en plaint pas, puisque le `Dockerfile` pilote tout le build.

Dépôt déployé : `https://github.com/steve20400/mutuelle-back` (branche `main`).
Le `Dockerfile` est à la racine, donc Render détecte seul l'environnement Docker.

## 2. Créer la base PostgreSQL

1. Dans le tableau de bord Render, `New +` puis `Postgres`.
2. Renseigner le nom, par exemple `mutuelle-db`.
3. Choisir la région la plus proche, `Frankfurt (EU Central)` depuis le Cameroun.
4. Plan `Free`.
5. Créer, puis ouvrir la page de la base et relever dans `Connections` :
   `Hostname`, `Port`, `Database`, `Username`, `Password`.

Prendre le **hostname interne**, celui sans suffixe public, et non l'externe. La
connexion interne est plus rapide et dispense de TLS. Elle n'est utilisable que si
le service web est créé dans la **même région** que la base.

## 3. Créer le service web

1. `New +` puis `Web Service`.
2. Connecter le compte GitHub et autoriser l'accès au dépôt `mutuelle-back`.
3. Sélectionner `steve20400/mutuelle-back`, branche `main`.
4. `Language` ou `Runtime` doit afficher `Docker`. Laisser le répertoire racine vide.
5. Région : **la même que la base de données**.
6. Plan `Free`.
7. Dans `Advanced`, renseigner `Health Check Path` avec `/actuator/health`.
   Ce chemin est déjà public dans `SecurityConfig`, donc il répond sans jeton.

## 4. Variables d'environnement

À saisir dans `Environment` avant le premier déploiement.

| Variable | Valeur |
|---|---|
| `SPRING_PROFILES_ACTIVE` | `prod` |
| `PGHOST` | hostname interne de la base |
| `PGPORT` | `5432` |
| `PGDATABASE` | nom de la base |
| `PGUSER` | utilisateur de la base |
| `PGPASSWORD` | mot de passe de la base |
| `JWT_SECRET` | chaîne aléatoire longue, au moins 32 caractères |
| `JWT_ACCESS_EXPIRATION` | `9000000000` |
| `JWT_REFRESH_EXPIRATION` | `604800000` |
| `SPRING_MAIL_USERNAME` | adresse Gmail d'envoi |
| `SPRING_MAIL_PASSWORD` | mot de passe d'application Gmail |
| `JAVA_OPTS` | `-Xms128m -Xmx400m` |

Ne pas définir `PORT`. Render l'injecte lui-même et
`application-prod.properties` le lit déjà via `server.port=${PORT:8080}`.

`JAVA_OPTS` doit être abaissé par rapport au `Dockerfile`, qui demande `-Xmx512m`.
Le plan gratuit ne dispose que de 512 Mo au total, donc la JVM et sa surcharge
dépasseraient la limite et le conteneur serait tué pour dépassement mémoire.

Générer le secret JWT avec :

```bash
openssl rand -base64 48
```

## 5. Premier déploiement

Lancer `Create Web Service`. Le build Maven dans Docker prend plusieurs minutes
sur le plan gratuit. Les tables sont créées automatiquement au premier démarrage,
`spring.jpa.hibernate.ddl-auto=update` étant actif en profil `prod`.

Vérifications une fois le service en ligne :

```bash
curl https://mutuelle-back.onrender.com/actuator/health
```

La documentation interactive de l'API est accessible sur `/swagger-ui/index.html`.

## 6. Points de vigilance du plan gratuit

- Le service s'endort après quinze minutes sans trafic. Le réveil prend environ
  cinquante secondes, ce qui se voit sur le premier appel depuis l'application.
- Les bases PostgreSQL gratuites de Render sont supprimées au bout de trente jours.
  Prévoir une sauvegarde par `pg_dump` ou le passage à un plan payant avant l'échéance.
- Les journaux sont dans l'onglet `Logs`. En cas d'arrêt immédiat, chercher
  `OutOfMemoryError` ou un échec de connexion à la base.

## 7. Autoriser le frontend

`SecurityConfig` fixe les origines CORS à `http://localhost:3000` et
`http://localhost:8081`. Une fois le backend en ligne, toute requête venant d'une
autre origine sera refusée par le navigateur.

Il faut donc ajouter l'origine de déploiement du frontend dans
`src/main/java/com/mutuelle/mobille/config/security/SecurityConfig.java`, à l'appel
`configuration.setAllowedOrigins(...)`. Cette liste gagnerait à être lue depuis une
variable d'environnement pour éviter de recompiler à chaque changement d'URL.

Côté application mobile, pointer `EXPO_PUBLIC_API_URL` vers l'URL Render du backend.
