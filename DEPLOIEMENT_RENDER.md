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

La base n'est pas obligatoirement hébergée sur Render. Deux options, comparées ici
sur les seuls plans gratuits.

| Critère | Render Postgres | Supabase |
|---|---|---|
| Durée de vie | expire 30 jours après création, puis 14 jours de grâce avant suppression des données | pas d'expiration, le projet est seulement mis en pause après une période d'inactivité |
| Réactivation | impossible sans passer à un plan payant | manuelle depuis le tableau de bord, sans perte de données |
| Stockage | 1 Go | 500 Mo |
| Sauvegardes | aucune sur le plan gratuit | aucune sur le plan gratuit |
| Nombre | une seule base par espace de travail | deux projets |
| Latence depuis Render | réseau privé interne, très faible | via Internet, plus élevée |

**Supabase est recommandé ici.** La suppression automatique au bout de trente jours
côté Render ferait perdre la base en pleine année scolaire, alors qu'une pause
Supabase se rattrape en un clic.

### Option retenue : Supabase

1. Aller sur `supabase.com/dashboard` et se connecter, le compte GitHub suffit.
2. `New project`, puis choisir l'organisation.
3. Nommer le projet, par exemple `mutuelle-mobile`.
4. Générer le mot de passe de la base et **le conserver immédiatement**. Il n'est
   plus affiché ensuite, et il faut le réinitialiser dans
   `Project Settings` puis `Database` en cas de perte.
5. Choisir la région la plus proche, `Central EU (Frankfurt)` depuis le Cameroun.
6. Créer, puis attendre la fin du provisionnement.
7. Cliquer sur `Connect` en haut de la page du projet, et choisir **`Session pooler`**.

Le choix du mode de connexion est déterminant. La connexion directe de Supabase
n'est joignable qu'en IPv6 sur le plan gratuit, or Render sort en IPv4, donc elle
échouerait. Le pooler partagé est en IPv4 sur tous les plans. Il faut le **mode
session** et non le mode transaction, car ce dernier ne gère pas les requêtes
préparées, dont Hibernate se sert.

La chaîne affichée a cette forme :

```
postgresql://postgres.[PROJECT-REF]:[MOT-DE-PASSE]@aws-[INDEX]-[REGION].pooler.supabase.com:5432/postgres
```

L'hôte doit être copié depuis la boîte de dialogue `Connect` et ne peut pas être
reconstitué à partir de la région. L'indice qui le précède est un numéro de grappe
de pooler, et une même région peut en compter plusieurs.

Elle se découpe ainsi dans les variables attendues par l'application :

| Variable | Valeur à relever |
|---|---|
| `PGHOST` | l'hôte du pooler |
| `PGPORT` | `5432` |
| `PGDATABASE` | `postgres` |
| `PGUSER` | `postgres.[PROJECT-REF]` |
| `PGPASSWORD` | le mot de passe de la base |

Aucune modification du code n'est nécessaire, `application-prod.properties`
reconstruisant déjà l'URL JDBC à partir de ces cinq variables. Le pilote
PostgreSQL négocie TLS de lui-même, ce que Supabase exige.

### Option alternative : Render Postgres

1. Dans le tableau de bord Render, `New +` puis `Postgres`, ou directement
   `dashboard.render.com/new/database`.
2. Renseigner le nom, par exemple `mutuelle-db`.
3. Choisir la région la plus proche, `Frankfurt (EU Central)` depuis le Cameroun.
4. Plan `Free`.
5. Créer, puis relever les identifiants dans le menu `Connect` en haut à droite de
   la page de la base.

Prendre le **hostname interne**, celui sans suffixe public, et non l'externe. La
connexion interne est plus rapide et dispense de TLS. Elle n'est utilisable que si
le service web est créé dans la **même région** que la base.

## 3. Créer le service web

1. `New +` puis `Web Service`.
2. Connecter le compte GitHub et autoriser l'accès au dépôt `mutuelle-back`.
3. Sélectionner `steve20400/mutuelle-back`, branche `main`.
4. `Language` ou `Runtime` doit afficher `Docker`. Laisser le répertoire racine vide.
5. Région : `Frankfurt (EU Central)`. Si la base est sur Render, cette région doit
   impérativement être la même que la sienne, sinon le hostname interne ne répond pas.
6. Plan `Free`.
7. Dans `Advanced`, renseigner `Health Check Path` avec `/actuator/health`.
   Ce chemin est déjà public dans `SecurityConfig`, donc il répond sans jeton.

## 4. Variables d'environnement

À saisir dans `Environment` avant le premier déploiement.

| Variable | Valeur |
|---|---|
| `SPRING_PROFILES_ACTIVE` | `prod` |
| `PGHOST` | hôte du pooler Supabase, ou hostname interne si la base est sur Render |
| `PGPORT` | `5432` |
| `PGDATABASE` | `postgres` sur Supabase, sinon le nom de la base |
| `PGUSER` | `postgres.[PROJECT-REF]` sur Supabase, sinon l'utilisateur de la base |
| `PGPASSWORD` | mot de passe de la base |
| `JWT_SECRET` | chaîne aléatoire longue, au moins 32 caractères |
| `JWT_ACCESS_EXPIRATION` | `9000000000` |
| `JWT_REFRESH_EXPIRATION` | `604800000` |
| `SPRING_MAIL_USERNAME` | adresse Gmail d'envoi |
| `SPRING_MAIL_PASSWORD` | mot de passe d'application Gmail |
| `JAVA_OPTS` | `-Xms128m -Xmx400m` |
| `MANAGEMENT_HEALTH_MAIL_ENABLED` | `false` |

La dernière variable n'est pas optionnelle. Le projet déclare
`spring-boot-starter-mail`, ce qui pousse l'actuateur à ajouter un indicateur de
santé SMTP. Cet indicateur ouvre une connexion vers Gmail à chaque appel de
`/actuator/health`. S'il échoue, pour un mot de passe d'application erroné ou un
refus de Gmail, l'état global passe à `DOWN` et le point de santé répond `503`
alors que l'application fonctionne parfaitement.

Render interpréterait ce `503` comme un service en panne et le redémarrerait en
boucle. Cette mise en échec a été observée sur l'intégration continue avant tout
déploiement. La désactivation ne touche que la surveillance, l'envoi de courriels
par l'application reste inchangé.

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
sur le plan gratuit.

### Aucune migration à exécuter

Le projet ne déclare ni Flyway ni Liquibase. Le schéma est donc créé par Hibernate
au premier démarrage, `spring.jpa.hibernate.ddl-auto=update` étant actif en profil
`prod`. Il n'y a aucune étape de migration manuelle avant le déploiement.

Les quelques `columnDefinition` des entités sont compatibles PostgreSQL, et les
identifiants en `GenerationType.IDENTITY` sont traduits en colonnes d'identité.

### Ne pas activer l'initialisation SQL

`src/main/resources/data.sql` est écrit pour MySQL : il utilise `INSERT IGNORE` et
`FROM dual`, deux constructions qui n'existent pas en PostgreSQL. Ce fichier ne
s'exécute pas, car Spring Boot réserve `data.sql` aux bases embarquées tant que
`spring.sql.init.mode` n'est pas défini.

Il ne faut donc surtout pas ajouter `spring.sql.init.mode=always`, qui ferait
échouer le démarrage sur PostgreSQL.

### Comptes créés automatiquement

`config/DataInitializer.java` est un `CommandLineRunner` actif dans tous les profils.
Il crée au premier démarrage, si les adresses n'existent pas déjà, quatre comptes
partageant le mot de passe `admin123` :

| Adresse | Rôle |
|---|---|
| `superadmin@mutuelle.com` | super administrateur |
| `admin@mutuelle.com` | administrateur |
| `president@mutuelle.com` | président |
| `tresorier@mutuelle.com` | trésorier |

L'opération est idempotente, les redémarrages ne créent pas de doublons. Aucun
endpoint public d'inscription n'existe, `AuthController` n'exposant que `/auth/login`,
`/auth/refresh` et `/auth/logout`, donc ces comptes sont le seul point d'entrée.

Sur un déploiement joignable publiquement, ces identifiants par défaut sont connus
de quiconque lit ce dépôt. Les changer avant toute mise en service réelle.

Vérifications une fois le service en ligne :

```bash
curl https://mutuelle-back.onrender.com/actuator/health
```

La documentation interactive de l'API est accessible sur `/swagger-ui/index.html`.

## 6. Points de vigilance du plan gratuit

- Le service s'endort après quinze minutes sans trafic. Le réveil prend environ
  cinquante secondes, ce qui se voit sur le premier appel depuis l'application.
- Les sept cent cinquante heures d'instance gratuites par mois sont partagées par
  tout l'espace de travail. Au-delà, les services gratuits sont suspendus jusqu'au
  mois suivant.
- Si la base est restée sur Render, elle expire trente jours après sa création, avec
  quatorze jours de grâce avant suppression définitive des données. Aucune sauvegarde
  n'est fournie sur ce plan, donc prévoir un `pg_dump` régulier.
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
