# 🐾 Smart Cat Flap — Système d'Ouverture Automatisé sous ESP32

![Langage](https://img.shields.io/badge/Langage-C%2FC%2B%2B-00599C?style=flat-square&logo=c%2B%2B)
![Plateforme](https://img.shields.io/badge/Plateforme-ESP32-E7352C?style=flat-square&logo=espressif)
![Simulation](https://img.shields.io/badge/Simulateur-Wokwi-4CAF50?style=flat-square)

Système embarqué de contrôle de chatière motorisée basé sur un microcontrôleur **ESP32**. Ce projet met en œuvre une **machine à états finis non-bloquante**, une **commande synchronisée de servomoteurs en opposition**, un **pilotage infrarouge/PIR**, ainsi qu'un **étage de filtrage acoustique RC** conçu pour le confort auditif de l'animal.

---

## 📋 Fonctionnalités principales

* **Double déclenchement (Automatique & Manuel) :**
  * **Capteur de mouvement (PIR) :** Ouverture automatique à l'approche de l'animal avec maintien en position haute pendant 10 secondes avant fermeture.
  * **Télécommande Infrarouge (Protocole NEC) :** Contrôle à distance permettant de déclencher un cycle de 10 s (`PLAY`), de verrouiller la trappe en position haute (`+`) ou de forcer la fermeture immédiate (`-`).
* **Cinématique symétrique ($+90^\circ / -90^\circ$) :** Pilotage synchronisé de deux servomoteurs fonctionnant en miroir autour d'un point de référence central ($0^\circ$ mécanique).
* **Mode Mécanique Non-Résistif (Sécurité Animale) :** Coupure matérielle de l'asservissement PWM (`detach()`) en position basse afin de relâcher le couple de maintien des moteurs et permettre une ouverture manuelle sans résistance électrique.
* **Synthétiseur Audio Multi-Profils & Filtrage Passe-Bas :**
  * 5 signatures acoustiques sélectionnables à la volée via la télécommande (Touches `1` à `5` : *Gazouillis félin, Carillon Do Majeur, Glissando, Quinte Zen, Ronronnement*).
  * Modulation du rapport cyclique (Duty Cycle abaissé à 14 %) combinée à un filtre passe-bas passif RC matériel pour éliminer les harmoniques impaires stridentes.

---

## 🛠️ Architecture Matérielle & Brochage (Pinout)

### Table d'interconnexion (ESP32)

| Composant | Broche Composant | GPIO ESP32 | Alimentation | Notes & Protections matérielles |
| :--- | :--- | :--- | :--- | :--- |
| **Capteur PIR** | `D` (Data Out) | `GPIO 2` | `5V` / `GND` | Niveau logique de sortie régulé à 3,3 V en interne. |
| **Récepteur IR** | `DAT` | `GPIO 4` | `5V` / `GND` | Décodage matériel protocole NEC. |
| **Servomoteur 1 (Gauche)** | `PWM` (Signal) | `GPIO 18` | `5V Externe` | Piloté à 50 Hz ($500\ \mu\text{s}$ à $2400\ \mu\text{s}$). |
| **Servomoteur 2 (Droite)** | `PWM` (Signal) | `GPIO 19` | `5V Externe` | Course inversée en miroir par rapport au Servo 1. |
| **Buzzer Passif** | `+` (Signal) | `GPIO 25` | `GND` | Interfacé via un filtre passe-bas RC ($R_1 = 330\ \Omega$, $C_1 = 470\text{ nF}$). |

### Électronique de puissance & Filtrage

1. **Découplage de l'alimentation (Servomoteurs) :**
   Les appels de courant au démarrage simultané des deux servomoteurs pouvant atteindre $1,5\text{ A}$, l'alimentation 5 V est découplée par un condensateur électrolytique réservoir ($C_2 = 1000\ \mu\text{F}$, 16 V) placé en parallèle au plus près des connecteurs moteurs pour éviter tout *Brownout Reset* de l'ESP32. Toutes les masses (`GND`) sont reliées en étoile au pôle négatif de l'alimentation.
2. **Filtre Passe-Bas RC (Sortie Audio GPIO 25) :**
   Pour lisser l'onde rectangulaire issue du périphérique `LEDC` et protéger l'étage de sortie CMOS (limitation $< 15\text{ mA}$), une cellule RC du 1er ordre est insérée entre le `GPIO 25` et le transducteur :

```text
                     R1 (330 Ω)
   GPIO 25 ────────────[====]────────────┬────────────── (+) Buzzer Passif
                                         │
                                      ┌──┴──┐
                                      │ C1  │ (470 nF)
                                      └──┬──┘
                                         │
   GND     ──────────────────────────────┴────────────── (-) Buzzer Passif
