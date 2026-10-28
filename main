#include <Arduino.h>
#include <IRremote.hpp>

// --- AFFECTATION DES BROCHES ---
#define PIN_PIR      2    // Capteur PIR sur GPIO 2
#define PIN_IR       4    // Récepteur Infrarouge sur GPIO 4
#define PIN_BUZZER  25    // Buzzer sur GPIO 25
#define PIN_SERVO_1  18   // Servomoteur haut sur GPIO 18
#define PIN_SERVO_2  19   // Servomoteur bas sur GPIO 19

// --- CONFIGURATION PWM (API ESP32 Core v3.x) ---
#define FREQ_SERVO     50   // 50 Hz standard pour les servomoteurs
#define RES_SERVO      16   // Résolution 16 bits pour un contrôle précis des impulsions
#define RESOLUTION_BUZ  8   // 8 bits pour le buzzer

// --- CODES DE LA TÉLÉCOMMANDE IR (Protocole NEC Wokwi) ---
#define IR_BTN_PLAY  0xA8 
#define IR_BTN_PLUS  0x02 
#define IR_BTN_MOINS 0x98 

#define IR_BTN_1     0x30 
#define IR_BTN_2     0x18 
#define IR_BTN_3     0x7A 
#define IR_BTN_4     0x10 
#define IR_BTN_5     0x38 

// --- RÉFÉRENTIEL DES SERVOMOTEURS ---
#define REF_ZERO_S1      90  
#define REF_ZERO_S2      90  
#define REF_PLUS_90_S1  180  
#define REF_MOINS_90_S2   0  

#define DUREE_OUVERTURE_MS 10000UL 

// --- ÉTATS DU SYSTÈME (Type énuméré C) ---
typedef enum {
    ETAT_BAS,
    ETAT_HAUT_TEMPO,
    ETAT_HAUT_FORCE
} EtatChatiere;

// Variables globales du système
EtatChatiere etat_actuel = ETAT_BAS;
unsigned long chrono_ouverture = 0;
int dernier_etat_pir = LOW;
int son_selectionne = 1;

// --- FONCTIONS DE CONTRÔLE DES SERVOS EN C PUR (PWM Direct) ---

// Convertit un angle (0 à 180°) en rapport cyclique (duty) 16 bits pour un signal de 50Hz (période = 20000 µs)
// Impulsion standard Servo SG90 : 500 µs (0°) à 2400 µs (180°)
static uint32_t angle_vers_duty(int angle) {
    if (angle < 0) angle = 0;
    if (angle > 180) angle = 180;
    
    // Calcul de la largeur d'impulsion en microsecondes
    uint32_t pulse_us = 500 + ((uint32_t)angle * (2400 - 500)) / 180;
    
    // Conversion en valeur brute PWM 16 bits (65535 pour 20000 µs)
    return (pulse_us * 65535UL) / 20000UL;
}

// Initialise et positionne un servo
void initialiser_servo(int pin, int angle_initial) {
    ledcAttach(pin, FREQ_SERVO, RES_SERVO);
    ledcWrite(pin, angle_vers_duty(angle_initial));
}

// Détache le servo (coupe le signal PWM pour le rendre non-résistif)
void relacher_servo(int pin) {
    ledcDetach(pin);
}

// Écrit un angle sur le servo
void ecrire_servo(int pin, int angle) {
    if (!ledcGetPinFreq(pin)) { // Vérifie si la broche est attachée
        ledcAttach(pin, FREQ_SERVO, RES_SERVO);
    }
    ledcWrite(pin, angle_vers_duty(angle));
}

// --- GÉNÉRATEUR SONORE DOUX (C Pur) ---
void emettre_note_feutree(int frequence, int duree_ms) {
    if (frequence <= 0) {
        ledcWrite(PIN_BUZZER, 0);
        delay(duree_ms);
        return;
    }

    ledcAttach(PIN_BUZZER, frequence, RESOLUTION_BUZ);
    ledcWrite(PIN_BUZZER, 35); // Rapport cyclique ~14% pour adoucir le son
    delay(duree_ms);
    ledcWrite(PIN_BUZZER, 0);
    ledcDetach(PIN_BUZZER);
}

// Profils sonores
void son_1_gazouillis(void) {
    int notes[] = {988, 1319, 1568, 1976};
    for (int i = 0; i < 4; i++) {
        emettre_note_feutree(notes[i], 45);
        delay(25);
    }
}

void son_2_arpege(void) {
    int notes[] = {523, 659, 784, 1047};
    int durees[] = {75, 75, 75, 140};
    for (int i = 0; i < 4; i++) {
        emettre_note_feutree(notes[i], durees[i]);
        delay(20);
    }
}

void son_3_glissando(void) {
    for (int f = 700; f <= 1400; f += 50) {
        emettre_note_feutree(f, 15);
    }
}

void son_4_zen(void) {
    emettre_note_feutree(880, 90);
    delay(80);
    emettre_note_feutree(1320, 160);
}

void son_5_ronron(void) {
    for (int c = 0; c < 5; c++) {
        emettre_note_feutree(130, 40);
        delay(35);
        emettre_note_feutree(95, 35);
        delay(30);
    }
}

void jouer_son_courant(void) {
    switch (son_selectionne) {
        case 1: son_1_gazouillis(); break;
        case 2: son_2_arpege();     break;
        case 3: son_3_glissando();  break;
        case 4: son_4_zen();        break;
        case 5: son_5_ronron();     break;
        default: son_1_gazouillis(); break;
    }
}

// --- LOGIQUE MÉCANIQUE DE LA CHATIÈRE ---
void mettre_position_haute(void) {
    jouer_son_courant();

    if (etat_actuel == ETAT_BAS) {
        for (int delta = 0; delta <= 90; delta += 2) {
            ecrire_servo(PIN_SERVO_1, REF_ZERO_S1 + delta); 
            ecrire_servo(PIN_SERVO_2, REF_ZERO_S2 - delta); 
            delay(15);
        }
    } else {
        ecrire_servo(PIN_SERVO_1, REF_PLUS_90_S1);
        ecrire_servo(PIN_SERVO_2, REF_MOINS_90_S2);
    }
}

void mettre_position_basse(void) {
    for (int delta = 90; delta >= 0; delta -= 2) {
        ecrire_servo(PIN_SERVO_1, REF_ZERO_S1 + delta);
        ecrire_servo(PIN_SERVO_2, REF_ZERO_S2 - delta);
        delay(15);
    }

    delay(200);

    // Relâchement des moteurs (coupe le PWM -> non-résistifs)
    relacher_servo(PIN_SERVO_1);
    relacher_servo(PIN_SERVO_2);
    printf(">>> Chatiere en position BASSE. Moteurs relaches.\n");
}

// --- FONCTIONS PRINCIPALES ARDUINO (C Standard) ---
void setup(void) {
    Serial.begin(115200);
    pinMode(PIN_PIR, INPUT);
    IrReceiver.begin(PIN_IR, DISABLE_LED_FEEDBACK);

    // Initialisation en position basse puis relâchement
    initialiser_servo(PIN_SERVO_1, REF_ZERO_S1);
    initialiser_servo(PIN_SERVO_2, REF_ZERO_S2);
    delay(500);
    relacher_servo(PIN_SERVO_1);
    relacher_servo(PIN_SERVO_2);

    printf("=== Chatiere Prete (Code 100% C Pur) ===\n");
}

void loop(void) {
    unsigned long maintenant = millis();

    // 1. DÉTECTION PIR
    int etat_pir = digitalRead(PIN_PIR);
    if (etat_pir == HIGH && dernier_etat_pir == LOW) {
        printf("[PIR] Mouvement detecte !\n");
        if (etat_actuel != ETAT_HAUT_FORCE) {
            if (etat_actuel == ETAT_BAS) {
                mettre_position_haute();
            }
            etat_actuel = ETAT_HAUT_TEMPO;
            chrono_ouverture = maintenant;
        }
    }
    dernier_etat_pir = etat_pir;

    // 2. TÉLÉCOMMANDE IR
    if (IrReceiver.decode()) {
        uint8_t cmd = IrReceiver.decodedIRData.command;

        if (!(IrReceiver.decodedIRData.flags & IRDATA_FLAGS_IS_REPEAT)) {
            switch (cmd) {
                case IR_BTN_PLAY:
                    printf("[IR] PLAY : Ouverture 10s\n");
                    mettre_position_haute();
                    etat_actuel = ETAT_HAUT_TEMPO;
                    chrono_ouverture = maintenant;
                    break;

                case IR_BTN_PLUS:
                    printf("[IR] [+] : Mode HAUT FORCE\n");
                    mettre_position_haute();
                    etat_actuel = ETAT_HAUT_FORCE;
                    break;

                case IR_BTN_MOINS:
                    printf("[IR] [-] : Position BASSE\n");
                    if (etat_actuel != ETAT_BAS) {
                        mettre_position_basse();
                        etat_actuel = ETAT_BAS;
                    }
                    break;

                case IR_BTN_1:
                    son_selectionne = 1;
                    printf("[SON] 1 : Gazouillis\n");
                    son_1_gazouillis();
                    break;

                case IR_BTN_2:
                    son_selectionne = 2;
                    printf("[SON] 2 : Carillon Do Majeur\n");
                    son_2_arpege();
                    break;

                case IR_BTN_3:
                    son_selectionne = 3;
                    printf("[SON] 3 : Glissando\n");
                    son_3_glissando();
                    break;

                case IR_BTN_4:
                    son_selectionne = 4;
                    printf("[SON] 4 : Quinte Zen\n");
                    son_4_zen();
                    break;

                case IR_BTN_5:
                    son_selectionne = 5;
                    printf("[SON] 5 : Ronronnement\n");
                    son_5_ronron();
                    break;

                default:
                    break;
            }
        }
        IrReceiver.resume();
    }

    // 3. MINUTEUR D'OUVERTURE (10 SECONDES)
    if (etat_actuel == ETAT_HAUT_TEMPO) {
        if (maintenant - chrono_ouverture >= DUREE_OUVERTURE_MS) {
            printf("[TIMER] 10s ecoulees -> Fermeture\n");
            mettre_position_basse();
            etat_actuel = ETAT_BAS;
        }
    }

    delay(20);
}
