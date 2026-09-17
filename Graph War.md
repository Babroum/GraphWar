# Graphwar - Structure du projet

## Notes générales

- Aucune persistance des données après fermeture du jeu ou en quittant une partie ; les données ne durent que le temps de la partie (option future : statistiques joueur en JSON).
- Le jeu est en **P2P**, aucun serveur hôte n'est nécessaire au jeu lui-même (tous les clients font le même calcul). Il y a un créateur de partie (l'hôte), mais il n'est pas le "chef d'orchestre" de la logique de jeu ; son rôle est transféré simplement au joueur suivant en cas de déconnexion, sans impact sur le déroulement.
- Pas de système de reconnexion pour l'instant : un joueur déconnecté quitte définitivement la partie en cours, personne ne reprend sa place.
- La communication entre l'Affichage et les autres domaines se fait par **callback** (l'Affichage prévient quand un événement se produit, sans appeler directement le domaine concerné) — pas de "Logique UI" séparée, pas de polling.
- Le jeu est organisé en 3 niveaux temporels autonomes, chacun parlant directement au Réseau avec ses propres protocoles : **App** (menu, connexion) → **Gestion du Salon** (page autonome) → **Moteur de Jeu** (partie en cours).
- Chaque fiche correspond à un **package Java**, avec une classe façade portant le nom du domaine (ex : `moteurjeu.MoteurJeu`). Les autres domaines n'appellent jamais directement une classe interne à un package qui n'est pas le leur — uniquement la façade.

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

- Des joueurs (chacun avec un ou plusieurs pions)
- Une partie
- Un terrain (zone 2D fixe, x de -25 à 25, y de -15 à 15)
- Des obstacles (cercles pleins à rayon variable, potentiellement destructibles plus tard)
- Un projectile / fonction mathématique

## Qui a besoin de savoir quoi ?

### App
- **Entrée** : saisies du menu (pseudo, IP, port, paramètres de création de salon)
- **Sortie** : initie la connexion réseau (rejoindre) ou crée le salon (hôte), puis passe la main à la Gestion du Salon
- Gère uniquement le menu d'accueil ; une fois connecté, ne gère plus rien du salon
- Testable en CLI (validation Réseau/protocoles de base)
- N'a pas besoin de savoir si les autres entités existent

### Gestion du Salon
- **Entrée** : inputs UI de la page salon (ajout de bots, équipes, nombre de personnages, chat), protocoles réseau du salon
- **Sortie** : mise à jour de l'affichage du salon, protocoles réseau envoyés (l'hôte envoie les modifs, reçoit les connexions ; les invités reçoivent tout, envoient juste "prêt" et le chat)
- Page UI autonome et indépendante d'App ; gère aussi sa propre fermeture (quitter le salon)
- Nécessite l'Affichage complet pour être testée
- N'a pas besoin de savoir si les autres entités existent

### Affichage
- **Entrée** : données du jeu (positions, texte, …)
- **Sortie** : affichage du jeu, déclenche des callbacks vers le domaine concerné lors d'un input (clic, texte tapé, …)
- N'a pas besoin de savoir si les autres entités existent

### Réseau
- **Entrée / Sortie** : messages taggés par un protocole (entier constant, ex `SET_NAME = 12`, classe statique dédiée), pour tous les contextes (App, Salon, Partie, Chat)
- Format de message : objet `Message implements Serializable` (protocole + données) ; une seule paire de méthodes d'envoi/réception générique, peu importe le nombre de protocoles
- Ne décide de rien, transmet uniquement ; c'est le domaine destinataire (App / Gestion du Salon / Moteur de Jeu / Chat) qui interprète le protocole via un switch
- Détecte aussi localement les pertes de connexion et prévient le domaine concerné (ex : joueur déconnecté en plein tour)
- N'a pas besoin de savoir si les autres entités existent

### Chat
- **Entrée** : message + expéditeur à transmettre (vers le Réseau), message + expéditeur reçu (depuis le Réseau)
- **Sortie** : callback vers l'Affichage pour ajouter le message à la chat box
- Domaine transversal, potentiellement partagé entre salon et partie (flexibilité d'implémentation)
- N'a pas besoin de savoir si les autres entités existent

### Parsing des données
- **Entrée** : formule du joueur/bot, message de chat, tout input UI
- **Sortie** : arbre d'expression exploitable
- Deux types de nœuds : opérations binaires (+, -, *, /, ^ — deux enfants) et fonctions unaires (sin, cos, sqrt — un seul enfant)
- Parenthèses gérées dès la V1 par la récursivité du parser (descente récursive)
- N'a pas besoin de savoir si les autres entités existent

### Physique / Trajectoire
- **Entrée** : arbre d'expression (formule déjà parsée)
- **Sortie** : collection de points (la trajectoire)
- Ne connaît pas les positions des joueurs/obstacles, ne détecte aucune collision
- N'a pas besoin de savoir si les autres entités existent

### Moteur de Jeu
- Chaque client fait tourner son propre jeu ; chaque moteur doit fonctionner à l'identique chez tous les joueurs, sinon désynchronisation.
- Pas de moteur serveur/client : tous les joueurs transmettent leurs données, et un moteur désynchronisé se resynchronise.
- Système de tour **par pion** (pas par joueur entier) : tous les pions n°1 jouent, puis tous les n°2, etc.
- Génère les positions (obstacles, pions) depuis la seed commune au lancement — simple méthode interne, appelée une fois par tous (ordre : nombre d'obstacles, puis position x/y et rayon pour chacun ; les obstacles peuvent se superposer sans contrainte). L'état d'un obstacle reste modifiable pour préparer la destructibilité future.
- **Cycle de tour** : 60 secondes fixes par pion. Seul le client dont c'est le tour lance son propre timer local ; les autres attendent passivement le protocole "tour terminé" (envoyé après action du joueur, ou à l'expiration des 60s). Si le joueur actif se déconnecte en plein tour, le Réseau détecte la perte de connexion localement et prévient le Moteur de Jeu, qui traite ça comme un "tour terminé".
- **Entrée** : données de toutes les autres entités (positions, arbre parsé, protocole réseau reçu, …)
- **Sortie** : données pour les joueurs, l'affichage, le réseau (décide du déroulement du jeu, compare chaque point de la trajectoire reçue de la Physique aux positions connues pour détecter une collision joueur/obstacle, décide de la fin de partie, des données à transmettre et à afficher)
- Ne s'occupe que d'une partie déjà lancée (ne gère pas le menu/salon)
- Si un joueur se déconnecte (même le créateur/hôte), un autre client devient l'hôte et la partie continue
- Fin de partie : comportement de base = déconnexion + retour au menu ; affichage victoire/défaite en bonus (méthodes prêtes côté Affichage)
- N'a pas besoin de savoir si les autres entités existent

### Moteur de Jeu Bot
- **Entrée** : données du jeu (positions, obstacles)
- **Sortie** : formule mathématique en texte (ne calcule pas, décide juste d'une formule à jouer)
- La formule du bot passe par le même chemin que celle d'un joueur humain (Parsing → Physique)
- N'a pas besoin de savoir si les autres entités existent

## Synchronisation P2P du Moteur de Jeu Bot

- Les joueurs sont numérotés de 1 à n. À tour de rôle (dans l'ordre des numéros), un joueur décide du coup d'un bot ; les autres valident.
- Si un joueur se déconnecte, on passe simplement au numéro suivant, sans réattribuer son numéro.
- **Seed commune** : donnée par l'hôte au lancement de la partie, transmise une seule fois à tous les joueurs présents à ce moment.
- Les positions des joueurs sont calculées localement par chacun, puis partagées à tous en début de partie.
- **Vérification / resynchronisation** : à la fin de chaque tour. On compare les données utiles (la formule elle-même, les positions joueurs/obstacles), pas les points calculés d'une fonction.
- En cas de désaccord entre deux joueurs à égalité (pas de majorité possible), le résultat retenu est celui du joueur avec le numéro le plus faible.

## Priorités de réalisation

1. **Parsing** — texte → arbre d'expression exploitable
2. **Physique** — arbre → collection de points
3. **Moteur de Jeu** — détection de collision, logique de partie (testable en local, un seul client)
4. **Réseau** — transmission des données entre clients, protocoles
5. **Affichage minimal** — visualiser la trajectoire pour valider visuellement
6. **App** — testable en CLI, connexion/protocoles de base
7. **Affichage complet** — vrais boutons, vraie UI, callbacks
8. **Gestion du Salon** — nécessite l'Affichage complet
9. **Moteur de Jeu Bot** — dernier, car optionnel et indépendant du reste (Chat, en bonus, peut se glisser à tout moment après le Réseau/Affichage complet selon l'envie)

## Prochaines étapes possibles

- Squelettes de classes façades vides pour chaque package, pour valider que le découpage tient en vrai code.
- Détail exact des données transportées par chaque protocole (quels champs pour quel numéro).
- Décider si un exemple complet tracé de bout en bout (scénario `x^2+3` à travers toutes les fiches) est utile avant de coder.