# Mini projet 442 - streetfighter

## Objectifs

Créer un jeu streetfighter en réseau, impliquant deux cartes STM32F7-discovery (clients) ainsi qu'un serveur TCP fonctionnant sur un ordinateur en Python. Travail réalisé avec Gatien Séguy. 

## Etapes du projet

1. Créer des tâches de gestion des joueurs et d'affichage en local sur chaque carte. On utilise pour cela des variables globales décrivant la position des joueurs, s'ils sont en train de sauter, d'attaquer, ainsi que leur vie et enfin l'état global du jeu : joueur1/joueur2 KO, fin du jeu...
2. Créer une tâche d'envoi/réception des informations pertinentes à l'autre carte. Du point de vue de la carte 1, par exemple, on envoie la vie, l'état (attaque ou non), la position du joueur 1, et quelques variables globales du jeu. On reçoit alors de l'autre carte les mêmes informations concernant l'autre joueur.
3. Créer un script Python permettant de créer un serveur TCP (Transmission Control Protocol) pour "relier" les clients entre eux.

> L'ensemble des fonctions du jeu se trouvent dans les fichiers `freertos.c`, `main.c` (initialisation de l'écran) et `tcp_server.py`.

## 1. Tâches de gestion des joueurs

On a récupéré les images des joueurs, en position passive, d'attaque "jab" ou d'attaque spéciale "hadouken". On affiche une barre de vie pour chacun des joueurs. Pour éviter les conflits d'accès à l'écran, une seule tâche se charge de l'affichage des joueurs, s'exécutant toutes les 30ms.

![Capture d’écran 2025-04-29 à 16 31 avec arrière-plan supprimé 15](https://github.com/user-attachments/assets/c127e392-0107-4633-8864-477b801b693e)
![Capture d’écran 2025-04-29 à 16 32 avec arrière-plan supprimé 02](https://github.com/user-attachments/assets/8c6df4a0-680b-4c6d-b35b-b7e8238cd7c8)
![Capture d’écran 2025-04-29 à 16 32 avec arrière-plan supprimé 41](https://github.com/user-attachments/assets/58078017-60e1-47ae-a19a-2ccabf7c4cbc)
![Capture d’écran 2025-04-29 à 16 35 avec arrière-plan supprimé 04](https://github.com/user-attachments/assets/58bfb8c9-49fb-42db-ba98-39b5fda0b6ac)
![Capture d’écran 2025-04-29 à 16 35 avec arrière-plan supprimé 10](https://github.com/user-attachments/assets/4090d6cf-0e90-4884-bec7-9b75e762b64b)

Une autre tâche permet de récupérer les "inputs" de l'utilisateur. 
Le joystick permet le déplacement et les sauts :
Axe horizontal : déplacement gauche/droite selon la valeur lue via l’ADC.
Axe vertical : si la valeur dépasse un seuil, déclenche un saut (montée puis descente simulée par une durée).
Les boutons BP1 et BP2 permettent d’attaquer :
BP1 déclenche une attaque courte (jab), avec une durée d’attaque de 10.
BP2 déclenche une attaque longue (hadouken), avec une durée de 35 et un projectile.
Si le jeu est terminé, un appui sur BP1 après relâchement permet de le réinitialiser (vies, positions, états).


## 2. Tâche d'envoi et de réception des informations via TCP

La communication entre les cartes et le serveur Python repose sur le protocole TCP. La configuration nécessaire dans CubeMX étant complexe, on a dans ce projet réutilisé le travail de Pierrot Cadeilhan dont le projet se trouve [ici](https://github.com/pierrot-cadeilhan/miniprojet-LwIP/tree/main). On remarquera notament la nécessité de modifier le fichier `STM32F746NGHX_FLAHS.Id` afin de gérer l'allocation d'un espace mémoire suffisant dans la flash pour le bon fonctionnement du driver de la connection Ethernet `LwIP`. Ensuite, on initialise la connection au serveur une unique fois au démarrage dans la tâche `DefaultTask()`. Une fois la connection établie, on envoie et on récupère les informations de l'autre carte dans cette même tâche. 

> ### Remarque :
> Il est primordial de modifier au moins l'adresse MAC de l'une des cartes. Celles-ci étant programmables, par défaut les deux cartes ont la même adresse MAC : `00:80:E1:00:00:00`. Cela cause un conflit lors de l'affectation des adresses IP par le routeur (addressage DHCP), qui ne peut pas alors donner deux adresses IP différentes aux deux cartes.


## 3. Serveur Python

Celui-ci renvoie en permanence les informations qu'il reçoit selon le principe suivant. Lors de son lancement, le serveur se lance et attend la connection de 2 clients. Il faut cependant commencer par récupérer l'adresse IP du PC faisant tourner le serveur TCP pour pouvoir connecter les cartes. On fixe le Port de la connexion à 5000. Puisque, du côté des tâches, une seule tâche effectue l'envoi et la réception des données, on utilise un buffer au niveau du serveur comme suit :

* Réception de données de la carte 1, stockée dans le 1er emplacement du buffer ;
* Envoie du contenu du 2ème emplacement du buffer (i.e. les données de la 2ème carte) à la carte 1.

Cela permet d'éviter de filtrer les messages reçus par les cartes, même si par précaution, on prend soin de vérifier à la réception d'un packet que celui-ci provient bien de l'autre carte.

## Résultats

On obtient un streetfighter fonctionnant modérément bien. Le problème principal réside dans le fait que les calculs de perte de point de vie sont faits localement sur chaque carte, en se basant uniquement sur la position et l'état (attaque ou non) de l'autre joueur. Cela pose une multitude de problèmes de synchronisation. 

> On pourrait résoudre simplement ces problèmes en utilisant une architecture plus adaptée à des jeux en réseau :
> Le serveur reçoit uniquement les "inputs" des joueurs, effectue les calculs nécessaires (déplacement, perte de point de vie, fin du jeu, attente de RaZ...), et renvoie le résultat de ces calculs aux deux cartes pour l'affichage. Cela reste donc une piste d'amélioration importante pour notre jeu. 
