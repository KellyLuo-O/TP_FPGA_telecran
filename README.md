# TP FPGA — Télécran

TP réalisé dans le cadre du **TP FPGA de 3A à l'ENSEA**.
Ce projet consiste à réaliser un **écran magique numérique (télécran)** sur FPGA. Le dessin est effectué à l'aide des deux encodeurs de la carte et affiché sur un écran via la sortie HDMI.
La partie avancée du TP ajoute un **soft-processeur Nios V** et un **accéléromètre ADXL345** afin de reproduire plus fidèlement le fonctionnement d'un véritable télécran : l'écran peut être effacé en retournant la carte.

## 📌 Objectifs

Le TP est divisé en deux parties principales :

* **TP FPGA** : conception du télécran en VHDL ;
* **TP FPGA avancé** : intégration d'un processeur Nios V, d'une interface I2C et de l'accéléromètre ADXL345.

Les différentes étapes permettent notamment de travailler sur :

* la conception de circuits séquentiels en VHDL ;
* la gestion d'encodeurs en quadrature ;
* la génération d'un affichage HDMI ;
* la gestion d'un framebuffer avec une RAM dual-port ;
* la communication I2C ;
* l'utilisation d'un soft-processeur Nios V ;
* la programmation en C d'un système embarqué sur FPGA.

## 🖥️ Fonctionnement du télécran

Le stylet du télécran est contrôlé par les deux encodeurs de la carte :

* l'encodeur de gauche contrôle la position **horizontale** ;
* l'encodeur de droite contrôle la position **verticale**.

La position du stylet est convertie en coordonnées `(x, y)`.

Le contrôleur HDMI parcourt ensuite l'ensemble des pixels de l'écran. Lorsque les coordonnées du pixel correspondent à celles du stylet, le pixel est affiché en blanc ; sinon, il reste noir.

Une mémoire framebuffer permet ensuite de conserver les pixels déjà dessinés afin que le tracé reste affiché lorsque le stylet se déplace.

## 🔧 Partie 1 — Conception du télécran en VHDL

### 1. Gestion des encodeurs

Les encodeurs renvoient deux signaux `A` et `B` en quadrature.

La direction de rotation est déterminée en observant les transitions des deux signaux. Des bascules D permettent de mémoriser les valeurs précédentes afin de détecter les fronts montants et descendants.

Deux compteurs permettent ensuite de mémoriser les coordonnées du stylet :

```text
Encodeur gauche  → X
Encodeur droit   → Y
```

Les compteurs sont bornés afin que le stylet reste dans les limites de l'écran.

Les simulations permettent notamment de vérifier :

* la remise à zéro par `rst_n` ;
* l'incrémentation et la décrémentation ;
* la saturation aux limites du compteur.

### 2. Contrôleur HDMI

Le contrôleur HDMI fournit les signaux nécessaires à l'affichage ainsi que les coordonnées du pixel actuellement traité :

```text
s_x_counter
s_y_counter
```

Il est cadencé par l'horloge issue de la PLL.

Les composantes de couleur sont codées sur 24 bits :

```text
[23:16] → Rouge
[15:8]  → Vert
[7:0]   → Bleu
```

Le pixel correspondant à la position du stylet est alors affiché en blanc :

```vhdl
if (s_x_counter = s_x_encoder) and
   (s_y_counter = s_y_encoder) then
    o_hdmi_tx_d <= x"FFFFFF";
else
    o_hdmi_tx_d <= x"000000";
end if;
```

### 3. Déplacement du pixel

À cette étape, un seul pixel est affiché à l'écran.

La rotation des encodeurs permet de déplacer ce pixel horizontalement et verticalement.

Cette étape permet de valider l'interaction entre :

* les encodeurs ;
* les compteurs de position ;
* le contrôleur HDMI.

### 4. Mémorisation des pixels

Afin de conserver le dessin, un **framebuffer** est utilisé.

La mémoire implémentée est une **RAM dual-port**, ce qui permet d'utiliser simultanément deux ports indépendants :

* **Port A** : écriture des pixels en fonction de la position du stylet ;
* **Port B** : lecture des pixels pour l'affichage HDMI.

L'adresse mémoire correspondant à un pixel `(x, y)` est calculée par :

```text
adresse = x + y × résolution_horizontale
```

Les pixels parcourus par le stylet sont alors enregistrés dans le framebuffer.

### 5. Effacement de l'écran

L'effacement consiste à écrire `0` dans toutes les adresses du framebuffer.

Une RAM ne pouvant effectuer qu'une écriture par cycle sur le port utilisé, les adresses sont parcourues séquentiellement.

Une logique de contrôle permet donc :

1. de détecter la demande d'effacement ;
2. de parcourir toutes les adresses de la mémoire ;
3. d'écrire `0` à chaque adresse ;
4. de revenir au fonctionnement normal une fois l'écran entièrement effacé.

## 🚀 Partie 2 — FPGA avancé

La seconde partie du TP consiste à intégrer un **soft-processeur Nios V** au projet.

Le système utilise notamment :

* un processeur **Nios V** ;
* une mémoire On-Chip ;
* un **JTAG UART** ;
* des **PIO** ;
* un contrôleur **I2C**.

La configuration matérielle est réalisée avec **Platform Designer**, puis intégrée au projet Quartus. Les consignes du TP recommandent notamment une organisation séparant les fichiers `rtl`, `synt`, `sim`, `sopc` et `soft`.

### 1. Premier programme sur Nios V

Un premier programme C permet de vérifier le fonctionnement du soft-processeur avec un simple :

```c
printf("Hello, world!\n");
```

La sortie est récupérée via le **JTAG UART** avec `juart-terminal`.

### 2. Chenillard avec le Nios V

Le Nios V contrôle les LEDs à travers un périphérique PIO.

La macro utilisée pour écrire sur le PIO est :

```c
IOWR_ALTERA_AVALON_PIO_DATA(PIO_0_BASE, led_value);
```

Un programme en C permet ainsi de faire défiler une LED allumée parmi les 10 LEDs disponibles.

### 3. Utilisation de l'accéléromètre

L'accéléromètre **ADXL345** est connecté au FPGA via le bus **I2C**.

Le Nios V est donc équipé d'un contrôleur I2C permettant :

* d'initialiser l'ADXL345 ;
* de lire son identifiant ;
* de configurer le mode de mesure ;
* de récupérer les accélérations sur les axes `X`, `Y` et `Z`.

Les données sont ensuite exploitées dans le programme C.

### 4. Niveau à bulle

Les valeurs d'accélération mesurées sur l'axe `Y` permettent de déterminer l'inclinaison de la carte.

Les 10 LEDs sont utilisées pour représenter cette inclinaison :

```text
LED gauche  ← carte inclinée à gauche
LED centrale ← carte horizontale
LED droite  ← carte inclinée à droite
```

Cette étape permet de valider la communication entre le Nios V et l'accéléromètre avant de l'intégrer au télécran.

## 🔄 Retour de l'écran magique

La dernière étape consiste à intégrer le système Nios V au projet télécran.

Un PIO supplémentaire est utilisé comme **signal de demande d'effacement**.

Le Nios V lit continuellement les valeurs de l'accéléromètre et détecte lorsque la carte est retournée.

La condition utilisée est :

```c
if (acc_z < -25)
    IOWR_ALTERA_AVALON_PIO_DATA(PIO_0_BASE, 0);
else
    IOWR_ALTERA_AVALON_PIO_DATA(PIO_0_BASE, 1);
```

Le signal du PIO est ensuite connecté au VHDL du télécran.

Ainsi :

```text
                 ┌─────────────────┐
                 │   Accéléromètre │
                 │     ADXL345     │
                 └────────┬────────┘
                          │ I2C
                          ▼
                 ┌─────────────────┐
                 │     Nios V      │
                 │                 │
                 │ Lecture de acc_z│
                 └────────┬────────┘
                          │ PIO
                          ▼
                 ┌─────────────────┐
                 │  Contrôleur     │
                 │  d'effacement   │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    Framebuffer  │
                 │      RAM        │
                 └─────────────────┘
```

Lorsque `acc_z < -25`, le signal d'effacement est activé et le framebuffer est parcouru afin de remettre tous les pixels à zéro.

On obtient ainsi un comportement proche d'un véritable télécran : **le dessin est effacé lorsque la carte est retournée**.

## 🧩 Architecture globale

L'architecture finale peut être résumée ainsi :

```text
             ┌───────────────────────┐
             │       Encodeurs       │
             │                       │
             │   X              Y   │
             └──────┬──────────┬─────┘
                    │          │
                    ▼          ▼
             ┌───────────────────────┐
             │ Compteurs de position │
             └──────────┬────────────┘
                        │
                        ▼
             ┌───────────────────────┐
             │      Framebuffer      │
             │       Dual-Port       │
             └──────────┬────────────┘
                        │
                        ▼
             ┌───────────────────────┐
             │    Contrôleur HDMI    │
             └──────────┬────────────┘
                        │
                        ▼
                     HDMI
                        │
                        ▼
                     Écran


             ┌───────────────────────┐
             │      ADXL345          │
             └──────────┬────────────┘
                        │ I2C
                        ▼
             ┌───────────────────────┐
             │        Nios V         │
             └──────────┬────────────┘
                        │ PIO
                        ▼
                 Signal d'effacement
                        │
                        └──────────────► Framebuffer
```

## 🛠️ Environnement de développement

Le projet est réalisé avec :

* **Quartus Prime Lite 24.1**
* **VHDL**
* **Platform Designer**
* **Nios V**
* **C**
* **I2C**
* **ModelSim** pour les simulations
* **JTAG UART / `juart-terminal`**

Le FPGA utilisé est un **Cyclone V `5CSEBA6U23I7`**.

## 📁 Organisation du projet

Les fichiers du projet sont disponibles dans ce dépôt :

[TP_FPGA_telecran — GitHub](https://github.com/KellyLuo-O/TP_FPGA_telecranet?utm_source=chatgpt.com)

L'organisation exacte des fichiers peut dépendre de la version du projet présente dans le dépôt. Les principaux éléments correspondent aux différentes parties du TP :

```text
TP_FPGA_telecranet/
├── RTL / VHDL
├── Quartus
├── Platform Designer / Nios
├── Software C
└── Simulations
```

## 📋 Bilan

Ce TP a permis de mettre en œuvre une chaîne complète de conception FPGA, depuis la description matérielle en VHDL jusqu'à l'intégration d'un processeur embarqué et de périphériques externes.

Les principales fonctionnalités réalisées sont :

* [x] Prise en main de Quartus
* [x] Gestion du reset et des horloges
* [x] Chenillard
* [x] Gestion des encodeurs en quadrature
* [x] Génération de l'affichage HDMI
* [x] Déplacement du pixel
* [x] Mémorisation avec une RAM dual-port
* [x] Effacement du framebuffer
* [x] Intégration d'un Nios V
* [x] Communication I2C
* [x] Lecture de l'accéléromètre ADXL345
* [x] Affichage de l'inclinaison sur les LEDs
* [x] Effacement du télécran lorsque la carte est retournée

