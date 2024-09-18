# Environnement de Développement Docker avec Traefik, MariaDB et deux sites web

Ce projet configure un environnement de développement utilisant Docker pour héberger deux sites web communiquant avec une base de données MariaDB, le tout géré par Traefik comme reverse proxy.

## Prérequis

- Une VM (Machine Virtuelle) sous Rocky Linux 8
- Docker et Docker Compose installés sur la VM

## Structure du Projet

```
.
├── docker-compose.yml
├── site1/
│   ├── Dockerfile
│   └── index.php
├── site2/
│   ├── Dockerfile
│   └── index.php
└── README.md
```

## Configuration

### 1. Docker Compose

Le fichier `docker-compose.yml` à la racine du projet définit tous les services nécessaires :

```yaml
version: '3'

services:
  traefik:
    image: traefik:v2.5
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  mariadb:
    image: mariadb:10.5
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydb
      MYSQL_USER: user
      MYSQL_PASSWORD: password

  site1:
    build: ./site1
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.site1.rule=Host(`mysite1.lan`)"
    depends_on:
      - mariadb

  site2:
    build: ./site2
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.site2.rule=Host(`mysite2.lan`)"
    depends_on:
      - mariadb
```

### 2. Sites Web

Chaque site web est contenu dans son propre dossier (`site1/` et `site2/`) avec un Dockerfile et un fichier PHP.

#### Dockerfile (identique pour site1 et site2)

```dockerfile
FROM php:7.4-apache
RUN docker-php-ext-install mysqli
COPY . /var/www/html/
```

#### index.php (exemple pour site1, à adapter pour site2)

```php
<?php
$servername = "mariadb";
$username = "user";
$password = "password";
$dbname = "mydb";

// Create connection
$conn = new mysqli($servername, $username, $password, $dbname);

// Check connection
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

// Create table if not exists
$sql = "CREATE TABLE IF NOT EXISTS messages (id INT AUTO_INCREMENT PRIMARY KEY, message TEXT)";
$conn->query($sql);

// Insert a new message if form is submitted
if ($_SERVER["REQUEST_METHOD"] == "POST") {
    $message = $_POST['message'];
    $sql = "INSERT INTO messages (message) VALUES ('$message')";
    $conn->query($sql);
}

// Fetch all messages
$sql = "SELECT * FROM messages";
$result = $conn->query($sql);

?>

<!DOCTYPE html>
<html>
<head>
    <title>Simple Message Board</title>
</head>
<body>
    <h1>Simple Message Board</h1>
    <form method="post">
        <input type="text" name="message" placeholder="Enter a message">
        <input type="submit" value="Submit">
    </form>
    <h2>Messages:</h2>
    <?php
    if ($result->num_rows > 0) {
        while($row = $result->fetch_assoc()) {
            echo "<p>" . $row["message"] . "</p>";
        }
    } else {
        echo "No messages yet.";
    }
    $conn->close();
    ?>
</body>
</html>
```

## Installation et Démarrage

1. Assurez-vous que Docker et Docker Compose sont installés :
   ```
   sudo yum install -y docker docker-compose
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

2. Naviguez vers le dossier du projet et lancez les conteneurs :
   ```
   cd /chemin/vers/le/projet
   docker-compose up -d
   ```

3. Configurez votre fichier hosts local (sur votre machine, pas la VM) pour pointer vers l'IP de votre VM :
   ```
   <IP_DE_VOTRE_VM> mysite1.lan mysite2.lan
   ```

4. Accédez aux sites via votre navigateur :
   - http://mysite1.lan
   - http://mysite2.lan

## Utilisation

- Les deux sites web sont de simples tableaux de messages permettant d'ajouter et d'afficher des messages stockés dans la base de données MariaDB.
- Traefik gère le routage vers les sites appropriés en fonction de l'URL.
- Vous pouvez accéder à l'interface Traefik sur http://<IP_DE_VOTRE_VM>:8080 pour surveiller vos services.

## Dépannage

- Si les sites ne sont pas accessibles, vérifiez que les conteneurs sont en cours d'exécution avec `docker-compose ps`.
- Vérifiez les logs des conteneurs avec `docker-compose logs`.
- Assurez-vous que votre fichier hosts est correctement configuré.

## Sécurité

Note : Cette configuration est destinée à un environnement de développement. Pour la production, des mesures de sécurité supplémentaires seraient nécessaires, comme l'activation de HTTPS, la sécurisation de l'API Traefik, et l'utilisation de mots de passe plus robustes.
