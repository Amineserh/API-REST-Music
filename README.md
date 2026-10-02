# API REST Music — PHP Slim 4
 
API REST développée en PHP avec le micro-framework **Slim 4**, exposant une base de données musicale (artistes, albums, notes). Projet réalisé dans le cadre du BTS CIEL option Informatique et Réseaux.
 
**API en ligne :** https://serhouani.alwaysdata.net
 
| Membre | Partie |
|---|---|
| … | Environnement et installation |
| … | Base de données et connexion PDO |
| … | Routage et opérations CRUD |
| … | Codes HTTP et tests Postman |
 
---
 
## Sommaire
 
1. [Technologies](#technologies)
2. [Installation locale](#installation-locale)
3. [Base de données](#base-de-données)
4. [Configuration](#configuration)
5. [Routes disponibles](#routes-disponibles)
6. [Authentification JWT](#authentification-jwt)
7. [Mise en ligne](#mise-en-ligne-alwaysdata)
8. [Étape suivante : le pattern Repository](#étape-suivante--le-pattern-repository)
9. [Structure du projet](#structure-du-projet)
10. [Problèmes rencontrés](#problèmes-rencontrés)
---
 
## Technologies
 
- PHP 8.5 (Homebrew)
- Slim 4 + PHP-DI 7
- MySQL / MariaDB
- `firebase/php-jwt`
- Postman pour les tests
- Hébergement : AlwaysData
---
 
## Installation locale
 
### Prérequis (macOS)
 
```bash
brew install php composer mysql
brew services start mysql
```
 
### Récupérer le projet
 
```bash
git clone https://github.com/VOTRE_COMPTE/mediatheque-api.git
cd mediatheque-api
composer install
```
 
### Lancer le serveur
 
```bash
php -S localhost:8080 -t public
```
 
> La commande doit être lancée **depuis le dossier du projet**. `-t public` est relatif au dossier courant : lancée ailleurs, elle renvoie 404 sur toutes les routes.
 
Vérification : http://localhost:8080 doit afficher `Hello world!`
 
---
 
## Base de données
 
Base `music`, trois tables liées par clés étrangères :
 
```
artists (idArtist, Name, Annee, Description)
   └─< albums (idAlbums, Titre, Artist_idArtist)
          └─< ratings (idRatings, Grade, Albums_idAlbums)
```
 
### Import
 
Le dump fourni ne contient pas de `CREATE DATABASE` :
 
```bash
mysql -u root -e "CREATE DATABASE music CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root music < music.sql
```
 
---
 
## Configuration
 
### `app/settings.php`
 
Ajouter un bloc `db` dans le tableau de `new Settings([...])` :
 
```php
'db' => [
    'host'     => '127.0.0.1',
    'port'     => 3306,
    'database' => 'music',
    'username' => 'root',
    'password' => '',
    'charset'  => 'utf8mb4',
    'flags'    => [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ],
],
```
 
> Sur macOS, utiliser `127.0.0.1` et non `localhost`, qui fait passer PDO par un socket Unix.
 
### `app/dependencies.php`
 
Déclarer PDO dans le conteneur d'injection de dépendances :
 
```php
PDO::class => function (ContainerInterface $c) {
    $db  = $c->get(SettingsInterface::class)->get('db');
    $dsn = "mysql:host={$db['host']};port={$db['port']};"
         . "dbname={$db['database']};charset={$db['charset']}";
 
    return new PDO($dsn, $db['username'], $db['password'], $db['flags']);
},
```
 
Dans toutes les routes, `$this->get(PDO::class)` renvoie alors la même connexion.
 
### `public/index.php`
 
Après la création de l'application :
 
```php
$app->addBodyParsingMiddleware();
```
 
Sans cette ligne, `getParsedBody()` renvoie `null` et aucun POST ne fonctionne.
 
---
 
## Routes disponibles
 
### Artistes
 
| Méthode | URL | Action | Réponses |
|---|---|---|---|
| GET | `/artists` | liste des artistes | 200 |
| GET | `/artists/{id}` | un artiste | 200 / 404 |
| GET | `/artists/{id}/albums` | albums d'un artiste | 200 |
| POST | `/artists` | créer un artiste | 201 / 400 |
| PUT | `/artists/{id}` | modifier un artiste | 200 |
| DELETE | `/artists/{id}` | supprimer un artiste | 204 / 404 / 409 |
 
### Albums
 
| Méthode | URL | Action | Réponses |
|---|---|---|---|
| GET | `/albums` | liste + nom de l'artiste (jointure) | 200 |
| GET | `/albums/{id}` | un album | 200 / 404 |
| GET | `/albums/{id}/ratings` | notes d'un album | 200 |
| POST | `/albums` | créer un album | 201 / 400 |
| PUT | `/albums/{id}` | modifier un album | 200 |
| DELETE | `/albums/{id}` | supprimer un album | 204 / 404 / 409 |
 
### Notes
 
| Méthode | URL | Action | Réponses |
|---|---|---|---|
| POST | `/ratings` | ajouter une note | 201 / 400 |
 
### Règles appliquées sur toutes les routes
 
- **Requêtes préparées** avec marqueurs nommés : aucune variable concaténée dans le SQL.
- **Objets PSR-7 immuables** : chaque `withHeader()` / `withStatus()` est retourné.
- **Un code de statut par cas**, y compris les échecs ; chaque accès à la base est protégé par un `try / catch (PDOException)`.
### Codes HTTP utilisés
 
| Code | Signification |
|---|---|
| 200 | OK |
| 201 | Created — avec en-tête `Location` |
| 204 | No Content — suppression réussie |
| 400 | Bad Request — champ obligatoire manquant ou clé étrangère invalide |
| 401 | Unauthorized — jeton absent ou invalide |
| 404 | Not Found — ressource inexistante |
| 405 | Method Not Allowed — route existante, mauvais verbe |
| 409 | Conflict — suppression bloquée par une clé étrangère |
| 500 | Erreur serveur |
 
### Tests Postman
 
Body → `x-www-form-urlencoded`.
 
| Requête | Résultat attendu |
|---|---|
| POST `/artists` avec Name, Annee, Description | 201 + `idArtist` |
| POST `/artists` sans Name | 400 |
| PUT `/artists/14` | 200, `lignes: 1` |
| PUT `/artists/14` rejoué à l'identique | 200, `lignes: 0` (idempotence) |
| POST `/albums` avec `Artist_idArtist` = 999 | 400 |
| DELETE `/artists/999` | 404 |
| DELETE `/artists/4` (possède des albums) | 409 |
| DELETE d'un artiste sans album | 204 |
 
---
 
## Authentification JWT
 
```bash
composer require firebase/php-jwt
composer dump-autoload
```
 
Deux classes dans `src/Middleware/` :
 
- **`JwtHelper`** — `generateToken()` et `validateToken()`
- **`JwtMiddleware`** — lit l'en-tête `Authorization: Bearer <token>`, laisse passer avec `$handler->handle($request)` ou renvoie 401
> **Clé secrète d'au moins 32 caractères** : HS256 produit une empreinte de 256 bits, `php-jwt` refuse toute clé plus courte (`Provided key is too short`).
 
### Routes
 
| Méthode | URL | Description |
|---|---|---|
| POST | `/login` | renvoie un jeton (`username` / `password`) |
| GET | `/protected` | route protégée par `->add(new JwtMiddleware())` |
 
### Tests
 
| Requête | Résultat |
|---|---|
| POST `/login` avec les bons identifiants | 200 + jeton |
| POST `/login` avec un mauvais mot de passe | 401 |
| GET `/protected` sans jeton | 401 |
| GET `/protected` avec `Bearer <jeton>` | 200 |
| GET `/protected` avec un jeton modifié ou expiré | 401 |
 
> Le payload d'un JWT est encodé en base64, **pas chiffré** : il est lisible par tous. On n'y place jamais de mot de passe.
 
---
 
## Mise en ligne (AlwaysData)
 
1. Créer la base MySQL dans l'admin, importer le dump via phpMyAdmin.
2. Envoyer le projet :
```bash
   rsync -avz --exclude 'logs/*' --exclude 'var/cache/*' \
     ./ COMPTE@ssh-COMPTE.alwaysdata.net:~/www/mediatheque-api/
```
3. Deux fichiers `.htaccess` : un dans `public/` (tout renvoyer vers `index.php`), un à la racine (tout renvoyer vers `public/`).
4. Adapter `app/settings.php` : hôte `mysql-COMPTE.alwaysdata.net`, base `COMPTE_music`, utilisateur et mot de passe MySQL.
5. **Web → Sites** : type PHP, répertoire racine `www/mediatheque-api/`.
> Linux est sensible à la casse : `Middleware` ≠ `middleware`.
 
---
 
## Étape suivante : le pattern Repository
 
> Partie à réaliser — slides 42 à 48 du cours.
 
### Le problème
 
Chaque route répète le même code : `prepare()`, `bindValue()`, `execute()`, `fetch()`, `json_encode()`. Pour 3 tables et 5 opérations, c'est 15 blocs quasi identiques.
 
### La solution
 
Une classe parente **abstraite** contient le CRUD, écrit une seule fois. Chaque table a son repository enfant qui en hérite et n'ajoute que ses méthodes spécifiques.
 
```
src/
├── Entity/
│   └── Artist.php              ← une ligne SQL = un objet PHP
└── Repository/
    ├── BaseRepository.php      ← CRUD générique (abstract)
    ├── ArtistRepository.php    ← hérite + findByYear()
    └── AlbumRepository.php     ← hérite + findByArtist()
```
 
C'est exactement ce que fait un ORM comme Doctrine ; l'écrire à la main permet de comprendre ce que l'ORM automatise.
 
### Étape 1 — `src/Repository/BaseRepository.php`
 
```php
<?php
 
declare(strict_types=1);
 
namespace App\Repository;
 
use PDO;
 
abstract class BaseRepository
{
    protected string  $table;                 // ex. 'artists'
    protected string  $primaryKey;            // ex. 'idArtist'
    protected array   $columns;               // colonnes autorisées (liste blanche)
    protected ?string $entityClass = null;    // ex. Artist::class -> renvoie des objets
 
    public function __construct(protected PDO $db) {}   // PDO injecté par PHP-DI
 
    // Ligne SQL -> entité si $entityClass est défini, sinon on garde le tableau
    protected function hydrate(array $row): array|object
    {
        return $this->entityClass ? $this->entityClass::fromArray($row) : $row;
    }
 
    public function findAll(): array
    {
        $rows = $this->db->query("SELECT * FROM `{$this->table}`")->fetchAll(PDO::FETCH_ASSOC);
        return array_map([$this, 'hydrate'], $rows);
    }
 
    public function findById(int $id): array|object|null
    {
        $sth = $this->db->prepare("SELECT * FROM `{$this->table}` WHERE `{$this->primaryKey}` = :id");
        $sth->execute(['id' => $id]);
        $row = $sth->fetch(PDO::FETCH_ASSOC);
 
        return $row ? $this->hydrate($row) : null;
    }
 
    public function insert(array $data): int
    {
        $data = array_intersect_key($data, array_flip($this->columns));   // colonnes autorisées seulement
        if (!$data) {
            throw new \InvalidArgumentException('Aucune colonne valide');
        }
 
        $cols   = implode('`, `', array_keys($data));       // Name`, `Annee
        $params = ':' . implode(', :', array_keys($data));  // :Name, :Annee
 
        $this->db->prepare("INSERT INTO `{$this->table}` (`$cols`) VALUES ($params)")->execute($data);
 
        return (int) $this->db->lastInsertId();
    }
 
    public function update(int $id, array $data): bool
    {
        $data = array_intersect_key($data, array_flip($this->columns));
        if (!$data) {
            throw new \InvalidArgumentException('Aucune colonne valide');
        }
 
        $set = implode(', ', array_map(fn ($c) => "`$c` = :$c", array_keys($data)));
        $sql = "UPDATE `{$this->table}` SET $set WHERE `{$this->primaryKey}` = :id";
 
        return $this->db->prepare($sql)->execute($data + ['id' => $id]);
    }
 
    public function delete(int $id): bool
    {
        $sth = $this->db->prepare("DELETE FROM `{$this->table}` WHERE `{$this->primaryKey}` = :id");
        $sth->execute(['id' => $id]);
 
        return $sth->rowCount() > 0;   // false si rien n'a été supprimé
    }
}
```
 
**Pourquoi une liste blanche `$columns` ?** Les *valeurs* passent par des paramètres liés, mais un *nom de colonne* ne peut pas être lié en SQL. Sans filtre, un client pourrait envoyer un champ `"idArtist"` ou un nom de colonne malveillant. `array_intersect_key` ne garde que les colonnes autorisées.
 
### Étape 2 — `src/Entity/Artist.php`
 
```php
<?php
 
declare(strict_types=1);
 
namespace App\Entity;
 
class Artist implements \JsonSerializable
{
    public function __construct(
        private ?int $idArtist = null,
        private string $name = '',
        private ?int $annee = null,
        private ?string $description = null,
    ) {}
 
    // Ligne SQL -> objet
    public static function fromArray(array $row): self
    {
        return new self(
            isset($row['idArtist']) ? (int) $row['idArtist'] : null,
            $row['Name'] ?? '',
            isset($row['Annee']) ? (int) $row['Annee'] : null,
            $row['Description'] ?? null,
        );
    }
 
    // Objet -> tableau
    public function toArray(): array
    {
        return [
            'idArtist'    => $this->idArtist,
            'Name'        => $this->name,
            'Annee'       => $this->annee,
            'Description' => $this->description,
        ];
    }
 
    // Permet json_encode($artist) directement dans les routes
    public function jsonSerialize(): array
    {
        return $this->toArray();
    }
 
    public function getId(): ?int { return $this->idArtist; }
    public function getName(): string { return $this->name; }
}
```
 
Le constructeur utilise la **promotion de propriétés** (PHP 8) : les propriétés sont déclarées directement dans ses paramètres.
 
### Étape 3 — `src/Repository/ArtistRepository.php`
 
```php
<?php
 
declare(strict_types=1);
 
namespace App\Repository;
 
use App\Entity\Artist;
use PDO;   // obligatoire : on utilise PDO::FETCH_ASSOC dans un namespace
 
class ArtistRepository extends BaseRepository
{
    protected string  $table       = 'artists';
    protected string  $primaryKey  = 'idArtist';
    protected array   $columns     = ['Name', 'Annee', 'Description'];
    protected ?string $entityClass = Artist::class;
 
    // Méthode spécifique — tout le CRUD est hérité
    public function findByYear(int $annee): array
    {
        $sth = $this->db->prepare("SELECT * FROM `artists` WHERE `Annee` = :annee");
        $sth->execute(['annee' => $annee]);
 
        return array_map([$this, 'hydrate'], $sth->fetchAll(PDO::FETCH_ASSOC));
    }
}
```
 
> ⚠ Le `use PDO;` manque sur la slide 46 du cours. Sans lui, PHP cherche une classe `App\Repository\PDO` et lève une erreur fatale.
 
### Étape 4 — `src/Repository/AlbumRepository.php`
 
```php
<?php
 
declare(strict_types=1);
 
namespace App\Repository;
 
use PDO;
 
class AlbumRepository extends BaseRepository
{
    protected string $table      = 'albums';
    protected string $primaryKey = 'idAlbums';
    protected array  $columns    = ['Titre', 'Artist_idArtist'];
 
    public function findByArtist(int $idArtist): array
    {
        $sth = $this->db->prepare("SELECT * FROM `albums` WHERE `Artist_idArtist` = :id");
        $sth->execute(['id' => $idArtist]);
 
        return $sth->fetchAll(PDO::FETCH_ASSOC);
    }
}
```
 
> ⚠ La slide 46 utilise `idAlbum`, `Title`, `Year`, `idArtist`. Ces colonnes **n'existent pas** dans `music.sql`, qui contient `idAlbums`, `Titre` et `Artist_idArtist`. Les noms ci-dessus correspondent à la vraie base.
 
Pas de `$entityClass` : les albums sont renvoyés sous forme de tableaux, ce qui suffit.
 
### Étape 5 — Régénérer l'autoload
 
```bash
composer dump-autoload
```
 
### Étape 6 — Les routes dans `app/routes.php`
 
Imports en haut du fichier :
 
```php
use App\Repository\ArtistRepository;
use App\Repository\AlbumRepository;
use Slim\Interfaces\RouteCollectorProxyInterface as Group;
```
 
Routes, à l'intérieur de `return function (App $app) { ... }` :
 
```php
$app->group('/api', function (Group $group) {
 
    // --- Artistes ---------------------------------------------------------
    $group->get('/artists', function (Request $request, Response $response) {
        $repo = $this->get(ArtistRepository::class);   // PHP-DI injecte PDO tout seul
        $response->getBody()->write(json_encode($repo->findAll()));
        return $response->withHeader('Content-Type', 'application/json');
    });
 
    $group->get('/artists/{id}', function (Request $request, Response $response, array $args) {
        $artist = $this->get(ArtistRepository::class)->findById((int) $args['id']);
        $response->getBody()->write(json_encode($artist ?? ['error' => 'Artiste introuvable']));
        return $response
            ->withHeader('Content-Type', 'application/json')
            ->withStatus($artist ? 200 : 404);
    });
 
    $group->get('/artists/year/{annee}', function (Request $request, Response $response, array $args) {
        $artists = $this->get(ArtistRepository::class)->findByYear((int) $args['annee']);
        $response->getBody()->write(json_encode($artists));
        return $response->withHeader('Content-Type', 'application/json');
    });
 
    $group->post('/artists', function (Request $request, Response $response) {
        $data = (array) $request->getParsedBody();
 
        if (empty($data['Name'])) {
            $response->getBody()->write(json_encode(['error' => 'Le champ Name est obligatoire']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(400);
        }
 
        $id = $this->get(ArtistRepository::class)->insert($data);
        $response->getBody()->write(json_encode(['idArtist' => $id]));
        return $response
            ->withHeader('Content-Type', 'application/json')
            ->withHeader('Location', '/api/artists/' . $id)
            ->withStatus(201);
    });
 
    $group->put('/artists/{id}', function (Request $request, Response $response, array $args) {
        $repo = $this->get(ArtistRepository::class);
        $id   = (int) $args['id'];
 
        if (!$repo->findById($id)) {
            $response->getBody()->write(json_encode(['error' => 'Artiste introuvable']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(404);
        }
 
        $repo->update($id, (array) $request->getParsedBody());
        $response->getBody()->write(json_encode($repo->findById($id)));
        return $response->withHeader('Content-Type', 'application/json');
    });
 
    $group->delete('/artists/{id}', function (Request $request, Response $response, array $args) {
        try {
            $deleted = $this->get(ArtistRepository::class)->delete((int) $args['id']);
            return $response->withStatus($deleted ? 204 : 404);
        } catch (PDOException $e) {
            $response->getBody()->write(json_encode(['error' => 'Cet artiste possède encore des albums']));
            return $response->withHeader('Content-Type', 'application/json')->withStatus(409);
        }
    });
 
    // --- Albums ----------------------------------------------------------
    $group->get('/albums', function (Request $request, Response $response) {
        $response->getBody()->write(json_encode($this->get(AlbumRepository::class)->findAll()));
        return $response->withHeader('Content-Type', 'application/json');
    });
 
    $group->get('/artists/{id}/albums', function (Request $request, Response $response, array $args) {
        $albums = $this->get(AlbumRepository::class)->findByArtist((int) $args['id']);
        $response->getBody()->write(json_encode($albums));
        return $response->withHeader('Content-Type', 'application/json');
    });
 
})->add(new JwtMiddleware());   // toutes les routes /api sont protégées
```
 
`$this->get(ArtistRepository::class)` fonctionne sans déclaration supplémentaire : PHP-DI voit que le constructeur attend un `PDO` et l'**injecte automatiquement** depuis le conteneur (*autowiring*).
 
### Étape 7 — Tester
 
Toutes les routes `/api` exigent le jeton : POST `/login` d'abord, puis Authorization → Bearer Token.
 
| Requête | Résultat attendu |
|---|---|
| GET `/api/artists` sans jeton | 401 |
| GET `/api/artists` avec jeton | 200, liste d'objets Artist |
| GET `/api/artists/4` | 200 |
| GET `/api/artists/999` | 404 |
| GET `/api/artists/year/2000` | Aani, Eric, celine |
| POST `/api/artists` avec Name | 201 + `idArtist` |
| POST `/api/artists` sans Name | 400 |
| PUT `/api/artists/{id}` | 200 + artiste mis à jour |
| DELETE `/api/artists/4` | 409 (possède des albums) |
| GET `/api/artists/4/albums` | SoloSun, Sunrise |
 
### Checklist
 
- [ ] `BaseRepository.php`
- [ ] `Entity/Artist.php`
- [ ] `ArtistRepository.php` (avec `use PDO;`)
- [ ] `AlbumRepository.php` (colonnes de `music.sql`)
- [ ] `composer dump-autoload`
- [ ] Groupe `/api` dans `routes.php`, protégé par JWT
- [ ] Tests Postman
- [ ] Mise en ligne des nouveaux fichiers
### Comparaison avec Doctrine
 
| | Notre BaseRepository | Doctrine |
|---|---|---|
| Tout lire | `$repo->findAll()` | `$repo->findAll()` |
| Un élément | `$repo->findById(1)` | `$repo->find(1)` |
| Filtrer | `$repo->findByYear(2000)` | `$repo->findBy(['Annee' => 2000])` |
| Créer | `$repo->insert($data)` | `$em->persist($a); $em->flush();` |
| Spécifique | `extends BaseRepository` | `extends EntityRepository` |
 
---
 
## Structure du projet
 
```
mediatheque-api/
├── app/
│   ├── dependencies.php     ← PDO dans le conteneur
│   ├── middleware.php
│   ├── routes.php           ← toutes les routes
│   └── settings.php         ← paramètres BDD
├── public/
│   ├── .htaccess
│   └── index.php            ← point d'entrée unique
├── src/
│   ├── Middleware/
│   │   ├── JwtHelper.php
│   │   └── JwtMiddleware.php
│   ├── Entity/              ← étape Repository
│   └── Repository/          ← étape Repository
├── vendor/                  ← non versionné
├── .htaccess
└── composer.json
```
 
---
 
## Problèmes rencontrés
 
| Symptôme | Cause | Solution |
|---|---|---|
| Avertissements `Deprecated` puis erreur 500 | PHP-DI 6 incompatible PHP 8.4+ | `composer require php-di/php-di:^7 -W` |
| 404 sur toutes les routes en local | Serveur lancé hors du dossier du projet | `cd` dans le projet avant `php -S` |
| `Class "App\Middleware\JwtMiddleware" not found` | Fichiers absents ou autoload périmé | créer `src/Middleware/` puis `composer dump-autoload` |
| `Provided key is too short` | Clé JWT de moins de 32 caractères | allonger la clé secrète |
| `$input` vaut `null` sur un POST | middleware de parsing absent | `$app->addBodyParsingMiddleware()` |
| Import SQL : `BINLOG ADMIN privilege` | dump contenant `SET SQL_LOG_BIN` | `mysqldump --set-gtid-purged=OFF` et supprimer la ligne |
| Import SQL : `errno 150` | tables créées dans le désordre | lister les tables dans l'ordre : `artists albums ratings` |
| Page « hébergé par alwaysdata » | racine du site sur `/www/` | racine `www/mediatheque-api/` |
| 405 sur `/login` ou `/protected` | mauvais verbe HTTP | POST pour `/login`, GET pour `/protected` |
 
