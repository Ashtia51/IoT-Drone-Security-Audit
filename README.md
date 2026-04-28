# Audit de Sécurité IoT : Drone PNJ Galaxy HD
IoT-Audit : Analyse de la surface d'attaque et des risques de confidentialité (Drone PNJ)

## 📋 Sommaire
1. [I. Introduction](#i-introduction)
2. [II. Couche Dispositif (Device)](#ii-dispositif)
3. [III. Couche Réseau (Network)](#iii-reseau)
4. [IV. Couche de Soutien (Support)](#iv-soutien)
5. [V. Couche Application](#v-application)
6. [VI. Conclusion et axes d'amélioration](#vi-conclusion)

<a name="i-introduction"></a>
## I. Introduction

Ce projet a été réalisé dans le cadre de mon intérêt pour les objets connectés (IoT). J’ai donc pris la décision d’acheter un drone à bas prix pour l’analyser afin de savoir si les drones vendus en grandes surfaces en France étaient protégés. Pour information, ces drones sont accessibles à tout le monde mais recommandés à partir de 14 ans. Ainsi, des mineurs peuvent les utiliser.
Ce projet aura pour but d’analyser si ce drone est sécuritaire ou non et cela à tous les niveaux.

Matériel utilisé :
Un drone PNJ Galaxy HD acheté en grande surface, IoT grand public
Une télécommande de drone
Un téléphone Android 16 avec l’application PCAPdroid
Le logiciel Wireshark depuis un ordinateur Windows 11 comme OS

Méthodologie :
Je me réfère à l’International Telecommunication Union – ITU-T Y.4000. Cette norme définit l’architecture de l’IoT selon quatre couches distinctes, ce qui permet de structurer l'analyse de la surface d'attaque : 

Couche Dispositif (Device) : Analyse du matériel et du point d'accès Wi-Fi ouvert
Couche Réseau (Network) : Étude des protocoles de transport (UDP/TCP) et des flux via Wireshark
Couche de Soutien (Service/Support) : Analyse des communications
Couche Application : Application mobile et permissions

<a name="ii-dispositif"></a>
## II. Couche Dispositif (Device)

Le drone PNJ Galaxy HD est fabriqué en Chine par Guanxu Technology. Celui-ci dispose d’une caméra HD proposant du 780P pour les médias. Il est doté de différents capteurs pour la télémétrie. On peut les citer :

Le Capteur Vidéo / Caméra 
Le Capteur d’Orientation (Gyroscope / Accéléromètre) 
Le Capteur d’Altitude 

Nous n’avons pas de capteur GPS intégré. L'accès physique à ces composants est ouvert dès la mise sous tension, sans mécanisme d'authentification matériel.

**Aperçu du matériel :**
![Drone](Captures/physical_drone.png)

<a name="iii-reseau"></a>
## III. Couche Réseau (Network)

| Élément | Rôle | Adresse IP |
| :--- | :--- | :--- |
| **Drone (AP)** | Passerelle par défaut (Gateway A) | `192.168.1.1` |
| **Drone (AP)** | Passerelle par défaut (Gateway B) | `192.168.28.1` |
| **Smartphone** | Interface de capture PCAPdroid (VPN) | `10.215.173.1` |

La connexion au point d’accès du drone ne requiert aucune clé (Open Wi-Fi). L’absence de protocole WPA2/WPA3 permet une interception immédiate du trafic par n’importe quel attaquant à portée de signal.

Depuis l’application PCAPdroid, j’ai pu capturer les paquets avant de les exporter en.pcap. En analysant les logs sur Wireshark, je me suis aperçu de plusieurs points critiques. En effet, les services sont exposés en clair. Comme il n’y a pas de protocole TLS, plusieurs points critiques sont visibles :

Port 7777 (UDP) 
Flux de pilotage. Les commandes sont envoyées via des trames de 7 octets. L'absence de chiffrement permettrait à un attaquant d'injecter ses propres commandes pour détourner le drone. 

Ports 7070 et 7080 (TCP) 
Flux vidéo et commandes multimédias. Le flux est interceptable en temps réel, portant atteinte à la vie privée de l'utilisateur. 

### 📸 Preuves d'Audit et Analyses de Flux

Ci-dessous, les captures d'écran réalisées lors de l'interception des flux entre le drone et le smartphone (Android 16).

| Capture | Description Technique |
| :--- | :--- |
| ![info_wifi1](Captures/info_wifi1.png) | **Info WiFi 1** : Configuration initiale du point d'accès. |
| ![info_wifi2](Captures/info_wifi2.png) | **Info WiFi 2** : Détails des paramètres IP et DNS du drone. |
| ![first_analyse](Captures/first_analyse.png) | **Première Analyse** : Identification des premiers échanges TCP/UDP. |
| ![second_analyse](Captures/second_analyse.png) | **Seconde Analyse** : Confirmation de la stabilité des flux vidéos. |
| ![UDP_spam](Captures/UDP_spam.png) | **Déni de Service / Spam** : Observation du flux constant de paquets de pilotage. |
| ![payload](Captures/payload.jpg) | **Analyse Payload** : Structure de la trame de 7 octets (Commandes de vol). |

<a name="iv-soutien"></a>
## IV. Couche de Soutien (Service/Support)

Le drone possédant son propre réseau wifi sans accès à internet donc même en étant connecté en 4G/5G en plus du wifi, je n’ai pas pu retracer certaines informations liés au cloud. De plus, les communications doivent sûrement êtres chiffrées.
En l'absence de visibilité, le risque principal réside dans le transfert de données de géolocalisation (récupérées par l'application en Couche V) vers des serveurs situés hors de l'Union Européenne, sans garantie de protection conforme au RGPD.

<a name="v-application"></a>
## V. Couche Application

L’application auditée est « PNJ View » dans la version 1.4.7 via mon téléphone sous Android 16. L’application sert de terminal de visualisation (FPV), de panneau de commande et de gestionnaire de médias. 

Géolocalisation (ACCESS_FINE_LOCATION) : L'application exige l'accès au GPS du smartphone alors que le drone est dépourvu de module GPS. Cette donnée n'est pas nécessaire au vol mais est collectée par l'application.
Accès aux fichiers (READ_EXTERNAL_STORAGE) : Une permission étendue qui permet à l'application d'accéder à l'ensemble des photos et documents de l'utilisateur, bien au-delà des seuls médias capturés par le drone.
État du Wi-Fi (CHANGE_NETWORK_STATE) : Permet à l'application de manipuler les connexions sans intervention manuelle, exposant le téléphone aux réseaux non sécurisés du drone de manière automatique.

Nous pouvons donc en conclure que la collecte de données de localisation sans justification technique échappe au principe de minimisation des données (RGPD). 
Le drone étant accessible dès 14 ans, l'absence de chiffrement des flux (vu en Couche V) combinée à ces permissions intrusives expose une population vulnérable à des risques de détournement de données privées et de géolocalisation.

![Vue d'ensemble](Captures/overview.jpg)
![Interface Application](Captures/appli_pnj.jpg)

<a name="vi-conclusion"></a>
## VI. Conclusion et Axes d'Amélioration

Concernant le wifi on pourrait implémenter par défaut un chiffrement WPA2/WPA3 avec un mot de passe unique par appareil (inscrit sous le châssis) pour éviter les connexions non autorisées. 
Pour le firmware, mettre en place une signature au démarrage donc un secure boot pour éviter l’installation de logiciels malveillants.
En ce qui concerne les flux, instaurer des chiffrements via des protocoles sécurisés tels que TLS (commandes) et SRTP (pour les vidéos).
Les ports devraient êtres masqués et n'autoriser les connexions que depuis l'adresse IP du contrôleur légitime.
Supprimer la demande d'accès à la localisation GPS si le drone ne dispose pas de fonction GPS.
Adopter les standards d'Android 16  plutôt que de demander un accès complet au stockage externe.
Ajouter une mention claire dans l'application sur la destination des données de télémétrie envoyées vers la Couche de Soutien (Cloud). 



