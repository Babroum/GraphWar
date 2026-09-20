# Graphwar - Structure du projet

## Notes générales

- Aucune persistance des données après fermeture du jeu ou en quittant une partie ; les données ne durent que le temps de la partie (option future : statistiques joueur en JSON).
- Le jeu est en **P2P**, aucun serveur hôte n'est nécessaire au jeu lui-même (tous les clients font le même calcul). Il y a un créateur de partie (l'hôte), mais il n'est pas le "chef d'orchestre" de la logique de jeu ; son rôle est transféré simplement au joueur suivant en cas de déconnexion, sans impact sur le déroulement.
- Pas de système de reconnexion pour l'instant : un joueur déconnecté quitte définitivement la partie en cours, personne ne reprend sa place.
- La communication entre l'Affichage et les autres domaines se fait par **callback** (l'Affichage prévient quand un événement se produit, sans appeler directement le domaine concerné) — pas de "Logique UI" séparée, pas de polling.
- Le jeu est organisé en 3 niveaux temporels autonomes, chacun parlant directement au Réseau avec ses propres protocoles : **MainMenu** → **RoomManager** (salon) → **GameEngine** (partie en cours).
- Chaque fiche correspond à un **package Java**, avec une classe façade qui orchestre le domaine. Les autres domaines n'appellent jamais directement une classe interne à un package qui n'est pas le leur — uniquement la façade, ou via des callbacks (`MessageListener`) pour le Réseau et l'Affichage.
- Point d'entrée Maven (`App.java` / `main`) séparé de toute logique métier : il ne fait que démarrer `InterfaceManager`/`MainMenu`.

## Actions possibles

- **Créer une partie**
    - Entrer un pseudo
    - Ajouter des joueurs bots
    - Modifier l'équipe des joueurs
    - Modifier le nombre de personnages par joueur
    - Quitter le salon de la partie
- **Rejoindre une partie**
    - Entrer un pseudo
    - Entrer une IP
    - Entrer un port
- **Jouer**
    - Quitter la partie
    - Entrer une fonction
    - Exécuter une fonction
    - Parler dans le chat
    - Perdre
    - Gagner

## Entités dans le jeu

- Des joueurs (chacun avec un ou plusieurs pions), y compris des bots
- Une partie
- Un terrain (zone 2D fixe, x de -25 à 25, y de -15 à 15)
- Des obstacles (cercles pleins à rayon variable, potentiellement destructibles plus tard)
- Un projectile / fonction mathématique

## Qui a besoin de savoir quoi ?

### MainMenu

- **Entrée** : saisies du menu (pseudo, IP, port, paramètres de création de salon)
- **Sortie** : initie la connexion réseau (rejoindre) ou crée le salon (hôte), puis passe la main à RoomManager
- Gère uniquement le menu d'accueil ; une fois connecté, ne gère plus rien du salon
- Testable en CLI (validation Réseau/protocoles de base)
- N'a pas besoin de savoir si les autres entités existent

### RoomManager (Gestion du Salon)

- **Entrée** : inputs UI de la page salon (ajout de bots, équipes, nombre de personnages, chat), protocoles réseau du salon
- **Sortie** : mise à jour de l'affichage du salon, protocoles réseau envoyés (l'hôte envoie les modifs, reçoit les connexions ; les invités reçoivent tout, envoient juste "prêt" et le chat)
- Page UI autonome et indépendante de MainMenu ; gère aussi sa propre fermeture (quitter le salon)
- Nécessite l'Affichage complet pour être testée
- N'a pas besoin de savoir si les autres entités existent

### Affichage (InterfaceManager)

- **Entrée** : données du jeu (positions, texte, …)
- **Sortie** : affichage du jeu, déclenche des callbacks vers le domaine concerné lors d'un input (clic, texte tapé, …)
- N'a pas besoin de savoir si les autres entités existent

### Network

- **Entrée / Sortie** : messages taggés par un protocole (entier constant dans `NetworkProtocol`), pour tous les contextes (MainMenu, Room, GameEngine, Chat)
- Format de message : `Message implements Serializable` (protocole + données) ; une seule paire de méthodes d'envoi/réception générique, peu importe le nombre de protocoles
- Ne décide de rien, ne connaît aucun domaine directement : chaque domaine s'enregistre via `MessageListener` (une lambda par protocole) ; Network appelle la bonne lambda à réception
- Détecte aussi localement les pertes de connexion et prévient le domaine concerné (ex : joueur déconnecté en plein tour)
- N'a pas besoin de savoir si les autres entités existent

### ChatManager

- **Entrée** : message + expéditeur à transmettre (vers Network), message + expéditeur reçu (depuis Network)
- **Sortie** : callback vers l'Affichage pour ajouter le message à la chat box
- Domaine transversal, potentiellement partagé entre salon et partie (flexibilité d'implémentation)
- N'a pas besoin de savoir si les autres entités existent

### Parsing

- **Entrée** : formule du joueur/bot, message de chat, tout input UI
- **Sortie** : arbre d'expression exploitable (`Noeud`)
- Deux types de nœuds : opérations binaires (+, -, *, /, ^ — deux enfants) et fonctions unaires (sin, cos, sqrt — un seul enfant)
- Parenthèses gérées dès la V1 par la récursivité du parser (descente récursive), tokenisation et parsing en méthodes internes de la façade
- N'a pas besoin de savoir si les autres entités existent

### Physics

- **Entrée** : arbre d'expression (`Noeud`, déjà parsé)
- **Sortie** : collection de `Point` (la trajectoire)
- Ne connaît pas les positions des joueurs/obstacles, ne détecte aucune collision
- N'a pas besoin de savoir si les autres entités existent

### GameEngine (Moteur de Jeu)

- Chaque client fait tourner son propre jeu ; chaque moteur doit fonctionner à l'identique chez tous les joueurs, sinon désynchronisation.
- Pas de moteur serveur/client : tous les joueurs transmettent leurs données, et un moteur désynchronisé se resynchronise.
- Système de tour **par pion** (pas par joueur entier) : tous les pions n°1 jouent, puis tous les n°2, etc. Un Bot est traité comme un joueur normal par GameEngine — seule la source de la formule diffère (BotEngine au lieu d'un input humain).
- Génère les positions (obstacles, pions) depuis la seed commune au lancement, via `SeedGenerator` (ordre : nombre d'obstacles, puis position x/y et rayon pour chacun ; les obstacles peuvent se superposer sans contrainte). L'état d'un obstacle reste modifiable pour préparer la destructibilité future.
- **Cycle de tour** : 60 secondes fixes par pion, géré directement par GameEngine (pas de classe séparée). Seul le client dont c'est le tour lance son propre timer local ; les autres attendent passivement le protocole `TURN_END`. Si le joueur actif se déconnecte en plein tour, Network détecte la perte de connexion localement et prévient GameEngine, qui traite ça comme un `TURN_END`.
- **Entrée** : données de toutes les autres entités (positions, arbre parsé, messages réseau reçus, …)
- **Sortie** : données pour les joueurs, l'affichage, le réseau (décide du déroulement du jeu, via `CollisionChecker` compare chaque point de la trajectoire reçue de Physics aux positions connues pour détecter une collision joueur/obstacle, décide de la fin de partie, des données à transmettre et à afficher)
- Ne s'occupe que d'une partie déjà lancée (ne gère pas le menu/salon)
- Si un joueur se déconnecte (même le créateur/hôte), un autre client devient l'hôte et la partie continue
- Fin de partie : comportement de base = déconnexion + retour au menu ; affichage victoire/défaite en bonus (méthodes prêtes côté Affichage)
- N'a pas besoin de savoir si les autres entités existent

### BotEngine (Moteur de Jeu Bot)

- **Entrée** : données du jeu (positions, obstacles)
- **Sortie** : formule mathématique en texte (ne calcule pas, décide juste d'une formule à jouer)
- La formule du bot passe par le même chemin que celle d'un joueur humain (Parsing → Physics)
- N'a pas besoin de savoir si les autres entités existent

## Synchronisation P2P du Moteur de Jeu Bot

- Les joueurs sont numérotés de 1 à n. À tour de rôle (dans l'ordre des numéros), un joueur décide du coup d'un bot ; les autres valident.
- Si un joueur se déconnecte, on passe simplement au numéro suivant, sans réattribuer son numéro.
- **Seed commune** : donnée par l'hôte au lancement de la partie, transmise une seule fois à tous les joueurs présents à ce moment.
- Les positions des joueurs sont calculées localement par chacun, puis partagées à tous en début de partie.
- **Vérification / resynchronisation** (`SyncChecker`) : à la fin de chaque tour. On compare les données utiles (la formule elle-même, les positions joueurs/obstacles, la liste des pions restants), pas les points calculés d'une fonction.
- En cas de désaccord entre deux joueurs à égalité (pas de majorité possible), le résultat retenu est celui du joueur avec le numéro le plus faible.

## Protocoles réseau (V1)

- **MainMenu** : `JOIN` (pseudo)
- **Room** : `ROOM_STATE` (état complet du salon, transporte directement les `Player`/`Bot`/`Pawn`), `READY` (signal vide)
- **Chat** : `CHAT_MESSAGE` (texte + pseudo), `CHAT_SYSTEM` (texte seul, pas de joueur associé)
- **GameEngine** :
    - `INITIAL_POSITIONS` — positions de tous les pions d'un joueur, envoyé une fois en début de partie
    - `SHOOT` — id du pion qui tire + formule texte ; envoyé **avant** que le tireur lance son propre calcul Parsing/Physics, pour minimiser le décalage perçu entre joueurs
    - `TURN_END` — id du pion qui va jouer au tour suivant
    - `SYNC_CHECK` — formule jouée + positions joueurs/obstacles + liste des pions restants ; envoyé automatiquement par tous (sans demande) à chaque fin de tour pour comparaison/resync

## Scénario de bout en bout validé (un tir)

1. Alice tape `x^2+3` et clique sur "Exécuter".
2. L'Affichage (GameScene) capte le clic et déclenche un callback vers GameEngine.
3. GameEngine envoie immédiatement le protocole `SHOOT` (id pion + formule) via Network à Bob — avant de lancer ses propres calculs, pour que le message voyage pendant qu'Alice calcule.
4. En parallèle, GameEngine appelle Parsing (texte → `Noeud`), puis Physics (`Noeud` → collection de `Point`).
5. GameEngine (via CollisionChecker) teste chaque point de la trajectoire pour une collision (joueur/obstacle), et envoie chaque point à l'Affichage au fur et à mesure pour un tracé progressif.
6. Bob reçoit le protocole `SHOOT` (via son `MessageListener` enregistré) et refait exactement le même chemin qu'Alice de son côté (Parsing → Physics → CollisionChecker → Affichage) — chaque client recalcule plutôt que de recevoir des points déjà calculés (choix assumé pour la V1, coût de calcul jugé négligeable).
7. Si le pion de Bob est touché, chaque client détecte la collision indépendamment de son côté.
8. À la fin du tour, tous les clients envoient automatiquement `SYNC_CHECK` (formule jouée, positions, pions restants) pour vérifier/harmoniser l'état entre eux via SyncChecker.
9. GameEngine envoie `TURN_END` avec l'id du prochain pion à jouer, et le tour suivant commence.

## Priorités de réalisation

1. **Parsing** — texte → arbre d'expression exploitable
2. **Physics** — arbre → collection de points
3. **GameEngine** — détection de collision, logique de partie (testable en local, un seul client)
4. **Network** — transmission des données entre clients, protocoles
5. **Affichage minimal** — visualiser la trajectoire pour valider visuellement
6. **MainMenu** — testable en CLI, connexion/protocoles de base
7. **Affichage complet** — vrais boutons, vraie UI, callbacks
8. **RoomManager** — nécessite l'Affichage complet
9. **BotEngine** — dernier, car optionnel et indépendant du reste (Chat, en bonus, peut se glisser à tout moment après Network/Affichage complet selon l'envie)

## Packages et classes

- **entities** : `Player`, `Bot` (extends `Player`, ajoute level), `Pawn`, `Obstacle`, `Point`
- **parsing** : `Parsing` (façade), `Node` (interface, publique), `OperationNode`, `VariableNode`, `FunctionNode`, `NumberNode` (package-private)
- **physics** : `Physics` (façade)
- **network** : `Network` (façade), `Message` (implements Serializable), `NetworkProtocol` (constantes statiques), `MessageListener` (interface fonctionnelle)
- **gameengine** : `GameEngine` (façade, gère aussi directement les tours), `CollisionChecker`, `SeedGenerator`, `SyncChecker`
- **chat** : `ChatManager` (façade)
- **mainmenu** : `MainMenu` (façade)
- **room** : `RoomManager` (façade), `Room` (données, sérialisable)
- **interface** : `InterfaceManager` (façade, gestion des scènes), `MainMenuScene`, `RoomScene`, `GameScene`
- **bot** : `BotEngine` (façade), `Bot` (voir entities)

## Prochaines étapes possibles

- Écrire les squelettes de classes vides (signatures de méthodes publiques uniquement) pour chaque package, dans l'ordre des priorités.
- Éventuellement, tracer un deuxième scénario (connexion au salon, ou fin de partie) pour renforcer la validation avant de coder.