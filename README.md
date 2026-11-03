# 💬 Déploiement d'une infrastructure Matrix souveraine (Synapse, PostgreSQL, Element Web & Synapse-Admin) sur TrueNAS SCALE

Ce guide détaille la mise en place d'un serveur de messagerie instantanée chiffrée **Matrix** auto-hébergé sur **TrueNAS SCALE**, conçu pour fonctionner en cercle privé (inscriptions verrouillées par jetons d'invitation, sans fédération publique ouverte) avec intégration automatique des nouveaux membres dans des salons prédéfinis.

L'architecture s'articule autour de 4 briques :
1. **PostgreSQL (`postgres-synapse`)** : Base de données relationnelle haute performance (remplace SQLite, inadapté en production).
2. **Synapse (`synapse`)** : Le moteur (*Homeserver*) qui gère les identités (`@utilisateur:utilisateur.org`), les salons et les médias.
3. **Element Web (`element-web`)** : Le client web léger servi aux utilisateurs.
4. **Synapse-Admin (`synapse-admin`)** : L'interface d'administration web permettant de gérer les comptes, les salons et les jetons d'invitation.

## 🔧 Prérequis

* Un serveur **TrueNAS SCALE** opérationnel (ici `192.168.1.xxx`) disposant idéalement d'un pool **SSD** pour les configurations/bases de données (`AppsPool`) et d'un pool **HDD** pour le stockage massif des médias (`storage`).
* Un nom de domaine (ici `utilisateur.org`) géré chez Cloudflare pointant vers l'IP publique du serveur.
* **Nginx Proxy Manager (NPM)** déployé et fonctionnel avec génération de certificats SSL Let's Encrypt (Challenge DNS-01 Cloudflare).
* Un accès SSH ou Shell root sur l'interface TrueNAS.

## 🔬 Étape 1 - Préparation du réseau, du DNS et des Datasets ZFS

### 1.1 Déclaration des sous-domaines DNS
1. Connecte-toi à ton tableau de bord **Cloudflare** et sélectionne la zone de ton domaine (`utilisateur.org`).
2. Crée deux nouveaux enregistrements de type `A` (ou `CNAME` vers `@`) pointant vers ton adresse IP publique :
   * `matrix.utilisateur.org` : point d'entrée technique du Homeserver Synapse.
   * `element.utilisateur.org` : adresse publique du client web Element.

### 1.2 Architecture de stockage hybride (SSD + HDD)
Pour optimiser les performances d'accès tout en préservant l'espace du SSD, la configuration et la base de données sont placées sur le SSD tandis que les fichiers envoyés dans les salons (photos, vidéos, documents) sont déportés sur la grappe HDD RAIDZ1.

1. Dans l'interface TrueNAS SCALE, rends-toi dans **Datasets**.
2. Sur ton pool **SSD** (`AppsPool`), crée deux datasets génériques :
   * `synapse-config` (chemin complet : `/mnt/AppsPool/synapse-config`)
   * `element-config` (chemin complet : `/mnt/AppsPool/element-config`)
3. Sur ton pool **HDD** (`storage`), crée le dataset dédié aux fichiers lourds :
   * `synapse-media` (chemin complet : `/mnt/storage/synapse-media`)
4. **Configuration des permissions (ACL)** : L'image officielle de Synapse s'exécute par défaut sous l'UID/GID `991` (ou `568` selon l'enveloppe TrueNAS). Modifie les permissions de `synapse-config` et `synapse-media` pour attribuer les droits de lecture/écriture à l'utilisateur `991` (et au groupe `568` `apps`). Attribue les droits de lecture à `element-config`.

## 🔬 Étape 2 - Déploiement de la base de données (`postgres-synapse`)

1. Dans TrueNAS SCALE, ouvre le menu **Apps** puis clique sur **Discover Apps** (ou déploie une instance PostgreSQL dédiée).
2. Configure l'application avec les paramètres suivants :
   * **Application Name** : `postgres-synapse`
   * **Database Name** : `synapse`
   * **Database User** : `synapse`
   * **Database Password** : Choisis un mot de passe robuste (note-le précieusement pour l'étape suivante).
   * **Port réseau** : `5432`
   * **Storage** : Laisse le stockage sur **PVC automatique (ixVolume)** rattaché au pool SSD d'applications. Cela garantit des performances d'I/O maximales et évite les conflits d'ACL ZFS propres à PostgreSQL.
3. Lance le déploiement et vérifie que le conteneur passe à l'état **Running**.

## 🔬 Étape 3 - Déploiement et configuration du Homeserver (`Synapse`)

### 3.1 Premier lancement et génération de `homeserver.yaml`
1. Dans **Apps**, installe **Synapse** (ou crée une *Custom App* basée sur l'image `matrixdotorg/synapse:latest`).
2. Renseigne les variables d'environnement indispensables à l'initialisation :
   * `SYNAPSE_SERVER_NAME` : `utilisateur.org` *(Attention : indique bien le domaine racine `utilisateur.org` et non `matrix.utilisateur.org` pour que tes identifiants ressemblent à `@pseudo:utilisateur.org`)*.
   * `SYNAPSE_REPORT_STATS` : `yes` (ou `no`).
   * `UID` : `991` / `GID` : `991`
3. Configure le réseau et les montages de volumes (**Host Path**) :
   * **Port Web / HTTP** : `8008`
   * **Volume de configuration** : Monte `/mnt/AppsPool/synapse-config` sur `/data` dans le conteneur.
   * **Volume des médias** : Monte `/mnt/storage/synapse-media` sur `/data/media_store` dans le conteneur.
4. Démarre le conteneur une première fois pour qu'il génère l'arborescence initiale et le fichier `/mnt/AppsPool/synapse-config/homeserver.yaml`.

*Astuce de dépannage : Si le dossier `/mnt/AppsPool/synapse-config` reste vide au premier lancement, ouvre le Shell TrueNAS et force la génération du fichier de configuration avec la commande suivante :*
```bash
docker run -it --rm \
  -v /mnt/AppsPool/synapse-config:/data \
  -e SYNAPSE_SERVER_NAME=utilisateur.org \
  -e SYNAPSE_REPORT_STATS=yes \
  matrixdotorg/synapse:latest generate
```

### 3.2 Édition chirurgicale du fichier `homeserver.yaml`
1. Arrête temporairement le conteneur `synapse` depuis l'interface TrueNAS.
2. Ouvre le Shell TrueNAS et édite le fichier de configuration :
```bash
nano /mnt/AppsPool/synapse-config/homeserver.yaml
```
3. Vérifie et ajuste l'URL publique du serveur ainsi que le chemin des médias :
```yaml
server_name: "utilisateur.org"
public_baseurl: "https://matrix.utilisateur.org/"
media_store_path: "/data/media_store"
max_upload_size: 100M
```
4. **Connexion à PostgreSQL** : Commente ou supprime le bloc `database` par défaut (SQLite3) et remplace-le par le connecteur `psycopg2`.
*(Note technique : L'ajout de `allow_unsafe_locale: true` permet d'accepter la collation `en_US.utf8` native du conteneur PostgreSQL de TrueNAS sans avoir à réinitialiser la base en locale `C`)* :
```yaml
database:
  name: psycopg2
  allow_unsafe_locale: true
  args:
    user: synapse
    password: "TON_MOT_DE_PASSE_POSTGRES"
    database: synapse
    host: 192.168.1.xxx
    port: 5432
    cp_min: 5
    cp_max: 10
```
5. **Verrouillage en serveur privé (Jetons d'invitation) et Auto-Join** : Ajoute ou modifie les directives suivantes pour autoriser uniquement les inscriptions munies d'un jeton, couper la fédération publique et forcer l'arrivée des nouveaux membres dans tes salons d'accueil :
```yaml
enable_registration: true
registration_requires_token: true

# Désactive la fédération publique (serveur 100% privé)
federation_domain_whitelist: []

# Intégration automatique dans les salons par défaut dès l'inscription
auto_join_rooms:
  - "#accueil:utilisateur.org"
  - "#general:utilisateur.org"
autocreate_auto_join_rooms: true
```
6. Sauvegarde (`Ctrl+O`, `Entrée`, `Ctrl+X`) et relance l'application `synapse`.

### 3.3 Création du compte Administrateur initial
1. Une fois le conteneur `synapse` démarré, ouvre un Shell à l'intérieur du conteneur `synapse` depuis l'interface TrueNAS.
2. Exécute l'utilitaire de création de compte :
```bash
register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```
3. Renseigne ton nom d'utilisateur, ton mot de passe, et réponds impérativement **`yes`** à la question `Make admin [no]:`.

## 🔬 Étape 4 - Déploiement et personnalisation du client (`element-web`)

### 4.1 Création du fichier `config.json` personnalisé
Pour éviter que tes utilisateurs aient à saisir manuellement l'adresse `https://matrix.utilisateur.org` à chaque connexion (Element pointant sur `matrix.org` par défaut), on injecte un fichier de configuration pré-rempli.

1. Dans le Shell TrueNAS, crée le fichier `/mnt/AppsPool/element-config/config.json` :
```bash
nano /mnt/AppsPool/element-config/config.json
```
2. Colle la configuration suivante (qui intègre également la préparation pour ton instance Jitsi locale) :
```json
{
  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://matrix.utilisateur.org",
      "server_name": "utilisateur.org"
    }
  },
  "disable_custom_urls": false,
  "disable_guests": true,
  "brand": "Element - Zaganos",
  "default_country_code": "FR",
  "show_labs_settings": true,
  "jitsi": {
    "preferred_domain": "jitsi.utilisateur.org"
  }
}
```
3. Sauvegarde et vérifie que le fichier est bien lisible par tous (`chmod 644 /mnt/AppsPool/element-config/config.json`).

### 4.2 Déploiement du conteneur `element-web`
1. Avant de choisir le port hôte, vérifie dans le Shell TrueNAS que le port visé (ici `30246`) est libre :
```bash
netstat -tulpen | grep 30246
```
2. Dans **Apps**, déploie **element-web** (catalogue TrueNAS ou image `vectorim/element-web:latest`) :
   * **Port Hôte** : `30246` (redirigé vers le port `80` du conteneur).
   * **Storage (Host Path)** : Monte le fichier `/mnt/AppsPool/element-config/config.json` directement sur `/app/config.json` dans le conteneur.
3. Démarre le conteneur.

## 🔬 Étape 5 - Configuration du Reverse Proxy (Nginx Proxy Manager) et délégation `.well-known`

### 5.1 Création des Proxy Hosts pour `matrix` et `element`
1. Ouvre l'interface de **Nginx Proxy Manager** et crée un premier **Proxy Host** pour le Homeserver :
   * **Domain Names** : `matrix.utilisateur.org`
   * **Scheme** : `http` | **Forward Hostname / IP** : `192.168.1.xxx` | **Forward Port** : `8008`
   * Active **Block Common Exploits** et **Websockets Support**.
   * **Onglet SSL** : Sélectionne ton certificat Let's Encrypt (`*.utilisateur.org`) et active **Force SSL** + **HTTP/2 Support**.
   * **Onglet Advanced** : Ajoute la directive suivante pour autoriser l'envoi de fichiers volumineux :
```nginx
client_max_body_size 100M;
```
2. Crée un second **Proxy Host** pour le client web :
   * **Domain Names** : `element.utilisateur.org`
   * **Scheme** : `http` | **Forward Hostname / IP** : `192.168.1.xxx` | **Forward Port** : `30246`
   * Active **Block Common Exploits**, **Websockets Support**, et le certificat SSL avec **Force SSL**.

### 5.2 Configuration de la délégation `.well-known` sur le domaine racine
Comme tes identifiants utilisent le domaine racine `@utilisateur:utilisateur.org` alors que Synapse écoute sur `matrix.utilisateur.org`, les applications clientes (Element Desktop, Element Mobile, Jitsi) interrogent automatiquement `https://utilisateur.org/.well-known/matrix/client`. Au lieu de créer des fichiers JSON sur un serveur web, Nginx Proxy Manager peut renvoyer directement la réponse en mémoire.

1. Dans Nginx Proxy Manager, édite le **Proxy Host** correspondant à ton domaine racine **`utilisateur.org`**.
2. Rends-toi dans l'onglet **Advanced** et ajoute ces deux blocs `location` :
```nginx
location /.well-known/matrix/server {
    default_type application/json;
    add_header Access-Control-Allow-Origin *;
    return 200 '{"m.server": "matrix.utilisateur.org:443"}';
}

location /.well-known/matrix/client {
    default_type application/json;
    add_header Access-Control-Allow-Origin *;
    return 200 '{"m.homeserver": {"base_url": "https://matrix.utilisateur.org"}, "org.matrix.msc3575.proxy": {"url": "https://matrix.utilisateur.org"}, "im.vector.riot.jitsi": {"preferredDomain": "jitsi.utilisateur.org"}}';
}
```
3. Enregistre. Tu peux immédiatement tester la délégation en ouvrant `https://utilisateur.org/.well-known/matrix/client` dans ton navigateur : le JSON doit s'afficher proprement.

## 🔬 Étape 6 - Déploiement de la console d'administration (`synapse-admin`) et gestion des invitations

Pour administrer ton serveur sans passer par des requêtes API `curl` en ligne de commande, on déploie **Synapse-Admin** en tant qu'application personnalisée accessible uniquement depuis ton réseau local (ou via WireGuard), ce qui évite d'exposer le panneau d'administration sur Internet.

### 6.1 Déploiement en Custom App sur TrueNAS
1. Dans **Apps**, clique sur **Discover Apps** puis sur **Custom App** en haut à droite.
2. Remplis les paramètres du conteneur :
   * **Application Name** : `synapse-admin`
   * **Image Repository** : `awesometechnologies/synapse-admin`
   * **Image Tag** : `latest`
   * **Networking (Port Forwarding)** : Mappe le port conteneur `80` vers le port hôte `30099` (protocole TCP).
3. Déploie l'application. Aucune persistance disque n'est requise : Synapse-Admin est une interface purement statique qui interroge directement l'API d'administration de Synapse.

### 6.2 Initialisation des salons et création des jetons d'invitation
1. Connecte-toi d'abord sur **`https://element.utilisateur.org`** avec ton compte administrateur créé à l'étape 3.3.
2. Crée deux salons (ou un Espace contenant deux salons) et attribue-leur dans leurs paramètres d'adresse locale les alias exacts déclarés dans `homeserver.yaml` :
   * `#accueil:utilisateur.org`
   * `#general:utilisateur.org`
3. Ouvre ensuite un nouvel onglet sur **`http://192.168.1.xxx:30099`** pour accéder à **Synapse-Admin**.
4. Connecte-toi avec tes identifiants administrateur en renseignant **`https://matrix.utilisateur.org`** dans le champ **Homeserver URL**.
5. Dans le menu latéral gauche, clique sur **Registration Tokens** puis sur **Create** :
   * Définis le nombre d'utilisations autorisées (*Uses allowed*, par exemple `1` pour un jeton nominatif à usage unique) et éventuellement une date d'expiration.
6. Transmets ce jeton à ton invité : lorsqu'il cliquera sur **Créer un compte** sur `https://element.utilisateur.org`, le serveur lui demandera ce jeton et l'intégrera automatiquement dans `#accueil:utilisateur.org` et `#general:utilisateur.org` dès sa première connexion.

![8](lien_vers_ta_capture_8)
*📸 [NOTE POUR TOI : Capture de l'onglet Registration Tokens dans Synapse-Admin montrant la création d'un jeton d'invitation.]*
