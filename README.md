# Pré-réquis
- Générer un nouvelle app express grace au générateur express (express-cli)
`express nodejs-starter --no-view --git`
- installer les dépendences nécéssaire à la configuration
`npm i --save sequelize mysql2`
- Création d'une nouvelle base de donnée
```shell
mysql -u root -p
mysql > create database my_db_name;
# mysql> create database my_db_name;
# Query OK, 1 row affected (0.00 sec)
mysql > exit
```
# Base de donnée
## Configuration
Si ce n'est pas déjà fait, installer le cli de sequelize
```shell
npm install -g sequelize-cli
```
puis
```shell
sequelize init
# Sequelize CLI [Node: 8.11.3, CLI: 4.1.1, ORM: 4.41.2]

# Created "config/config.json"
# Successfully created models folder at "/nodeProjects/nodejs-starter/models".
# Successfully created migrations folder at "/nodeProjects/nodejs-starter/migrations".
# Successfully created seeders folder at "/nodeProjects/nodejs-starter/seeders".
```
Comme on peut le constater, le cli sequelize nous à généré 3 nouveaux dossiers et 1 fichier
```
/config/config.json
/models
/migrations
/seeders
```
Il faut ensuite remplir le fichier `config.json` avec les bonnes infos pour la connexion à la base de donnée.
```json
{
  "development": {
    "username": "root",
    "password": "MyRootPassword",
    "database": "my_db_name_development",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "test": {
    "username": "root",
    "password": null,
    "database": "my_db_name_test",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "production": {
    "username": "root",
    "password": null,
    "database": "my_db_name_production",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
}
```

## Création des modèles
Dans une application web, un modèle est une classe liée à une table de la base de donnée, c'est lui qui gèrera la connexion avec la table concerné. Même si le générateur de sequelize le fait pour nous, il faut savoir que le modèle doit être au singulier contrairement à la table qui elle est au pluriel.
Nous allons ici créer 3 modèles, 2 pour la relation n:n et 1 qui servira pour la jointure des deux, car 1 Media peut avoir plusieurs tags et 1 Tag peut avoir plusieurs Media
- Media (table: medias) n:n
- Tag (table: tags) n:n
- MediaTag (table: media_tags) 1:1
Pour ce faire nous allons utiliser le générateur sequelize de la manière suivante
```shell
sequelize model:generate --name Media --attributes name:STRING,type:STRING,url:STRING
sequelize model:generate --name Tag --attributes name:STRING
sequelize model:generate --name MediaTag --attributes mediaId:INTEGER,tagId:INTEGER
```
Pour chaques modèle créé, sequelize nous génère 2 fichiers, 1 modèle et 1 migration

## Jointure des tables
Dans notre exemple nous utilisons une relation n:n (many-to-many) ce qui n'ecessite donc une table de jointure (media_tags) c'est elle qui va porter les références de Media et de Tag.
Dans le fichier de migration de MediaTag, nous allons devoir ajouter manuellement les références de la manière suivante :
```javascript
mediaId: {
  type: Sequelize.INTEGER,
  foreignKey: true,
  allowNull: false,
  references: {
    model: "Medias", // le model peut être une chaine de caractère représentant la table (pluriel)
    key: 'id',
  }
},
tagId: {
  type: Sequelize.INTEGER,
  foreignKey: true,
  allowNull: false,
  references: {
    model: "Tags", // le model peut aussi être un objet JS représentant le modèle lui même (singulier (import nécessaire))
    key: 'id',
  }
},
```
Nous devons ensuite gérer las associations dans chaque modèle
/models/media.js
```javascript
Media.associate = function(models) {
    Media.belongsToMany(models.Tag, {through: 'MediaTags', foreignKey : 'MediaId'})
  };
```
/models/tag.js
```javascript
Tag.associate = function(models) {
    Tag.belongsToMany(models.Tag, {through: 'MediaTags', foreignKey : 'TagId'})
  };
```
/models/mediatag.js
```javascript
MediaTag.associate = function(models) {
  MediaTag.belongsTo(models.Media)
  MediaTag.belongsTo(models.Tag)
};
```

# Routage
Nous allons maintenant avoir besoin de créer les routes pour la gestions des ressources
Créer deux nouveaux fichiers dans le dossier `/routes`
- `/routes/tags.js`
- `/routes/medias.js`
Il faut ensuite importer les deux fichiers dans `app.js` et les utiliser en tant que routeur afin que les routes soit accessible.
```javascript
var tagsRouter = require('./routes/tags');
var mediasRouter = require('./routes/medias');
...
app.use('/tags', tagsRouter);
app.use('/medias', mediasRouter);
```
## Actions
Implémenter les actions nécessaire à la gestion des ressources, on appelle ça un CRUD
### Tag
- [CREATE](./routes/tags.js#L27)
- [READ](./routes/tags.js#L15)
- [UPDATE](./routes/tags.js#L42)
- [DESTROY](./routes/tags.js#L64)
```text
Récupérer la liste de tout les médias existant
GET http://localhost:3000/tags
result = [{id: 1, name: "myTagName"}, {id: 2, name: "myTagName2"}, ...]

Creer un nouveau tag
POST http://localhost:3000/tags/create
body = {name: "tagName"}

Récupérer un tag donné
GET http://localhost:3000/tags/1
{id: 1, name: "mytagName"}

Editer un tag donné
PUT http://localhost:3000/tags/edit/1
body = {name: "newTagName"}

Supprimer un tag donné
DELETE http://localhost:3000/tags/1
result "Media has been deleted !"
```
### Media
- [CREATE](./routes/medias.js#L29)
- [READ](./routes/medias.js#L16)
- [UPDATE](./routes/medias.js#L52)
- [DESTROY](./routes/medias.js#L82)
```text
Récupérer la liste de tout les médias existant
GET http://localhost:3000/medias
result = [{id: 1, name: "myMediaName"}, {id: 2, name: "myMediaName2"}, ...]

Creer un nouveau média lié aux tags 1, 2 et 3 (id)
POST http://localhost:3000/medias/create
body = {name: "mediaName", type: "mediaType", url: "mediaUrl", tagIds: "1,2,3"}

Récupérer un média donné
GET http://localhost:3000/medias/1
{id: 1, name: "myMediaName"}

Editer un média donné
PUT http://localhost:3000/medias/edit/1
body = {name: "newMediaName", type: "newMediaType", url: "newMediaUrl", tagIds: "3"}

Supprimer un média donné
DELETE http://localhost:3000/medias/1
result "Media has been deleted !"
```

# Gestions utilisateurs
## Création du modèle et des migrations
```shell
sequelize model:generate --name User --attributes firstName:STRING,lastName:STRING,email:STRING,password:STRING,isAdmin:BOOLEAN

sequelize db:migrate
```
## Création des routes
- [routes/users.js](./routes/users.js)
- [routes/auth.js](./routes/auth.js)

## Création de l'authentification
### Signin
installation des librairies
```shell
npm i --save passport passport-local passport-jwt 
```
`passport.js` est une librairie qui permet de gérer tout type d'authentification utilisateur (strategies). nous utiliserons `local` pour la connexion par email et mot de passe, et `jwt` pour la protections des routes.
- [Configuration de la strategie local](./routes/strategies/local.js)
- [Utilisation sur la route auth/signin](./routes/auth.js#L7)

### Routes protégés
- [Configuration de la strategie jwt](./routes/strategies/jwt.js)
- [Protection d'une route](./routes/users.js#L7)
```shell
POST /auth/signup # body = {firstname: 'john', lastName: 'DOE', email: 'john-doe@domain.com', password: 'mypass'}
# It return 
{
    "user": {
        "id": 1,
        "firstName": "john",
        "lastName": "DOE",
        "email": "john-doe@domain.com",
        "password": "$2b$10$6X5cmJTDsWoJ7XB.fBnNm.geSVSUJX15x7caRk1JvIgmRuJefNNDi",
        "isAdmin": false,
        "updatedAt": "2018-11-20T22:26:04.439Z",
        "createdAt": "2018-11-20T22:26:04.439Z"
    }
}

POST /auth/signin # body = {email: 'john-doe@domain.com', password: 'mypass'}
# It return
{
    "user": {
        "id": 1,
        "email": "john-doe@domain.com",
        "isAdmin": false
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6NCwiZW1haWwiOiJqZWFuLWJlYmVyQGhvdG1haWwuZnIiLCJpc0FkbWluIjpmYWxzZSwiaWF0IjoxNTQyNzUyNzkwfQ.dbStYNlaXdpMzS-vo4VfxmFNEiJnbxA8C04sHN5RYCE"
}

GET /users # headers = {Authorization: "bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6NCwiZW1haWwiOiJqZWFuLWJlYmVyQGhvdG1haWwuZnIiLCJpc0FkbWluIjpmYWxzZSwiaWF0IjoxNTQyNzUyNzkwfQ.dbStYNlaXdpMzS-vo4VfxmFNEiJnbxA8C04sHN5RYCE"}
# It return
{
    "users": [
        {
            "id": 1,
            "firstName": "john",
            "lastName": "DOE",
            "email": "john-doe@domain.com",
            "password": "$2b$10$6X5cmJTDsWoJ7XB.fBnNm.geSVSUJX15x7caRk1JvIgmRuJefNNDi",
            "isAdmin": false,
            "updatedAt": "2018-11-20T22:26:04.439Z",
            "createdAt": "2018-11-20T22:26:04.439Z"
        },
        {
            "id": 2,
            "firstName": "kevin",
            "lastName": "MITNIK",
            "email": "kevin-mitnik@hotmail.fr",
            "password": "$2b$10$6X5cmJTDsWoJ7XB.fBnNm.geSVSUJX15x7caRk1JvIgmRuJefNNDi",
            "isAdmin": false,
            "createdAt": "2018-11-20T22:26:04.000Z",
            "updatedAt": "2018-11-20T22:26:04.000Z"
        }
    ]
}
```

## Controllers
Les controllers sont les fichiers qui doivent abriter les actions
- 1 modèle à une ou plusieurs routes
- 1 route correspont à une action

Afin de garder un projet propre est structuré, il est recommender des déplacer les actions dans les controllers. les routeur doivent servir uniquement à créer la route et lier l'action concerné.

Exemple d'application
[Route index users](./routes/users.js#L7) => [Action index users](./controllers/usersController.js#L4)