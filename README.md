<a id="francais"></a>

# 📡 Hardware Hacking d'un Routeur — TP-Link TL-WR841N V14.1

🇫🇷 Français (ce document) · 🇬🇧 [English version ↓](#english)

Mon premier projet de hardware hacking : j'ai acheté un TP-Link TL-WR841N V14.1 sur eBay. Au départ, je ne connaissais ni le routeur, ni la marque, ni vraiment ce qu'il y avait à l'intérieur — je m'étais juste renseigné pour en acheter un abordable. L'objectif : **apprendre un maximum de choses** — lecture de PCB, UART, dump de mémoire flash, extraction et analyse de firmware. Tout est documenté étape par étape (avec photos) comme un mini-tuto, pour que quelqu'un qui n'a jamais rien fait de tout ça puisse comprendre et reproduire.

> 🚧 **Ce dépôt est encore en cours.** Il n'y a pas de conclusion. Les constats 2 et 3 ne sont pas encore écrits. Le reverse de BusyBox dans Ghidra continue dans le **prochain commit**.

![Skill](https://img.shields.io/badge/Skill-Hardware%20Hacking-orange)
![Skill](https://img.shields.io/badge/Skill-Soldering-blue)
![Skill](https://img.shields.io/badge/Skill-UART-blue)
![Skill](https://img.shields.io/badge/Skill-Firmware%20Analysis-green)
![Tool](https://img.shields.io/badge/Tool-Binwalk-lightgrey)
![Tool](https://img.shields.io/badge/Tool-CH341A-lightgrey)
![Tool](https://img.shields.io/badge/Tool-PulseView-lightgrey)
![Target](https://img.shields.io/badge/Target-TL--WR841N%20V14.1-red)
![Status](https://img.shields.io/badge/Status-En%20cours-yellow)

> ⚠️ **Avertissement** : tout ce projet est réalisé sur **mon propre matériel**, dans un cadre de **labo personnel et d'apprentissage**. Ne testez jamais ces techniques sur un appareil qui ne vous appartient pas.

---

## 📑 Sommaire

1. [Matériel utilisé](#materiel)
2. [C'est quoi un routeur ?](#routeur)
3. [C'est quoi TP-Link ?](#tp-link)
4. [Étape 1 — Premier contact : l'interface web](#etape-1)
5. [Étape 2 — Ouverture du routeur et observation du PCB](#etape-2)
6. [Étape 3 — Repérer l'UART avec un analyseur logique](#etape-3)
7. [Étape 4 — Shell avec un adaptateur UART–TTL](#etape-4)
8. [Étape 5 — Dump de la flash SPI](#etape-5)
9. [Étape 6 — Analyse du firmware avec Binwalk](#etape-6)
10. [Étape 7 — Exploration du système de fichiers et constats](#etape-7)
11. [Glossaire](#glossaire)

---

<a id="materiel"></a>

## 🧰 Matériel utilisé

Voici **tout le matériel que j'ai acheté** pour ce projet, avec le lien et une petite explication. Les prix sont **ceux que j'ai payés**. Ils peuvent varier : le même article ne coûte pas pareil selon le moment et la boutique. Certains articles peuvent aussi ne plus être en vente : dans ce cas, il faut acheter un truc similaire.

> 💡 Le routeur (la cible) a été acheté **12 € sur eBay, hors frais de livraison**. Ce prix n'est pas dans le total du matériel ci-dessous.

### La cible

| Matériel                                                       | À quoi ça sert                                                                  | Prix                  | Lien          |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------- | --------------------- | ------------- |
| **TP-LINK TL-WR841N — 300 Mbps Wireless N, neuf sous blister** | Le routeur que j'ouvre, que je dump et que j'analyse. C'est la cible du projet. | 12 € (hors livraison) | *Pas de lien* |

### Ouverture du boîtier

| Matériel                                | À quoi ça sert                                                                                      | Prix    | Lien                                          |
| --------------------------------------- | --------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------- |
| **Kit tournevis de précision 120 en 1** | Ouvrir le routeur et dévisser les petits vis d'électronique (Torx, Phillips, etc.) sans les abîmer. | 10,99 € | [Amazon](https://www.amazon.fr/dp/B09WKQXKVV) |
| **Pince coupante de précision**         | Couper proprement des fils, des pattes de composants ou de la tresse à dessouder.                   | 6,99 €  | [Amazon](https://www.amazon.fr/dp/B09SL2TCH7) |

### Mesure électrique

| Matériel                                          | À quoi ça sert                                                                                                                                    | Prix    | Lien                                          |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------- |
| **Multimètre numérique KAIWEETS**                 | Mesurer la tension (par ex. 3,3 V sur l'UART), la continuité d'une piste, une résistance… L'outil de base pour ne pas brancher n'importe comment. | 15,99 € | [Amazon](https://www.amazon.fr/dp/B08CX9W7G3) |
| **Kit de pointes / cordons silicone 1000 V 20 A** | Des cordons plus souples et plus sûrs pour le multimètre (mieux que ceux fournis d'origine).                                                      | 6,29 €  | [Amazon](https://www.amazon.fr/dp/B0D7ZT2PH2) |

### Signaux et console série

| Matériel                                     | À quoi ça sert                                                                                                                           | Prix    | Lien                                          |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------- |
| **Analyseur logique USB 8 canaux 24 MHz**    | Enregistrer les signaux 0/1 sur les broches (UART, SPI…). Avec PulseView, on voit le TX, on trouve le baudrate, on vérifie que ça parle. | 11,80 € | [Amazon](https://www.amazon.fr/dp/B0D8Q886W9) |
| **Adaptateur USB → TTL FTDI (DTECH, 3,3 V)** | Relier le PC au port UART du routeur pour essayer d'obtenir un shell (console série). Puce FT232RNL, niveau 3,3 V.                       | 11,96 € | [Amazon](https://www.amazon.fr/dp/B0DXZ5Q13S) |

### Dump de la mémoire flash

| Matériel                                   | À quoi ça sert                                                                                                                                  | Prix    | Lien                                          |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------- |
| **Kit programmateur CH341A + pince SOIC8** | Lire (dumper) la puce flash SPI qui contient le firmware. La pince se clipse sur la puce ; le support 8 pattes sert une fois la puce dessoudée. | 11,49 € | [Amazon](https://www.amazon.fr/dp/B0H296SJB6) |

### Soudure et dessoudage

| Matériel                                 | À quoi ça sert                                                                                        | Prix    | Lien                                                               |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------ |
| **Fer à souder 80 W (180–520 °C)**       | Souder / ressouder des composants.                                                                    | 23,98 € | [Amazon](https://www.amazon.fr/dp/B09B3GRVTM)                      |
| **Station air chaud Podofo 858D 700 W**  | Dessouder une puce à plusieurs pattes (comme la flash SOIC8) avec de l'air chaud, sans tout arracher. | 23,35 € | [AliExpress](https://fr.aliexpress.com/item/1005013019903116.html) |
| **Tresses à dessouder (4 largeurs)**     | « Aspirer » l'étain fondu pour nettoyer les pastilles du PCB après un dessoudage.                     | 8,99 €  | [Amazon](https://www.amazon.fr/dp/B0GWF24LT8)                      |
| **Flux liquide no-clean #923 (30 mL)**   | Aide l'étain à bien mouiller les pattes. Sans flux, la soudure est souvent moche ou froide.           | 7,89 €  | [Amazon](https://www.amazon.fr/dp/B0G2GTRDBG)                      |
| **Alcool isopropylique 99,9 % (750 mL)** | Nettoyer le flux, les traces de doigts et la poussière sur le PCB une fois le travail fini.           | 7,70 €  | [Amazon](https://www.amazon.fr/dp/B0CK2FX99D)                      |

### Poste de travail et sécurité

| Matériel                                        | À quoi ça sert                                                                               | Prix    | Lien                                          |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------- | ------- | --------------------------------------------- |
| **Tapis de soudure silicone 30×20 cm (500 °C)** | Protège le bureau de la chaleur, et les petits compartiments évitent de perdre vis et puces. | 9,99 €  | [Amazon](https://www.amazon.fr/dp/B0CRB58Q93) |
| **Lunettes de protection**                      | Protègent les yeux des projections d'étain et de l'air chaud.                                | 10,19 € | [Amazon](https://www.amazon.fr/dp/B0FC6962G9) |
| **Masques FFP2 (lot de 50)**                    | Limitent l'inhalation des fumées de soudure / flux. À porter dès qu'on chauffe.              | 11,89 € | [Amazon](https://www.amazon.fr/dp/B0FS1Y3Q9Z) |

### Pour la suite du projet

| Matériel                           | À quoi ça sert                                                                                             | Prix    | Lien                                          |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------- |
| **Raspberry Pi Pico 2 W (RP2350)** | Microcontrôleur pour du **glitching** ou du **side-channel**. **Pas nécessaire pour le dump** du firmware. | 13,67 € | [Amazon](https://www.amazon.fr/dp/B0DQ1555PJ) |

> 💡 **Glitching**, c'est quoi ? On perturbe volontairement une puce pendant une fraction de seconde (une chute de tension, un « coup » sur l'horloge) au moment où elle fait un contrôle. Si le timing est bon, elle peut sauter ce contrôle. **Side-channel**, c'est l'inverse : on ne casse rien, on **observe** ce qui fuit autour (consommation électrique, temps de réponse, rayonnement) pendant qu'elle travaille, pour en déduire des informations qu'elle ne montre pas directement. Le Pico sert de déclencheur précis et bon marché pour ce genre d'essais. Le dump de la flash, lui, se fait avec le CH341A.

### Ce que j'ai payé

|                                            |              |
| ------------------------------------------ | ------------ |
| **Total du matériel listé (hors routeur)** | **193,16 €** |
| **Routeur (eBay, hors livraison)**         | **12 €**     |

> Ce sont les prix au moment où je les ai achetés. Le même matériel peut coûter plus ou moins cher ailleurs, ou plus tard. Si un article n'est plus disponible, un modèle similaire fait l'affaire. Ça reste un **premier labo hardware** tout à fait abordable : on n'a pas besoin d'un oscilloscope à 2000 € pour commencer.

**Logiciels :**

| Logiciel                                                                             | Usage                                                                                       |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| [PulseView / sigrok](https://github.com/sigrokproject/pulseview)                     | Logiciel libre pour visualiser les signaux de l'analyseur logique                           |
| [NeoProgrammer](https://github.com/GioLangLe/CH341B-NeoProgramer)                    | Logiciel de lecture de la flash avec le CH341A                                              |
| [PuTTY](https://www.putty.org/)                                                      | Terminal série utilisé pour tenter un shell sur l'UART                                      |
| [Machine virtuelle Kali Linux](https://www.kali.org/get-kali/#kali-virtual-machines) | Environnement d'analyse, os pour pentesteur outils préinstallés                             |
| [Binwalk](https://github.com/ReFirmLabs/binwalk)                                     | Analyse et extraction du firmware. Installé par défaut sur Kali, sinon il faut l'installer. |
| [Ghidra](https://github.com/NationalSecurityAgency/ghidra)                           | Ouvrir un programme compilé (ici BusyBox) pour le lire en assembleur et en C                |

---

<a id="routeur"></a>

## 🌐 C'est quoi un routeur ?

Un **routeur** fait passer les données d'un réseau à un autre. Il regarde l'adresse de destination de chaque paquet et choisit le chemin.

La boîte Wi-Fi à la maison, comme ce TL-WR841N, est un routeur **avec** d'autres fonctions en plus : le Wi-Fi, les 4 ports Ethernet, la distribution des adresses IP (DHCP) et un pare-feu basique. La fibre ou la ligne téléphonique arrivent sur la box ou le modem de l'opérateur. Ce routeur se branche derrière, sur le port WAN.

Dedans, c'est aussi un **mini-ordinateur** :

- un **processeur / SoC** (le cerveau),
- de la **RAM** (mémoire de travail),
- une **mémoire flash** (équivalent du disque dur : elle stocke le système),
- des puces radio Wi-Fi, des ports Ethernet…
- un **système d'exploitation**, très souvent un **Linux embarqué**.

C'est ce qui le rend si intéressant pour apprendre le hacking hardware : on peut l'ouvrir, parler directement à son système et récupérer son logiciel interne (le **firmware**).

<a id="tp-link"></a>

## 🏢 C'est quoi TP-Link ?

**TP-Link** est un fabricant chinois d'équipements réseau (routeurs, switchs, répéteurs Wi-Fi, caméras, prises connectées…), fondé en 1996. C'est l'un des plus gros fabricants de routeurs grand public au monde, réputé pour ses prix très accessibles.

Le **TL-WR841N** est un de leurs modèles d'entrée de gamme les plus vendus : un routeur Wi-Fi N 300 Mbps avec 4 ports Ethernet. La mention **V14.1** correspond à la **révision matérielle** : un même nom de modèle peut cacher des composants internes totalement différents d'une version à l'autre. Il est donc important de toujours noter la version précise de son appareil.

Le routeur tel que je l'ai reçu, encore fermé, sur le bureau.

![TP-Link TL-WR841N reçu, boîtier fermé](images/hardware/0%20photo%20du%20routeur.jpeg)

*Le TL-WR841N à la réception, avant ouverture.*

---

<a id="etape-1"></a>

## 🔍 Étape 1 — Premier contact : l'interface web

Avant d'ouvrir quoi que ce soit, on commence par **collecter un maximum d'informations** sur l'appareil, tel qu'il est vendu.

1. Branchement et allumage du routeur.
2. Connexion à son interface d'administration via **[http://tplinkwifi.net](http://tplinkwifi.net)**.
3. Identifiants par défaut : **`admin` / `admin`**.

> 💡 **Pourquoi c'est important ?** Des identifiants par défaut aussi simples sont une faiblesse de sécurité classique : beaucoup d'utilisateurs ne les changent jamais.

Page de login, avant d'ouvrir le boîtier. L'en-tête indique déjà **300M Wireless N Router** et le modèle **TL-WR841N**. La version matérielle, lue une fois le routeur allumé, est **V14.1**.

![Page de login du routeur sur tplinkwifi.net](images/hardware/0%2C1%20page%20de%20login%20du%20routeur.png)

*Page de login (*`http://tplinkwifi.net`*). Identifiants par défaut :* `admin` */* `admin`*.*

---

<a id="etape-2"></a>

## 🔧 Étape 2 — Ouverture du routeur et observation du PCB

Le **PCB** (*Printed Circuit Board*) est la carte électronique sur laquelle tous les composants sont soudés. Après avoir ouvert le boîtier, j'ai observé en détail la carte pour **identifier chaque puce** :

| Composant              | Référence                               | Rôle                                                                                                   |
| ---------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| SoC (processeur / CPU) | MediaTek **MT7628NN**                   | Cerveau du routeur. Le Wi-Fi 2,4 GHz est intégré à cette puce : pas de puce radio à part sur la carte. |
| RAM                    | Zentel **A3S56D40GTP-50L**              | Mémoire de travail                                                                                     |
| Mémoire flash SPI      | GigaDevice **GD25Q32** (32 Mbits, 4 Mo) | Stocke le firmware                                                                                     |
| UART                   | 4 pads (VCC, GND, RX, TX)               | Console série, à côté de la flash                                                                      |

> 💡 **Astuce :** les références imprimées sur les puces (par exemple `25Q32…`) permettent de retrouver leur **datasheet** sur Internet, qui indique le rôle de chaque broche.

J'ai également remarqué des **points de test et des pads** sur la carte, dont des broches qui ressemblent à un **port UART**.

Vue du dessus, boîtier ouvert. J'ai entouré en rouge les quatre éléments utiles pour la suite : la **RAM**, le **processeur**, la **flash SPI** et les **ports UART**.

![PCB vu du dessus, RAM, processeur, flash SPI et UART entourés en rouge](images/hardware/0%2C5%20photo%20routeur%20vue%20du%20dessus.jpeg)

*Vue du dessus. En rouge : RAM (Zentel), processeur (MediaTek MT7628NN), flash SPI et pads UART.*

Je n'ai pas de photo **pendant** le dessoudage. Les gros plans ci-dessous sont la flash, encore soudée sur la carte (repère **U4**).

![Gros plan de la puce flash encore soudée, repère U4](images/hardware/1%20premiere%20photo%20de%20la%20puce.jpeg)

*Première photo de la flash, encore sur le PCB (repère U4, boîtier 8 pattes).*

![Autre gros plan de la flash, pads UART à gauche](images/hardware/3%20secondes%20photo%20de%20la%20puce.jpeg)

*Autre angle. À gauche, le connecteur série est déjà visible (VCC, GND, RX, TX).*

---

<a id="etape-3"></a>

## 📈 Étape 3 — Repérer l'UART avec un analyseur logique

### C'est quoi l'UART ?

L'**UART** (*Universal Asynchronous Receiver/Transmitter*) est un protocole de communication série très simple. Sur beaucoup d'appareils électroniques, les constructeurs laissent un port UART utilisé en développement pour **afficher les messages de démarrage** et parfois donner accès à une **console (shell)**.

Un port UART comporte en général 4 broches :

| Broche  | Rôle                                                         |
| ------- | ------------------------------------------------------------ |
| **VCC** | Alimentation (3,3 V) — *on ne la branche généralement pas !* |
| **GND** | Masse (référence 0 V)                                        |
| **TX**  | Le routeur **émet** (envoie des données)                     |
| **RX**  | Le routeur **reçoit**                                        |

### C'est quoi un analyseur logique ?

C'est un outil qui **enregistre l'état (0 ou 1) des signaux électriques** au cours du temps. Avec le logiciel libre **[PulseView](https://github.com/sigrokproject/pulseview)** (projet [sigrok](https://sigrok.org/wiki/PulseView)), on peut voir ces signaux, décoder le protocole et ainsi **trouver quelle broche est le TX**, confirmer la vitesse de transmission (**baudrate**) et vérifier que ça communique bien.

Sur la vue du dessus, les pads UART sont entourés en rouge, juste à côté de la flash. Le gros plan ci-dessous montre les quatre noms sérigraphiés, puis le test UART en physique.

![Flash U4 et pads UART VCC GND RX TX](images/hardware/2%20seconde%20photo%20de%20la%20puce%20on%20voit%20aussi%20les%20ports%20uart.jpeg)

*La flash (U4) et, à gauche, les pads UART : **VCC**, **GND**, **RX**, **TX**.*

![Test UART en physique, pinces sur les pads](images/hardware/0%2C7%20test%20uart.jpeg)

*Test UART en physique. Les pinces sont sur les pads.*

Dans PulseView, l'analyseur apparaît sous le nom **Saleae Logic**. Le signal utile est sur **D0**. Les autres canaux restent plats. J'enregistre à **1 MHz**.

![PulseView : signal sur D0, pas encore de décodeur](images/software/0%2C1%20on%20voit%20le%20signal%20sur%20d0.png)

*Premier enregistrement. D0 bouge, les autres canaux ne bougent pas. Pas encore de décodeur.*

![Zoom PulseView sur les créneaux de D0](images/software/0%2C2%20on%20zoome%20et%20on%20voit%20les%20cr%C3%A9neaux.png)

*Zoom. On voit les créneaux : c'est une ligne série.*

J'ajoute le décodeur UART sur D0 : **115200** bauds, 8 bits, sans parité, 1 bit de stop. Au début, PulseView affiche une erreur.

![Décodeur UART 115200 sur D0, PulseView signale une erreur](images/software/0%2C3%20on%20ajoute%20le%20d%C3%A9codeur%20uart%20mais%20erreur.png)

*Décodeur UART sur D0, 115200. PulseView signale une erreur.*

Juste après, le même canal se décode en texte.

![Texte UART décodé : ntp connect failed](images/software/0%2C4%20le%20d%C3%A9codeur%20affiche%20ntp%20connect%20failed.png)

*Le texte apparaît :* `ntp connect failed`*.*

À la capture 0,5, je mets plus de samples : je passe à **50 millions**, pour voir plus loin dans le démarrage. La vitesse reste **1 MHz**.

![Capture longue du boot avec le décodeur UART](images/software/0%2C5%20capture%20longue%20du%20d%C3%A9marrage.png)

*50 millions d'échantillons. On voit le boot plus loin. D0 parle par paquets, le décodeur est en dessous.*

![UART décodé : U-Boot 1.1.3 du 19 mars 2018](images/software/0%2C6%20on%20lit%20u-boot%201.1.3%20mars%202018.png)

`U-Boot 1.1.3 (Mar 19 2018)`*. Le même bootloader que Binwalk trouvera dans le firmware.*

![UART : la flash est reconnue au démarrage](images/software/0%2C7%20la%20flash%20est%20reconnue%20au%20boot.png)

*La flash est reconnue au démarrage :* `device id 0x16 find flash`*.*

![UART : Ralink UBoot Version 4.3.0.0](images/software/0%2C8%20on%20lit%20ralink%20uboot%204.3.0.0.png)

`Ralink UBoot Version: 4.3.0.0`*. Ralink, c'est l'ancien nom qu'on retrouve encore dans le boot du SoC MediaTek.*

![UART : le boot parle du MT7628](images/software/0%2C9%20le%20boot%20parle%20du%20mt7628.png)

*Le boot parle du **MT7628**.*

![Suite du journal de boot UART](images/software/0%2C10%20suite%20du%20journal%20de%20boot.png)

*La suite du journal : composants et tailles en octets.*

![UART : 32 Mo de RAM et flash SPI](images/software/0%2C11%2032%20mo%20de%20ram%20et%20flash%20spi.png)

`Total memory 32 Mbytes`*, et la flash est en **SPI**.*

![UART : date de la flash en 2018](images/software/0%2C12%20date%20de%20la%20flash%202018.png)

*Date de la flash : **2018**.*

![UART : réglages Ethernet et Wi-Fi](images/software/0%2C13%20r%C3%A9glages%20ethernet%20et%20wifi.png)

*Réglages Ethernet et Wi-Fi au boot.*

![UART : Uncompressing Kernel Image](images/software/0%2C14%20uncompressing%20kernel%20image.png)

`Uncompressing Kernel Image`*. U-Boot décompresse le noyau.*

![UART : Booting image at bc010000](images/software/0%2C15%20booting%20image%20at%20bc010000.png)

`Booting image at bc010000`*.*

![UART : do_bootm lance le noyau](images/software/0%2C16%20le%20noyau%20d%C3%A9marre%20do_bootm.png)

`do_bootm`*. Le noyau est lancé.*

---

<a id="etape-4"></a>

## 🖥️ Étape 4 — Shell avec un adaptateur UART–TTL

Après PulseView, j'ai enlevé l'analyseur logique. Je passe à l'adaptateur **USB–TTL** et au terminal **PuTTY** : il traduit les signaux du routeur en un port série virtuel.

Sur la photo, les pinces sont reliées comme ça : **GND sur GND**, **RX sur TX** et **TX sur RX**. Les deux fils de données sont croisés : ce que le routeur envoie (son TX), l'adaptateur doit le recevoir (son RX), et l'inverse. Le **VCC** reste débranché. L'adaptateur ne doit pas alimenter la carte.

![Test UART : pinces GND sur GND, RX sur TX et TX sur RX](images/hardware/0%2C7%20test%20uart.jpeg)

*Test UART. GND sur GND, RX sur TX et TX sur RX.*

Dans PuTTY, la session est en **Serial**, sur **COM4**, à **115200**.

![PuTTY en Serial, COM4, 115200](images/software/0%2C16%2C5%20configuration%20putty%20com4%20115200.png)

*Réglages PuTTY : Serial, COM4, 115200.*

Ensuite j'ai spammé **Entrée** jusqu'à voir l'invite. La fenêtre s'appelle `COM4 - PuTTY`. L'invite est `#` : je suis root.

![PuTTY : fin du boot et invite root après avoir spammé Entrée](images/software/0%2C17%20shell%20ouvert%20en%20spammant%20entr%C3%A9e.png)

*Fin du boot, puis l'invite. C'est là que j'ai spammé Entrée.*

![PuTTY : ls à la racine du routeur](images/software/0%2C18%20ls%20de%20la%20racine.png)

`ls` *à la racine :* `bin`*,* `dev`*,* `etc`*,* `lib`*,* `linuxrc`*,* `mnt`*,* `proc`*,* `sbin`*,* `sys`*,* `usr`*,* `var`*,* `web`*.*

![PuTTY : cat passwd dans etc](images/software/0%2C19%20cat%20passwd%20dans%20etc.png)

*Dans* `etc`*,* `cat passwd`*. Le compte **admin** a un hash MD5-crypt et l'uid **0**.*

![PuTTY : listing de /proc](images/software/0%2C20%20listing%20de%20proc.png)

*Contenu de* `/proc`*.*

![PuTTY : partitions mtd boot kernel rootfs config radio](images/software/0%2C21%20partitions%20boot%20kernel%20rootfs%20config%20radio.png)

`cat /proc/mtd`*. Cinq partitions :* `boot`*,* `kernel`*,* `rootfs`*,* `config`*,* `radio`*.*

`dd` n'existe pas sur le routeur. La copie vers `/tmp` échoue aussi : ce dossier n'existe pas. Je copie donc vers `/var`.

![PuTTY : dd not found](images/software/0%2C22%20dd%20not%20found.png)

`dd if=/dev/mtd4 of=/tmp/radio.bin` *→* `dd: not found`*.*

![PuTTY : le dossier tmp n'existe pas](images/software/0%2C23%20tmp%20inexistant.png)

`cat /dev/mtd4 > /tmp/radio.bin` *→* `/tmp` *n'existe pas.*

![PuTTY : radio.bin, uboot.bin et config.bin dans /var](images/software/0%2C24%20copies%20radio%20uboot%20config%20dans%20var.png)

*Les copies passent par* `/var` *:* `radio.bin`*,* `uboot.bin` *et* `config.bin`*, 65536 octets chacun.*

J'essaie ensuite de les envoyer au PC avec **TFTP**. `tftp -h` et `tftp --help` sont refusés. L'envoi de `radio.bin` ne part pas non plus : timeout vers `192.168.0.100`, puis `network unreachable` vers `192.168.56.1`. Cette deuxième adresse est l'IP host-only par défaut de VirtualBox. Le routeur n'atteint pas le PC : c'est un problème de configuration réseau, pas une limite de TFTP.

![PuTTY : tftp refuse -h et --help](images/software/0%2C25%20tftp%20aide%20refus%C3%A9e.png)

`tftp -h` *et* `tftp --help` *sont refusés.*

![PuTTY : tftp timeout puis network unreachable](images/software/0%2C26%20tftp%20timeout%20et%20r%C3%A9seau%20injoignable.png)

*Envoi de* `radio.bin` *: timeout vers* `192.168.0.100`*, puis* `network unreachable` *vers* `192.168.56.1`*.*

**Résultat :** le shell root s'ouvre. Les fichiers restent sur le routeur, parce que le réseau vers le PC ne passe pas.

---

<a id="etape-5"></a>

## 💾 Étape 5 — Dump de la flash SPI

Le shell était ouvert. Les fichiers n'ont pas quitté le routeur : le TFTP tombait sur un timeout, puis sur un réseau injoignable. Je lis donc **directement la mémoire flash**, celle qui contient le firmware. On appelle ça faire un **dump**.

### Les outils

- **Mémoire flash SPI** : la puce (souvent en boîtier **SOIC8**, 8 pattes) qui stocke le système du routeur.
- **CH341A** : un petit programmateur USB bon marché pour lire et écrire ces puces.
- **Pince SOIC8** : une pince qui s'accroche sur la puce pour la lire **sans la dessouder**.
- **NeoProgrammer** : le logiciel qui pilote le CH341A.

### Tentative 1 — Lecture « en circuit » avec la pince

J'ai branché la pince SOIC8 sur la flash, toujours soudée dans le routeur, puis reliée au CH341A, et essayé de lire avec NeoProgrammer.

**Problème :** ça ne fonctionnait pas. Dès que je branchais, la **LED du CH341A baissait en luminosité**, comme si le programmateur alimentait aussi le reste de la carte.

> 💡 **Explication :** quand on lit une puce encore soudée sur son circuit, le courant du programmateur peut alimenter les autres composants (SoC, etc.) qui se mettent à « parler » sur le même bus. Cela crée des **interférences** et la lecture échoue.

### Tentative 2 — Dessoudage de la puce ✅

J'ai opté pour une méthode plus propre : **dessouder la puce flash** pour l'isoler complètement du reste de la carte, puis la **ressouder sur un support 8 pattes** (fourni dans le kit CH341A). Brancher ce support **directement** sur le CH341A n'a pas fonctionné. Avec la **pince** clipsée sur la puce, ça a fonctionné : j'ai pu **dumper le firmware**.

### Vérification

Pour être sûr que le dump est fiable, j'ai **refait une seconde lecture** et comparé les deux fichiers : aucune différence → le dump est bon ✅.

La puce lue par NeoProgrammer est une **GigaDevice GD25Q32**, SPI NOR, 3,3 V, 32 Mbits (4 194 304 octets).

Pas de photo **pendant** le dessoudage à l'air chaud. On reprend juste après : la puce est sortie, le plastique du boîtier est protégé avec de l'aluminium.

![PCB après dessoudage, aluminium autour du plastique](images/hardware/4%20photo%20apres%20dessoudage%20puce%20utilisation%20de%20aluminium%20pour%20proteger%20le%20pastique.jpeg)

*Après le dessoudage. L'aluminium protège le plastique du boîtier de la chaleur.*

![Flash tenue à la pince, empreinte U4 vide](images/hardware/5%20photo%20apres%20dessoudage%20puce.jpeg)

*La flash, sortie de la carte. L'empreinte **U4** est vide.*

Avant de la ressouder, je nettoie les pattes et le support à l'alcool isopropylique.

![Alcool isopropylique 99,9 % et cotons-tiges](images/hardware/6%20nettoyage%20du%20support%20de%20la%20puce.jpeg)

*Nettoyage : alcool isopropylique 99,9 % et cotons-tiges. Le flux liquide est là aussi.*

Puis je la ressoude sur le support 8 pattes du kit CH341A.

![Flash soudée sur l'adaptateur SOIC8](images/hardware/7%20puce%20souder%20sur%20le%20support%20.jpeg)

*Flash soudée sur l'adaptateur. Les pastilles numérotées (1 à 8) servent à ne pas se tromper de patte.*

![Autre angle de la flash soudée sur l'adaptateur](images/hardware/8%20autre%20angle%20de%20la%20puce%20souder.jpeg)

*Autre angle. Le support a des broches en dessous : l'idée était de l'enficher directement sur le programmateur.*

J'ai d'abord branché ce support **directement** sur le CH341A. Le sens compte : la puce a un point (repère de la patte 1), et sur ce montage **la patte 1 doit être en haut à droite**. Ça n'a pas fonctionné. La soudure devait être mauvaise : le contact entre la puce et le support ne passait pas.

![Support enfiché directement sur le CH341A, patte 1 en haut à droite](images/hardware/9%20je%20montre%20le%20sens%20le%201%20doit%20etre%20en%20haut%20a%20droite.jpeg)

*Support branché directement sur le CH341A. Repère : la patte **1** en haut à droite. Cette lecture n'a pas marché.*

J'ai donc clipé la pince SOIC8 sur la puce, déjà soudée sur le support, et relié la nappe au programmateur. Là, ça a fonctionné.

![Pince SOIC8 sur la flash, reliée au CH341A](images/hardware/10%20branche%20la%20pince%20sur%20la%20puce%20.jpeg)

*La pince sur la puce, à la place du branchement direct. Le CH341A est à gauche, relié en USB.*

![Nappe de la pince sur le support du CH341A](images/hardware/11%20autre%20angle%20du%20support%20de%20la%20pince%20sur%20le%20ch314a.jpeg)

*Autre angle : la nappe de la pince arrive sur le support du CH341A.*

![Second angle du CH341A avec la nappe branchée](images/hardware/12%20autre%20angle%20du%20support%20de%20la%20pince%20sur%20le%20ch314a.jpeg)

*Même branchement, autre angle. C'est dans cette configuration que le dump a fonctionné.*

### Lecture avec NeoProgrammer

Au lancement, le CH341A est vu (« CH341 Black ») mais la puce ne répond pas encore : il faut d'abord choisir le bon modèle.

![NeoProgrammer au lancement, puce non détectée](images/software/1%20on%20lance%20neoprogrammer.png)

*NeoProgrammer 2.2.0.10 au démarrage. Le programmateur est là, la puce ne répond pas.*

Je clique sur **Search** et je cherche la référence lue sur le boîtier.

![Recherche de puce GD25Q32 dans NeoProgrammer](images/software/2%20on%20clique%20sur%20search.png)

*Recherche : trois puces GigaDevice de 32 Mbits.*

![GD25Q32 et GD25Q32B, toutes deux 3,3 V et 32 Mbits](images/software/3%20les%20deux%20qui%20nous%20int%C3%A9resse.png)

*Les deux qui collent au marquage : **GD25Q32** et **GD25Q32B**, 3,3 V, 32 Mbits.*

Je sélectionne **GD25Q32 [3.3V]**. L'identifiant SPI lu est `C84016`, et la taille annoncée est 4 Mo.

![GD25Q32 3,3 V sélectionnée, SPI ID C84016](images/software/4%20selection%20de%20la%20bonne%20puce.png)

*GD25Q32 sélectionnée. SPI ID* `C84016`*, 4 194 304 octets.*

Puis **Read** : NeoProgrammer lit toute la flash.

![Juste avant la lecture de la flash](images/software/5%20on%20clique%20sur%20read%20pour%20extraire%20le%20fireware.png)

*Juste avant la lecture.*

![Lecture réussie, dump hexadécimal de 4 Mo](images/software/6%20succes%20de%20l'extraction%20du%20fireware.png)

*Lecture réussie. Le buffer fait 4 194 304 octets, CRC32* `0x9559AB35`*.*

Pour vérifier, je relance une lecture. Le fichier enregistré s'appelle `firmware.bin`.

![Vérification du dump firmware.bin](images/software/7%20v%C3%A9rification%20du%20fireware%20extrait%20pour%20etre%20sur.png)

*Seconde lecture, pour comparer.*

![Vérification réussie, firmware.bin prêt pour Kali](images/software/8%20v%C3%A9rification%20r%C3%A9ussit%20on%20glisse%20le%20fireware%20sur%20kali%20linux.png)

*Vérification réussie (*`Success`*). Je transfère ensuite* `firmware.bin` *sur la VM Kali.*

---

<a id="etape-6"></a>

## 🧩 Étape 6 — Analyse du firmware avec Binwalk

Le fichier obtenu est une **image brute** de la mémoire flash. J'ai transféré ce fichier sur une **VM Kali Linux** et utilisé **Binwalk**, un outil qui cherche et extrait ce qui est caché dans un firmware.

> 💡 **Binwalk, c'est quoi ?** Un firmware est un gros bloc de données où plusieurs éléments sont collés les uns aux autres. Binwalk repère les « signatures » de chaque élément, et permet de les **découper et extraire**. Sur **Kali Linux**, il est installé par défaut. Sur une autre distribution, il faut l'installer.

Dans l'extraction, on trouve trois grandes parties :

| Partie                | Rôle                                                                                   |
| --------------------- | -------------------------------------------------------------------------------------- |
| **Bootloader** (boot) | Le premier programme lancé à l'allumage. Il initialise le matériel et lance le noyau.  |
| **Kernel** (noyau)    | Le cœur du système Linux.                                                              |
| **Filesystem** (FS)   | Les fichiers du système : programmes, configuration, pages web de l'interface d'admin… |

Sur Kali, le fichier est rangé dans un dossier `hardwarehacking`, puis passé à Binwalk **sans** extraction, juste pour voir ce qu'il y a dedans.

![binwalk firmware.bin : U-Boot, LZMA et SquashFS](images/software/9%20binwalk%20seule.png)

`binwalk firmware.bin`*. Trois blocs : **U-Boot 1.1.3** (2018), un **noyau LZMA**, puis un **SquashFS** (xz, little endian).*

| Offset     | Contenu                      |
| ---------- | ---------------------------- |
| `0xD0C0`   | Bootloader U-Boot 1.1.3      |
| `0x10200`  | Noyau compressé en LZMA      |
| `0x100000` | Système de fichiers SquashFS |

`binwalk -e` repère bien le SquashFS, mais **n'arrive pas à l'extraire** : il manque les outils, et des liens symboliques sortent du dossier d'extraction.

![binwalk -e échoue sur le SquashFS](images/software/10%20commande%20binwalk%20-e.png)

`binwalk -e` *: « one or more files failed to extract ».*

![Installation de squashfs-tools et xz-utils](images/software/11%20extraction%20non%20effectu%C3%A9%20du%20coup%20on%20utilise%20squashfs-tools%20et%20xy-utils.png)

*J'installe ce qui manque :* `squashfs-tools` *et* `xz-utils`*.*

Binwalk avait donné l'offset du SquashFS : **1 048 576** (`0x100000`). Je découpe le firmware à partir de là avec `dd`, puis j'extrais l'image avec `unsquashfs`.

![dd découpe le SquashFS à partir de l'offset 1048576](images/software/12%20separation%20avec%20dd.png)

`dd` *copie les 3,1 Mo de SquashFS dans* `squashfs.img`*.*

![unsquashfs extrait 419 fichiers](images/software/13%20extraction%20avec%20unsquashfs%20.png)

`unsquashfs` *: 419 fichiers, 43 dossiers. Le système de fichiers est enfin lisible.*

---

<a id="etape-7"></a>

## 🕵️ Étape 7 — Exploration du système de fichiers et constats

Une fois le **filesystem** extrait, j'ai pu parcourir l'arborescence du routeur comme sur n'importe quel Linux.

![Liste de la racine du SquashFS](images/software/14%20exploration%20de%20la%20racine.png)

*Racine : `bin`, `etc`, `lib`, `usr`, `var`, et le dossier `web`.*

### Constat n°1 — hash MD5-crypt du compte `admin`

Dans `etc/passwd.bak`, le compte **admin** n'est pas stocké en clair. C'est un hash **MD5-crypt** (`$1$…`), avec un sel : ce n'est pas un MD5 simple. L'uid est **0** : sur ce routeur, `admin` est **root**.

**John** (`john --format=crypt`) casse ce hash. Sur la capture, il affiche `1234 (admin)` : le mot de passe qui sort est **1234**, et `admin` est le nom du compte.

![passwd.bak avec le hash MD5-crypt du compte admin](images/software/14%2C5%201%20ere%20faille%20trouv%C3%A9%20mot%20de%20passe%20en%20md5%20de%20admin.png)

`etc/passwd.bak` *: le compte* `admin` *et son hash MD5-crypt (*`$1$…`*).*

![John casse le hash : le mot de passe est 1234](images/software/15%20utilisatation%20de%20john%20pour%20cracker%20pour%20le%20fun.png)

*John affiche* `1234 (admin)`*. Le mot de passe est **1234**.*

Dans `bin`, presque toutes les commandes (`ls`, `sh`, `ping`…) sont en fait le même programme : **BusyBox**, un Linux miniature très courant sur les routeurs.

![Dossier bin, commandes fournies par BusyBox](images/software/16%20on%20regarde%20dans%20le%20dossier%20bin.png)

*Contenu de* `bin`*. Ghidra est lancé à côté pour ouvrir* `busybox`*.*

> 💡 **Ghidra, c'est quoi ?** Un outil qui ouvre un programme compilé et le réaffiche en assembleur (et parfois en pseudo-code), pour comprendre ce qu'il fait sans avoir les sources.

![BusyBox importé dans Ghidra, binaire MIPS](images/software/17%20reverse%20de%20busybox%20avec%20ghidra.png)

*BusyBox importé. C'est un binaire **MIPS** 32 bits, little endian — le type de processeur de ce routeur.*

![Désassemblage de busybox dans Ghidra](images/software/18%20a%20l'int%C3%A9rieur%20de%20busy%20box%20avec%20ghidra.png)

*À l'intérieur : le listing assembleur, à partir du point d'entrée.*

![Contenu du dossier web de l'interface](images/software/19%20d%C3%A9couverte%20du%20dossier%20web%20a%20la%20racine.png)

*Le dossier* `web` *: les pages de l'interface (*`index.htm`*,* `frame`*,* `main`*,* `css`*,* `js`*…).*

- 🔓 Constat n°2 : *en cours*
- 🔓 Constat n°3 : *en cours*

---

<a id="glossaire"></a>

## 📖 Glossaire

| Terme          | Définition                                                              |
| -------------- | ----------------------------------------------------------------------- |
| **Firmware**   | Le logiciel interne d'un appareil, stocké dans sa mémoire flash         |
| **PCB**        | Carte électronique sur laquelle sont soudés les composants              |
| **SoC**        | *System on Chip* : processeur + éléments essentiels dans une seule puce |
| **Flash SPI**  | Mémoire non volatile qui stocke le firmware, accessible via le bus SPI  |
| **SOIC8**      | Format de boîtier de puce à 8 pattes                                    |
| **UART**       | Protocole série simple, souvent utilisé pour déboguer                   |
| **TTL**        | Niveau de tension logique (ici 3,3 V) utilisé par l'UART                |
| **Baudrate**   | Vitesse de transmission d'une liaison série                             |
| **Dump**       | Copie complète du contenu d'une mémoire dans un fichier                 |
| **Bootloader** | Premier programme lancé au démarrage                                    |
| **Filesystem** | Organisation des fichiers stockés dans la mémoire                       |
| **Shell**      | Interface en ligne de commande                                          |

---

---

<a id="english"></a>

# 🇬🇧 English version

# 📡 Router Hardware Hacking — TP-Link TL-WR841N V14.1

🇫🇷 [Version française ↑](#francais) · 🇬🇧 English (this document)

My first hardware hacking project: I bought a TP-Link TL-WR841N V14.1 on eBay. At the start, I knew neither the router, nor the brand, nor really what was inside — I had just looked up how to buy an affordable one. The goal: **learn as much as possible** — PCB reading, UART, flash memory dumping, firmware extraction and analysis. Everything is documented step by step (with photos) as a mini-tutorial, so that someone who has never done any of this can follow along and reproduce it.

> 🚧 **This repo is still in progress.** There is no conclusion. Findings 2 and 3 are not written yet. The BusyBox reverse engineering in Ghidra continues in the **next commit**.

![Skill](https://img.shields.io/badge/Skill-Hardware%20Hacking-orange)
![Skill](https://img.shields.io/badge/Skill-Soldering-blue)
![Skill](https://img.shields.io/badge/Skill-UART-blue)
![Skill](https://img.shields.io/badge/Skill-Firmware%20Analysis-green)
![Tool](https://img.shields.io/badge/Tool-Binwalk-lightgrey)
![Tool](https://img.shields.io/badge/Tool-CH341A-lightgrey)
![Tool](https://img.shields.io/badge/Tool-PulseView-lightgrey)
![Target](https://img.shields.io/badge/Target-TL--WR841N%20V14.1-red)
![Status](https://img.shields.io/badge/Status-In%20progress-yellow)

> ⚠️ **Disclaimer**: this entire project is done on **my own hardware**, in a **personal lab and learning context**. Never try these techniques on a device you don't own.

---

## 📑 Table of contents

1. [Hardware used](#hardware)
2. [What is a router?](#what-is-a-router)
3. [What is TP-Link?](#what-is-tp-link)
4. [Step 1 — First contact: the web interface](#step-1)
5. [Step 2 — Opening the router and inspecting the PCB](#step-2)
6. [Step 3 — Finding the UART with a logic analyzer](#step-3)
7. [Step 4 — Shell with a UART–TTL adapter](#step-4)
8. [Step 5 — Dumping the SPI flash](#step-5)
9. [Step 6 — Analyzing the firmware with Binwalk](#step-6)
10. [Step 7 — Exploring the filesystem and the findings](#step-7)
11. [Glossary](#glossary)

---

<a id="hardware"></a>

## 🧰 Hardware used

This is **all the hardware I bought** for this project, with the link and a short explanation. The prices are **what I paid**. They can change: the same item does not cost the same depending on when and where you buy it. Some items may also be out of stock: in that case, buy something similar.

> 💡 The router (the target) was bought for **€12 on eBay, excluding shipping**. That price is not included in the hardware total below.

### The target

| Hardware                                                    | What it's for                                                           | Price                    | Link      |
| ----------------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------ | --------- |
| **TP-LINK TL-WR841N — 300 Mbps Wireless N, new and sealed** | The router I open, dump and analyze. This is the target of the project. | €12 (excluding shipping) | *No link* |

### Opening the case

| Hardware                               | What it's for                                                                                     | Price  | Link                                          |
| -------------------------------------- | ------------------------------------------------------------------------------------------------- | ------ | --------------------------------------------- |
| **120-in-1 precision screwdriver kit** | Open the router and remove tiny electronics screws (Torx, Phillips, etc.) without stripping them. | €10.99 | [Amazon](https://www.amazon.fr/dp/B09WKQXKVV) |
| **Precision flush cutters**            | Cleanly cut wires, component leads or leftover desoldering braid.                                 | €6.99  | [Amazon](https://www.amazon.fr/dp/B09SL2TCH7) |

### Electrical measurement

| Hardware                                  | What it's for                                                                                                             | Price  | Link                                          |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------ | --------------------------------------------- |
| **KAIWEETS digital multimeter**           | Measure voltage (e.g. 3.3 V on UART), continuity on a trace, resistance… The basic tool so you don't wire things blindly. | €15.99 | [Amazon](https://www.amazon.fr/dp/B08CX9W7G3) |
| **Silicone probe / lead kit 1000 V 20 A** | Softer, safer leads for the multimeter (better than the stock ones).                                                      | €6.29  | [Amazon](https://www.amazon.fr/dp/B0D7ZT2PH2) |

### Signals and serial console

| Hardware                                   | What it's for                                                                                                                  | Price  | Link                                          |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ | ------ | --------------------------------------------- |
| **USB logic analyzer, 8 channels, 24 MHz** | Record 0/1 signals on the pins (UART, SPI…). With PulseView you can see TX, find the baud rate and check that data is flowing. | €11.80 | [Amazon](https://www.amazon.fr/dp/B0D8Q886W9) |
| **USB → TTL FTDI adapter (DTECH, 3.3 V)**  | Connect the PC to the router's UART port to try to get a shell (serial console). FT232RNL chip, 3.3 V levels.                  | €11.96 | [Amazon](https://www.amazon.fr/dp/B0DXZ5Q13S) |

### Dumping the flash memory

| Hardware                               | What it's for                                                                                                                          | Price  | Link                                          |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------ | --------------------------------------------- |
| **CH341A programmer kit + SOIC8 clip** | Read (dump) the SPI flash chip that holds the firmware. The clip grips the chip; the 8-pin socket is used once the chip is desoldered. | €11.49 | [Amazon](https://www.amazon.fr/dp/B0H296SJB6) |

### Soldering and desoldering

| Hardware                              | What it's for                                                                            | Price  | Link                                                               |
| ------------------------------------- | ---------------------------------------------------------------------------------------- | ------ | ------------------------------------------------------------------ |
| **80 W soldering iron (180–520 °C)**  | Solder / resolder components.                                                            | €23.98 | [Amazon](https://www.amazon.fr/dp/B09B3GRVTM)                      |
| **Podofo 858D 700 W hot-air station** | Desolder a multi-pin chip (like the SOIC8 flash) with hot air, without ripping pads off. | €23.35 | [AliExpress](https://fr.aliexpress.com/item/1005013019903116.html) |
| **Desoldering braid (4 widths)**      | “Suck up” molten solder to clean the PCB pads after desoldering.                         | €8.99  | [Amazon](https://www.amazon.fr/dp/B0GWF24LT8)                      |
| **No-clean liquid flux #923 (30 mL)** | Helps solder wet the pins properly. Without flux, joints are often ugly or cold.         | €7.89  | [Amazon](https://www.amazon.fr/dp/B0G2GTRDBG)                      |
| **99.9% isopropyl alcohol (750 mL)**  | Clean flux residue, fingerprints and dust off the PCB when you're done.                  | €7.70  | [Amazon](https://www.amazon.fr/dp/B0CK2FX99D)                      |

### Workbench and safety

| Hardware                                     | What it's for                                                                           | Price  | Link                                          |
| -------------------------------------------- | --------------------------------------------------------------------------------------- | ------ | --------------------------------------------- |
| **Silicone soldering mat 30×20 cm (500 °C)** | Protects the desk from heat; the small pockets keep screws and chips from disappearing. | €9.99  | [Amazon](https://www.amazon.fr/dp/B0CRB58Q93) |
| **Safety glasses**                           | Protect your eyes from solder splashes and hot air.                                     | €10.19 | [Amazon](https://www.amazon.fr/dp/B0FC6962G9) |
| **FFP2 masks (pack of 50)**                  | Reduce inhalation of solder / flux fumes. Wear one as soon as you start heating.        | €11.89 | [Amazon](https://www.amazon.fr/dp/B0FS1Y3Q9Z) |

### Later in this project

| Hardware                           | What it's for                                                                              | Price  | Link                                          |
| ---------------------------------- | ------------------------------------------------------------------------------------------ | ------ | --------------------------------------------- |
| **Raspberry Pi Pico 2 W (RP2350)** | Microcontroller for **glitching** or **side-channel** work. **Not required for the dump**. | €13.67 | [Amazon](https://www.amazon.fr/dp/B0DQ1555PJ) |

> 💡 **What is glitching?** You briefly disturb a chip (a voltage dip, a “hit” on its clock) at the exact moment it runs a check. If the timing is right, it can skip that check. **Side-channel** is the other way around: you don't break anything, you **watch what leaks** around it (power use, response time, electromagnetic noise) while it works, and infer information it doesn't show directly. The Pico is a cheap, precise trigger for that kind of experiment. Dumping the flash is done with the CH341A.

### What I paid

|                                                     |             |
| --------------------------------------------------- | ----------- |
| **Total of listed hardware (excluding the router)** | **€193.16** |
| **Router (eBay, excluding shipping)**               | **€12**     |

> These are the prices at the time I bought them. The same gear can cost more or less somewhere else, or later. If an item is no longer available, a similar one is fine. This is still a **very affordable first hardware lab**: you don't need a €2000 oscilloscope to get started.

**Software:**

| Software                                                                           | Usage                                                                                             |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| [PulseView / sigrok](https://github.com/sigrokproject/pulseview)                   | Open-source software to visualize logic analyzer signals                                          |
| [NeoProgrammer](https://github.com/GioLangLe/CH341B-NeoProgramer)                  | Flash reading software used with the CH341A                                                       |
| [PuTTY](https://www.putty.org/)                                                    | Serial terminal used to try to get a shell on the UART                                            |
| [Kali Linux virtual machine](https://www.kali.org/get-kali/#kali-virtual-machines) | Analysis environment, an OS for pentesters, tools already installed                         |
| [Binwalk](https://github.com/ReFirmLabs/binwalk)                                   | Firmware analysis and extraction. Installed by default on Kali, otherwise it has to be installed. |
| [Ghidra](https://github.com/NationalSecurityAgency/ghidra)                         | Open a compiled program (here BusyBox) and read it as assembly and as C                           |

---

<a id="what-is-a-router"></a>

## 🌐 What is a router?

A **router** moves data from one network to another. It looks at the destination address of each packet and picks the path.

The Wi-Fi box at home, like this TL-WR841N, is a router **plus** other jobs: Wi-Fi, the 4 Ethernet ports, handing out IP addresses (DHCP) and a basic firewall. The fiber or phone line lands on the ISP's box or modem. This router plugs in behind it, on the WAN port.

Inside, it is also a **tiny computer**. It contains:

- a **processor / SoC** (the brain),
- **RAM** (working memory),
- a **flash memory** (like a hard drive: it stores the system),
- Wi-Fi radio chips, Ethernet ports…
- an **operating system**, very often an **embedded Linux**.

That's what makes it a great target for learning hardware hacking: you can open it, talk directly to its system and retrieve its internal software (the **firmware**).

<a id="what-is-tp-link"></a>

## 🏢 What is TP-Link?

**TP-Link** is a Chinese manufacturer of networking equipment (routers, switches, Wi-Fi extenders, cameras, smart plugs…), founded in 1996. It is one of the biggest consumer router makers in the world, known for its very affordable prices.

The **TL-WR841N** is one of their best-selling entry-level models: a 300 Mbps Wi-Fi N router with 4 Ethernet ports. **V14.1** is the **hardware revision**: the same model name can hide totally different internal components from one version to another. That's why it's important to always note the exact version of your device.

The router as I received it, still closed, on the desk.

![TP-Link TL-WR841N as received, case closed](images/hardware/0%20photo%20du%20routeur.jpeg)

*The TL-WR841N on arrival, before opening it.*

---

<a id="step-1"></a>

## 🔍 Step 1 — First contact: the web interface

Before opening anything, we start by **gathering as much information as possible** about the device as it is sold.

1. Plug in and power on the router.
2. Connect to its admin interface at **[http://tplinkwifi.net](http://tplinkwifi.net)**.
3. Default credentials: **`admin` / `admin`**.

> 💡 **Why does it matter?** Default credentials this simple are a classic security weakness: many users never change them.

Login page, before opening the case. The header already shows **300M Wireless N Router** and model **TL-WR841N**. The hardware version, read once the router was powered on, is **V14.1**.

![Router login page on tplinkwifi.net](images/hardware/0%2C1%20page%20de%20login%20du%20routeur.png)

*Login page (*`http://tplinkwifi.net`*). Default credentials:* `admin` */* `admin`*.*

---

<a id="step-2"></a>

## 🔧 Step 2 — Opening the router and inspecting the PCB

The **PCB** (*Printed Circuit Board*) is the electronic board on which all components are soldered. After opening the case, I inspected the board closely to **identify every chip**:

| Component        | Reference                              | Role                                                                                                     |
| ---------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| SoC (CPU)        | MediaTek **MT7628NN**                  | The router's brain. The 2.4 GHz Wi-Fi is inside this chip: there is no separate radio chip on the board. |
| RAM              | Zentel **A3S56D40GTP-50L**             | Working memory                                                                                           |
| SPI flash memory | GigaDevice **GD25Q32** (32 Mbit, 4 MB) | Stores the firmware                                                                                      |
| UART             | 4 pads (VCC, GND, RX, TX)              | Serial console, next to the flash                                                                        |

> 💡 **Tip:** the references printed on the chips (for example `25Q32…`) let you find their **datasheet** online, which describes the role of each pin.

I also noticed **test points and pads** on the board, including pins that look like a **UART port**.

Top view, case open. I circled four things in red for what comes next: the **RAM**, the **CPU**, the **SPI flash** and the **UART** pads. The word on the photo is French: **PROCESSEUR** means CPU.

![Top view of the PCB with RAM, CPU, SPI flash and UART circled in red](images/hardware/0%2C5%20photo%20routeur%20vue%20du%20dessus.jpeg)

*Top view. In red: RAM (Zentel), CPU (MediaTek MT7628NN, labeled PROCESSEUR), SPI flash and UART pads.*

I have no photo **during** desoldering. The close-ups below are the flash, still soldered on the board (designator **U4**).

![Close-up of the flash chip still soldered, designator U4](images/hardware/1%20premiere%20photo%20de%20la%20puce.jpeg)

*First photo of the flash, still on the PCB (designator U4, 8-pin package).*

![Another close-up of the flash, UART pads on the left](images/hardware/3%20secondes%20photo%20de%20la%20puce.jpeg)

*Another angle. On the left, the serial connector is already visible (VCC, GND, RX, TX).*

---

<a id="step-3"></a>

## 📈 Step 3 — Finding the UART with a logic analyzer

### What is UART?

**UART** (*Universal Asynchronous Receiver/Transmitter*) is a very simple serial communication protocol. On many electronic devices, manufacturers leave a UART port for development use, to **print boot messages** and sometimes give access to a **console (shell)**.

A UART port usually has 4 pins:

| Pin     | Role                                            |
| ------- | ----------------------------------------------- |
| **VCC** | Power supply (3.3 V) — *usually NOT connected!* |
| **GND** | Ground (0 V reference)                          |
| **TX**  | The router **transmits** (sends data)           |
| **RX**  | The router **receives**                         |

### What is a logic analyzer?

It's a tool that **records the state (0 or 1) of electrical signals** over time. With the open-source software **[PulseView](https://github.com/sigrokproject/pulseview)** (the [sigrok](https://sigrok.org/wiki/PulseView) project), you can see those signals, decode the protocol and thereby **find which pin is TX**, confirm the transmission speed (**baud rate**) and check that data is actually flowing.

On the top view, the UART pads are circled in red, right next to the flash. The close-up below shows the four silkscreen names, then the physical UART test.

![Flash U4 and UART pads VCC GND RX TX](images/hardware/2%20seconde%20photo%20de%20la%20puce%20on%20voit%20aussi%20les%20ports%20uart.jpeg)

*The flash (U4) and, on the left, the UART pads: **VCC**, **GND**, **RX**, **TX**.*

![Physical UART test, clips on the pads](images/hardware/0%2C7%20test%20uart.jpeg)

*Physical UART test. The clips are on the pads.*

In PulseView the analyzer shows up under the name **Saleae Logic**. The useful signal is on **D0**. The other channels stay flat. I record at **1 MHz**.

![PulseView: signal on D0, no decoder yet](images/software/0%2C1%20on%20voit%20le%20signal%20sur%20d0.png)

*First capture. D0 moves, the other channels do not. No decoder yet.*

![PulseView zoom on the D0 pulses](images/software/0%2C2%20on%20zoome%20et%20on%20voit%20les%20cr%C3%A9neaux.png)

*Zoomed in. The pulses are a serial line.*

I add the UART decoder on D0: **115200** baud, 8 bits, no parity, 1 stop bit. At first PulseView reports an error.

![UART decoder at 115200 on D0, PulseView reports an error](images/software/0%2C3%20on%20ajoute%20le%20d%C3%A9codeur%20uart%20mais%20erreur.png)

*UART decoder on D0, 115200. PulseView reports an error.*

Right after that, the same channel decodes as text.

![Decoded UART text: ntp connect failed](images/software/0%2C4%20le%20d%C3%A9codeur%20affiche%20ntp%20connect%20failed.png)

*The text shows up:* `ntp connect failed`*.*

On capture 0,5 I raise the sample count: I go up to **50 million**, to see further into the boot. The rate stays **1 MHz**.

![Long boot capture with the UART decoder](images/software/0%2C5%20capture%20longue%20du%20d%C3%A9marrage.png)

*50 million samples, so more of the boot is visible. D0 talks in bursts, with the decoder underneath.*

![Decoded UART: U-Boot 1.1.3 from 19 March 2018](images/software/0%2C6%20on%20lit%20u-boot%201.1.3%20mars%202018.png)

`U-Boot 1.1.3 (Mar 19 2018)`*. The same bootloader Binwalk will find in the firmware.*

![UART: the flash is detected at boot](images/software/0%2C7%20la%20flash%20est%20reconnue%20au%20boot.png)

*The flash is detected at boot:* `device id 0x16 find flash`*.*

![UART: Ralink UBoot Version 4.3.0.0](images/software/0%2C8%20on%20lit%20ralink%20uboot%204.3.0.0.png)

`Ralink UBoot Version: 4.3.0.0`*. Ralink is the old name that still shows up in this MediaTek SoC's boot log.*

![UART: the boot log mentions the MT7628](images/software/0%2C9%20le%20boot%20parle%20du%20mt7628.png)

*The boot log mentions the **MT7628**.*

![Rest of the UART boot log](images/software/0%2C10%20suite%20du%20journal%20de%20boot.png)

*The rest of the log: components and sizes in bytes.*

![UART: 32 MB of RAM and SPI flash](images/software/0%2C11%2032%20mo%20de%20ram%20et%20flash%20spi.png)

`Total memory 32 Mbytes`*, and the flash is **SPI**.*

![UART: flash date in 2018](images/software/0%2C12%20date%20de%20la%20flash%202018.png)

*Flash date: **2018**.*

![UART: Ethernet and Wi-Fi settings](images/software/0%2C13%20r%C3%A9glages%20ethernet%20et%20wifi.png)

*Ethernet and Wi-Fi settings at boot.*

![UART: Uncompressing Kernel Image](images/software/0%2C14%20uncompressing%20kernel%20image.png)

`Uncompressing Kernel Image`*. U-Boot decompresses the kernel.*

![UART: Booting image at bc010000](images/software/0%2C15%20booting%20image%20at%20bc010000.png)

`Booting image at bc010000`*.*

![UART: do_bootm starts the kernel](images/software/0%2C16%20le%20noyau%20d%C3%A9marre%20do_bootm.png)

`do_bootm`*. The kernel is started.*

---

<a id="step-4"></a>

## 🖥️ Step 4 — Shell with a UART–TTL adapter

After PulseView I unplugged the logic analyzer. I switch to the **USB–TTL** adapter and the **PuTTY** terminal: it turns the router's signals into a virtual serial port.

On the photo, the clips are wired like this: **GND to GND**, **RX to TX** and **TX to RX**. The two data wires are crossed: what the router sends (its TX), the adapter has to receive (its RX), and the other way around. **VCC** stays unplugged. The adapter must not power the board.

![UART test: clips with GND to GND, RX to TX and TX to RX](images/hardware/0%2C7%20test%20uart.jpeg)

*UART test. GND to GND, RX to TX and TX to RX.*

In PuTTY the session is **Serial**, on **COM4**, at **115200**.

![PuTTY set to Serial, COM4, 115200](images/software/0%2C16%2C5%20configuration%20putty%20com4%20115200.png)

*PuTTY settings: Serial, COM4, 115200.*

Then I mashed **Enter** until the prompt showed up. The window is `COM4 - PuTTY`. The prompt is `#`: I am root.

![PuTTY: end of boot and a root prompt after mashing Enter](images/software/0%2C17%20shell%20ouvert%20en%20spammant%20entr%C3%A9e.png)

*End of the boot, then the prompt. This is where I mashed Enter.*

![PuTTY: ls at the router root](images/software/0%2C18%20ls%20de%20la%20racine.png)

`ls` *at the root:* `bin`*,* `dev`*,* `etc`*,* `lib`*,* `linuxrc`*,* `mnt`*,* `proc`*,* `sbin`*,* `sys`*,* `usr`*,* `var`*,* `web`*.*

![PuTTY: cat passwd in etc](images/software/0%2C19%20cat%20passwd%20dans%20etc.png)

*In* `etc`*,* `cat passwd`*. The **admin** account has an MD5-crypt hash and uid **0**.*

![PuTTY: listing of /proc](images/software/0%2C20%20listing%20de%20proc.png)

*Contents of* `/proc`*.*

![PuTTY: mtd partitions boot kernel rootfs config radio](images/software/0%2C21%20partitions%20boot%20kernel%20rootfs%20config%20radio.png)

`cat /proc/mtd`*. Five partitions:* `boot`*,* `kernel`*,* `rootfs`*,* `config`*,* `radio`*.*

`dd` is not on the router. Copying to `/tmp` fails too: that directory does not exist. So I copy into `/var`.

![PuTTY: dd not found](images/software/0%2C22%20dd%20not%20found.png)

`dd if=/dev/mtd4 of=/tmp/radio.bin` *→* `dd: not found`*.*

![PuTTY: the tmp directory does not exist](images/software/0%2C23%20tmp%20inexistant.png)

`cat /dev/mtd4 > /tmp/radio.bin` *→* `/tmp` *does not exist.*

![PuTTY: radio.bin, uboot.bin and config.bin in /var](images/software/0%2C24%20copies%20radio%20uboot%20config%20dans%20var.png)

*The copies go through* `/var`*:* `radio.bin`*,* `uboot.bin` *and* `config.bin`*, 65536 bytes each.*

I then try to send them to the PC with **TFTP**. `tftp -h` and `tftp --help` are rejected. Sending `radio.bin` does not leave either: timeout to `192.168.0.100`, then `network unreachable` to `192.168.56.1`. That second address is VirtualBox's default host-only IP. The router cannot reach the PC: this is a network setup problem, not a limit of TFTP.

![PuTTY: tftp rejects -h and --help](images/software/0%2C25%20tftp%20aide%20refus%C3%A9e.png)

`tftp -h` *and* `tftp --help` *are rejected.*

![PuTTY: tftp timeout then network unreachable](images/software/0%2C26%20tftp%20timeout%20et%20r%C3%A9seau%20injoignable.png)

*Sending* `radio.bin`*: timeout to* `192.168.0.100`*, then* `network unreachable` *to* `192.168.56.1`*.*

**Result:** the root shell opens. The files stay on the router, because the network path to the PC does not work.

---

<a id="step-5"></a>

## 💾 Step 5 — Dumping the SPI flash

The shell was open. The files never left the router: TFTP hit a timeout, then an unreachable network. So I **read the flash memory directly**, the chip that holds the firmware. This is called a **dump**.

### The tools

- **SPI flash memory**: the chip (often in a **SOIC8**, 8-pin package) that stores the router's system.
- **CH341A**: a cheap USB programmer to read and write these chips.
- **SOIC8 clip**: a clip that grips the chip so you can read it **without desoldering**.
- **NeoProgrammer**: the software that drives the CH341A.

### Attempt 1 — In-circuit reading with the clip

I clipped the SOIC8 clamp onto the flash chip, still soldered in the router, connected it to the CH341A, and tried to read with NeoProgrammer.

**Problem:** it didn't work. As soon as I connected it, the **CH341A's LED dimmed**, as if the programmer was also powering the rest of the board.

> 💡 **Explanation:** when reading a chip still soldered on its circuit, current from the programmer can power the other components (SoC, etc.), which then start "talking" on the same bus. This causes **interference** and the read fails.

### Attempt 2 — Desoldering the chip ✅

I went for a cleaner method: **desolder the flash chip** to isolate it completely from the rest of the board, then **resolder it onto an 8-pin socket** (included in the CH341A kit). Plugging that socket **straight** into the CH341A did not work. With the **clip** on the chip, it worked: I managed to **dump the firmware**.

### Verification

To make sure the dump was reliable, I **read it a second time** and compared both files: no difference → the dump is good ✅.

The chip NeoProgrammer reads is a **GigaDevice GD25Q32**, SPI NOR, 3.3 V, 32 Mbit (4,194,304 bytes).

No photo **during** the hot-air desoldering. The sequence starts just after: the chip is off the board, and the case plastic is shielded with aluminum foil.

![PCB after desoldering, aluminum foil around the plastic](images/hardware/4%20photo%20apres%20dessoudage%20puce%20utilisation%20de%20aluminium%20pour%20proteger%20le%20pastique.jpeg)

*After desoldering. The foil protects the case plastic from the heat.*

![Flash held in tweezers, empty U4 footprint](images/hardware/5%20photo%20apres%20dessoudage%20puce.jpeg)

*The flash, off the board. The **U4** footprint is empty.*

Before soldering it back, I clean the pins and the socket with isopropyl alcohol.

![99.9% isopropyl alcohol and cotton swabs](images/hardware/6%20nettoyage%20du%20support%20de%20la%20puce.jpeg)

*Cleaning: 99.9% isopropyl alcohol and cotton swabs. The liquid flux is there too.*

Then I resolder it onto the 8-pin socket from the CH341A kit.

![Flash soldered onto the SOIC8 adapter](images/hardware/7%20puce%20souder%20sur%20le%20support%20.jpeg)

*Flash soldered onto the adapter. The numbered pads (1 to 8) keep the pinout straight.*

![Another angle of the flash soldered on the adapter](images/hardware/8%20autre%20angle%20de%20la%20puce%20souder.jpeg)

*Another angle. The socket has pins underneath: the plan was to plug it straight into the programmer.*

I first plugged this socket **directly** into the CH341A. Orientation matters: the chip has a dot (pin 1 mark), and on this setup **pin 1 must be at the top right**. It didn't work. The solder joint must have been bad: the contact between the chip and the socket wasn't making it through.

![Socket plugged directly into the CH341A, pin 1 at the top right](images/hardware/9%20je%20montre%20le%20sens%20le%201%20doit%20etre%20en%20haut%20a%20droite.jpeg)

*Socket plugged straight into the CH341A. Mark: pin **1** at the top right. This read failed.*

So I clipped the SOIC8 clip onto the chip, already soldered on the socket, and ran the ribbon cable to the programmer. That worked.

![SOIC8 clip on the flash, wired to the CH341A](images/hardware/10%20branche%20la%20pince%20sur%20la%20puce%20.jpeg)

*The clip on the chip, instead of the direct plug. The CH341A is on the left, connected over USB.*

![Clip ribbon cable on the CH341A socket](images/hardware/11%20autre%20angle%20du%20support%20de%20la%20pince%20sur%20le%20ch314a.jpeg)

*Another angle: the clip's ribbon cable lands on the CH341A socket.*

![Second angle of the CH341A with the ribbon cable connected](images/hardware/12%20autre%20angle%20du%20support%20de%20la%20pince%20sur%20le%20ch314a.jpeg)

*Same wiring, other angle. This is the setup that finally dumped the firmware.*

### Reading it with NeoProgrammer

On launch, the CH341A is detected ("CH341 Black") but the chip does not answer yet: the right model has to be selected first.

![NeoProgrammer at launch, chip not detected](images/software/1%20on%20lance%20neoprogrammer.png)

*NeoProgrammer 2.2.0.10 at startup. The programmer is there, the chip is not responding.*

I click **Search** and look up the reference printed on the package.

![Searching for the GD25Q32 chip in NeoProgrammer](images/software/2%20on%20clique%20sur%20search.png)

*Search: three 32 Mbit GigaDevice chips.*

![GD25Q32 and GD25Q32B, both 3.3 V and 32 Mbit](images/software/3%20les%20deux%20qui%20nous%20int%C3%A9resse.png)

*The two that match the marking: **GD25Q32** and **GD25Q32B**, 3.3 V, 32 Mbit.*

I select **GD25Q32 [3.3V]**. The SPI ID read back is `C84016`, and the announced size is 4 MB.

![GD25Q32 3.3 V selected, SPI ID C84016](images/software/4%20selection%20de%20la%20bonne%20puce.png)

*GD25Q32 selected. SPI ID* `C84016`*, 4,194,304 bytes.*

Then **Read**: NeoProgrammer reads the whole flash.

![Just before reading the flash](images/software/5%20on%20clique%20sur%20read%20pour%20extraire%20le%20fireware.png)

*Just before the read.*

![Read succeeded, 4 MB hex dump](images/software/6%20succes%20de%20l'extraction%20du%20fireware.png)

*Read succeeded. The buffer is 4,194,304 bytes, CRC32* `0x9559AB35`*.*

To check it, I read it again. The saved file is named `firmware.bin`.

![Verifying the firmware.bin dump](images/software/7%20v%C3%A9rification%20du%20fireware%20extrait%20pour%20etre%20sur.png)

*Second read, to compare.*

![Verification succeeded, firmware.bin ready for Kali](images/software/8%20v%C3%A9rification%20r%C3%A9ussit%20on%20glisse%20le%20fireware%20sur%20kali%20linux.png)

*Verification succeeded (*`Success`*). I then transfer* `firmware.bin` *to the Kali VM.*

---

<a id="step-6"></a>

## 🧩 Step 6 — Analyzing the firmware with Binwalk

The resulting file is a **raw image** of the flash memory. I transferred it to a **Kali Linux VM** and used **Binwalk**, a tool that searches for and extracts what's hidden inside a firmware.

> 💡 **What is Binwalk?** A firmware is a big blob of data where several elements are glued together. Binwalk spots the "signatures" of each element and lets you **cut out and extract** them. On **Kali Linux** it is installed by default. On another distribution, it has to be installed.

The extraction reveals three main parts:

| Part                  | Role                                                                                        |
| --------------------- | ------------------------------------------------------------------------------------------- |
| **Bootloader** (boot) | The first program that runs at power-on. It initializes the hardware and starts the kernel. |
| **Kernel**            | The core of the Linux system.                                                               |
| **Filesystem** (FS)   | The system's files: programs, configuration, the admin interface web pages…                 |

On Kali, the file goes into a `hardwarehacking` folder, then through Binwalk **without** extraction, just to see what's inside.

![binwalk firmware.bin: U-Boot, LZMA and SquashFS](images/software/9%20binwalk%20seule.png)

`binwalk firmware.bin`*. Three blocks: **U-Boot 1.1.3** (2018), an **LZMA kernel**, then a **SquashFS** (xz, little endian).*

| Offset     | Content                 |
| ---------- | ----------------------- |
| `0xD0C0`   | U-Boot 1.1.3 bootloader |
| `0x10200`  | LZMA-compressed kernel  |
| `0x100000` | SquashFS filesystem     |

`binwalk -e` does find the SquashFS, but **fails to extract it**: the tools are missing, and some symlinks point outside the extraction folder.

![binwalk -e fails on the SquashFS](images/software/10%20commande%20binwalk%20-e.png)

`binwalk -e`*: "one or more files failed to extract".*

![Installing squashfs-tools and xz-utils](images/software/11%20extraction%20non%20effectu%C3%A9%20du%20coup%20on%20utilise%20squashfs-tools%20et%20xy-utils.png)

*I install what's missing:* `squashfs-tools` *and* `xz-utils`*.*

Binwalk had given the SquashFS offset: **1,048,576** (`0x100000`). I cut the firmware from there with `dd`, then extract the image with `unsquashfs`.

![dd cuts the SquashFS from offset 1048576](images/software/12%20separation%20avec%20dd.png)

`dd` *copies the 3.1 MB SquashFS into* `squashfs.img`*.*

![unsquashfs extracts 419 files](images/software/13%20extraction%20avec%20unsquashfs%20.png)

`unsquashfs`*: 419 files, 43 directories. The filesystem is finally readable.*

---

<a id="step-7"></a>

## 🕵️ Step 7 — Exploring the filesystem and the findings

Once the **filesystem** was extracted, I could browse the router's directory tree like any Linux system.

![Listing of the SquashFS root](images/software/14%20exploration%20de%20la%20racine.png)

*Root: `bin`, `etc`, `lib`, `usr`, `var`, and the `web` folder.*

### Finding #1 — the `admin` account's MD5-crypt hash

In `etc/passwd.bak`, the **admin** account is not stored in clear. It is an **MD5-crypt** hash (`$1$…`), with a salt: this is not raw MD5. The uid is **0**: on this router, `admin` is **root**.

**John** (`john --format=crypt`) cracks this hash. On the screenshot it prints `1234 (admin)`: the password that comes out is **1234**, and `admin` is the account name.

![passwd.bak with the MD5-crypt hash of the admin account](images/software/14%2C5%201%20ere%20faille%20trouv%C3%A9%20mot%20de%20passe%20en%20md5%20de%20admin.png)

`etc/passwd.bak`*: the* `admin` *account and its MD5-crypt hash (*`$1$…`*).*

![John cracks the hash: the password is 1234](images/software/15%20utilisatation%20de%20john%20pour%20cracker%20pour%20le%20fun.png)

*John prints* `1234 (admin)`*. The password is **1234**.*

In `bin`, almost every command (`ls`, `sh`, `ping`…) is actually the same program: **BusyBox**, a tiny Linux very common on routers.

![bin directory, commands provided by BusyBox](images/software/16%20on%20regarde%20dans%20le%20dossier%20bin.png)

*Contents of* `bin`*. Ghidra is started next to it to open* `busybox`*.*

> 💡 **What is Ghidra?** A tool that opens a compiled program and shows it as assembly (and sometimes pseudo-code), so you can see what it does without the source.

![BusyBox imported into Ghidra, MIPS binary](images/software/17%20reverse%20de%20busybox%20avec%20ghidra.png)

*BusyBox imported. It is a 32-bit **MIPS** binary, little endian — the kind of CPU in this router.*

![Disassembly of busybox in Ghidra](images/software/18%20a%20l'int%C3%A9rieur%20de%20busy%20box%20avec%20ghidra.png)

*Inside: the assembly listing, starting at the entry point.*

![Contents of the web interface folder](images/software/19%20d%C3%A9couverte%20du%20dossier%20web%20a%20la%20racine.png)

*The* `web` *folder: the interface pages (*`index.htm`*,* `frame`*,* `main`*,* `css`*,* `js`*…).*

- 🔓 Finding #2: *in progress*
- 🔓 Finding #3: *in progress*

---

<a id="glossary"></a>

## 📖 Glossary

| Term           | Definition                                                             |
| -------------- | ---------------------------------------------------------------------- |
| **Firmware**   | The internal software of a device, stored in its flash memory          |
| **PCB**        | Electronic board on which components are soldered                      |
| **SoC**        | *System on Chip*: processor + essential elements in a single chip      |
| **SPI flash**  | Non-volatile memory storing the firmware, accessed through the SPI bus |
| **SOIC8**      | 8-pin chip package format                                              |
| **UART**       | Simple serial protocol, often used for debugging                       |
| **TTL**        | Logic voltage level (here 3.3 V) used by UART                          |
| **Baud rate**  | Transmission speed of a serial link                                    |
| **Dump**       | A complete copy of a memory's contents into a file                     |
| **Bootloader** | First program run at startup                                           |
| **Filesystem** | The way files are organized in storage                                 |
| **Shell**      | Command-line interface                                                 |


