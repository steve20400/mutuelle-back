# Mutuelle Mobile — Backend

API Spring Boot (Java 17, Maven) du projet Mutuelle Mobile.

## Lancer en local

Copier `.env.example` en `.env` et renseigner les valeurs, puis :

```bash
mvn spring-boot:run "-Dspring.profiles.active=dev"
```

Ou via Docker (voir le `docker-compose.yml` à la racine du repo pour lancer back + front + DB ensemble) :

```bash
docker compose up --build
```

## Déploiement (profil prod)

Le profil `prod` (`application-prod.properties`) lit toute sa config depuis des variables
d'environnement (voir `.env.example`) : connexion PostgreSQL, JWT, SMTP. Ne jamais committer
de vraies valeurs — seules les clés doivent apparaître dans `.env.example`.

```bash
docker compose up --build -d
```

### Provisionner la base PostgreSQL sur le serveur

```bash
sudo -u postgres psql <<EOF
CREATE USER mutuelle_user WITH PASSWORD 'CHANGE_ME';
CREATE DATABASE mutuelle_mobile OWNER mutuelle_user;
GRANT ALL PRIVILEGES ON DATABASE mutuelle_mobile TO mutuelle_user;
EOF
```

### Autoriser les connexions réseau à PostgreSQL

```bash
PG_VERSION=$(ls /etc/postgresql/)
PG_HBA="/etc/postgresql/$PG_VERSION/main/pg_hba.conf"
PG_CONF="/etc/postgresql/$PG_VERSION/main/postgresql.conf"

sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" $PG_CONF
echo "host    mutuelle_mobile    mutuelle_user    10.50.30.0/24    md5" >> $PG_HBA

systemctl restart postgresql
```

### Installer Docker (Debian)

```bash
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Voir aussi `Guide_Deploiement_MutueleMobile_GANDAL.docx` pour le guide de déploiement détaillé.
