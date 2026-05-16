# STM32 Camera Driver (OV7725 / OV5640) - Robust HAL Implementation

[![Embedded](https://shields.io)](https://st.com)
[![Language](https://shields.io)](https://wikipedia.org)
[![License](https://shields.io)](LICENSE)

Cette bibliothèque fournit un pilote (driver) optimisé et robuste pour contrôler les capteurs d'images **OmniVision OV7725** et **OV5640** sur microcontrôleurs **STM32** en utilisant l'écosystème **STM32CubeHAL**.

L'implémentation bas niveau résout les problèmes de synchronisation classiques de l'I2C (conditions de *RESTART* natives) et s'adapte automatiquement aux capteurs utilisant des registres 8 bits ou 16 bits.

---

## ✨ Caractéristiques clés

- **Gestion I2C Correcte (HAL Fix)** : Utilisation exclusive de `HAL_I2C_Mem_Read` / `HAL_I2C_Mem_Write`. Évite les pertes de trames causées par l'enchaînement manuel d'un Transmit + Receive (Stop/Start défectueux).
- **Auto-Détection du Capteur** : Lecture dynamique des ID fabricants (`manuf_id`) et périphériques (`device_id`). Support transparent des registres 16 bits de l'OV5640.
- **Large Palette de Résolutions** : Prise en charge de plus de 30 résolutions standard, du très petit format pour le traitement FFT ($64\times32$) jusqu'au 5 Mégapixels ($2592\times1944$).
- **Contrôles Avancés Image (OV7725)** : Fonctions prêtes à l'emploi pour le miroir horizontal, le retournement vertical (VFLIP) et l'inversion d'octets (Byte Swap RGB565).

---

## 📂 Structure des fichiers

- `camera.h` : Définitions globales, formats de pixels (`RGB565`, `JPEG`, `YUV422`), résolutions (`FRAMESIZE_VGA`, etc.) et structures de contrôle.
- `camera.c` : Fonctions d'accès aux registres I2C de bas niveau et routine de détection automatique `Camera_read_id`.
- `ov7725.h` : Registres complets (DSP, exposition, fenêtrage, mire de test) et prototypes d'initialisation spécifiques à l'OV7725.

---

## 🚀 Guide de démarrage rapide

### 1. Configuration matérielle (STM32CubeMX)
1. Activez votre périphérique **I2C** (ex: `I2C1`) en mode standard (100 kHz) ou rapide (400 kHz).
2. Configurez les broches du protocole parallèle **DVP** (ou interface GPIO équivalente) pour récupérer les signaux `PCLK`, `HREF`, `VSYNC` et les données `D0-D7`.
3. Configurez une horloge maître **XCLK** pour le capteur (via une sortie `MCO` ou un canal de `Timer` PWM) réglée généralement entre 10 MHz et 24 MHz.

### 2. Initialisation du Driver

Déclarez le handle global et configurez-le dans votre fichier `main.c` :

```c
#include "camera.h"
#include "ov7725.h"

// Instance I2C générée par CubeMX
extern I2C_HandleTypeDef hi2c1; 

int main(void) {
    // 1. Initialisation standard du microcontrôleur
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C1_Init();

    // 2. Configuration du handle de la caméra
    hcamera.hi2c = &hi2c1;
    hcamera.addr = OV7725_ADDRESS; // Adresse de base (0x42)
    hcamera.timeout = 100;         // Timeout I2C en ms

    // 3. Identification du capteur connecté
    if (Camera_read_id(&hcamera) == Camera_OK) {
        printf("Capteur détecté avec succès !\n");
        printf("Manufacturer ID : 0x%04X\n", hcamera.manuf_id);
        printf("Device ID       : 0x%04X\n", hcamera.device_id);
    } else {
        printf("Erreur : Aucun capteur détecté.\n");
        Error_Handler();
    }

    // 4. Initialisation des registres du capteur en QVGA (320x240)
    if (ov7725_init(FRAMESIZE_QVGA) == Camera_OK) {
        hcamera.framesize = FRAMESIZE_QVGA;
        hcamera.pixformat = PIXFORMAT_RGB565;
    }

    // Boucle principale
    while (1) {
        // Votre logique de capture d'image (DVP / DMA) ici
    }
}
```

### 3. Exemples d'utilisation des contrôles d'image

Le pilote permet de modifier dynamiquement l'orientation ou le format de sortie de l'image pour s'adapter directement aux écrans LCD sans coût CPU :

```c
// Corriger l'effet miroir si la caméra est face à l'utilisateur
ov7725_set_hmirror(1); 

// Retourner l'image si le capteur est monté à l'envers sur le PCB
ov7725_set_vflip(1);

// Permuter l'ordre des octets (MSB/LSB) pour une compatibilité directe LCD
ov7725_set_byte_swap(1);
```

---

## 💡 Conseils d'optimisation pour la communauté

1. **Vitesse I2C** : Si vos câbles de liaison sont courts, configurez l'I2C à **400 kHz** (Fast Mode) dans CubeMX pour accélérer considérablement le temps d'initialisation des listes de registres.
2. **Alignement d'adresses** : La HAL STM32 décale automatiquement l'adresse I2C vers la gauche pour intégrer le bit R/W. Laissez `OV7725_ADDRESS` à `0x42`. Ne forcez jamais de `+1` manuel sur l'adresse passée aux fonctions HAL.
3. **Capture DMA** : Pour de meilleures performances, utilisez l'interface **DCMI** (Digital Camera Interface) du STM32 couplée au **DMA** en mode circulaire afin de transférer directement les pixels vers la mémoire RAM ou la mémoire de l'écran.

---

## 🤝 Contribution

Les contributions sont les bienvenues ! Si vous souhaitez ajouter le support complet des listes de registres pour l'**OV5640** ou l'**OV7670** en utilisant cette structure de bas niveau robuste, n'hésitez pas à :
1. Forker le projet.
2. Créer une branche pour votre fonctionnalité (`git checkout -b feature/AmazingFeature`).
3. Ouvrir une *Pull Request*.

## 📄 Licence

Distribué sous la licence MIT. Voir le fichier `LICENSE` pour plus d'informations.
## 📸 Gestion de la Capture d'Image (DCMI + DMA)

Pour afficher ou traiter les images, vous devez configurer le périphérique **DCMI** (Digital Camera Interface) et le **DMA** associé dans STM32CubeMX (Mode: Circular, Data Width: Word/Half-Word).

### 1. Variables globales et Callbacks
Le DMA utilise des fonctions de rappel (callbacks) pour vous avertir lorsqu'une trame est entièrement reçue en mémoire RAM.

```c
#include "main.h"
#include "camera.h"

// Handle DCMI généré par CubeMX
extern DCMI_HandleTypeDef hdcmi;

// Flag pour savoir si le DMA est en cours de transfert
volatile bool dma_busy = false;
volatile bool frame_ready = false;

// Buffer en mémoire pour stocker l'image (Ex: QVGA RGB565 = 320 x 240 pixels x 2 octets = 153600 octets)
// Note: Alignez le buffer sur 32 octets si vous activez le cache D-Cache sur STM32F7/H7
ALIGN_32BYTES (uint16_t frame_buffer[320 * 240]); 

// Callback appelé par la HAL à la fin de la réception d'une image complète
void HAL_DCMI_FrameEventCallback(DCMI_HandleTypeDef *hdcmi) {
    // Arrêter la capture après une trame si vous êtes en mode Snapshot (Normal)
    HAL_DCMI_Stop(hdcmi); 
    dma_busy = false;
    frame_ready = true;
}

// Callback de secours en cas d'erreur de synchronisation ou de débordement DMA
void HAL_DCMI_ErrorCallback(DCMI_HandleTypeDef *hdcmi) {
    HAL_DCMI_Stop(hdcmi);
    dma_busy = false;
    // Réinitialiser le matériel si nécessaire ici
}
```

### 2. Méthode A : Capture en mode Normal / Snapshot (Bloquant)
Idéal pour prendre une photo à la demande, traiter l'image, puis attendre l'action suivante de l'utilisateur.

```c
void Capture_Snapshot(void) {
    if (dma_busy) return;

    frame_ready = false;
    dma_busy = true;

    // Déclenche la capture d'UNE SEULE TRAME (DCMI_MODE_SNAPSHOT)
    // L'adresse de destination doit être castée en uint32_t. La taille s'exprime en MOTS (32-bits)
    // Donc : (Largeur * Hauteur * 2 octets par pixel) / 4 octets par mot
    uint32_t buffer_size_words = (320 * 240 * 2) / 4;

    if (HAL_DCMI_Start_DMA(&hdcmi, DCMI_MODE_SNAPSHOT, (uint32_t)frame_buffer, buffer_size_words) != HAL_OK) {
        dma_busy = false;
        printf("Erreur lors du démarrage du DMA DCMI\n");
        return;
    }

    // Attente (bloquante mais sécurisée) de la fin du transfert par l'interruption
    while (!frame_ready) {
        // Optionnel : ajouter un timeout logiciel pour éviter le blocage infini
    }

    // À ce stade, frame_buffer contient votre image RGB565 prête à être envoyée à l'écran !
    printf("Image capturée avec succès !\n");
}
```

### 3. Méthode B : Capture en mode Continu / Interruption (Flux Vidéo)
Idéal pour streamer un flux vidéo fluide en temps réel sur un écran LCD. Le DMA remplit le buffer en arrière-plan sans intervention du CPU.

```c
void Start_Video_Stream(void) {
    frame_ready = false;
    dma_busy = true;
    
    uint32_t buffer_size_words = (320 * 240 * 2) / 4;

    // Utilisation du mode DCMI_MODE_CONTINUOUS
    // Le DMA va boucler indéfiniment sur votre buffer
    HAL_DCMI_Start_DMA(&hdcmi, DCMI_MODE_CONTINUOUS, (uint32_t)frame_buffer, buffer_size_words);
}

// Dans votre boucle principale (main.c)
while (1) {
    if (frame_ready) {
        frame_ready = false; // Reset le flag

        // TRÈS IMPORTANT : Si vous utilisez un STM32F7 ou H7 avec D-Cache activé,
        // vous devez vider le cache avant de lire le buffer écrit par le DMA :
        // SCB_CleanInvalidateDCache_by_Addr((uint32_t*)frame_buffer, sizeof(frame_buffer));

        // Transférer directement le buffer vers l'écran LCD via SPI/FMC
        LCD_DrawImage(0, 0, 320, 240, frame_buffer);
        
        // Autoriser la détection de la trame suivante
        dma_busy = true; 
    }
}
```
