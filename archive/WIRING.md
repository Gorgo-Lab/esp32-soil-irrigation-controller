# Cablaggio completo — Controller Pompa di Sentina

## Architettura generale

```
Batteria 12V
    │
    ├──► Buck converter 12V→5V ──► ESP32 (VIN) + Relay (JD-VCC)
    │
    └──► Relay COM ──► [contatti relè] ──► Pompa (+)
                                               │
                                           [diodo] ← back-EMF
                                               │
    GND ◄──────────────────────────────── Pompa (-)
```

---

## ⚠️ Avvertenze prima di collegare

- La pompa da 1100 GPH a 12V assorbe circa **3.5–5A**. Il relè Songle SRD-05VDC ha un limite di **3A su carichi induttivi DC**: siamo al limite o sopra. Se la pompa dovesse bruciare ancora i contatti del relè, il passo successivo è interporre un **contattore DC da 10A+** pilotato dal relè.
- Un motore DC genera un picco di tensione inversa (back-EMF) all'apertura del relè. **Senza il diodo di ricircolo i contatti si bruciano progressivamente** — questa è probabilmente la causa dei fumi precedenti.
- Inserire sempre un **fusibile** sul positivo della batteria verso il circuito di potenza.

---

## Componenti necessari

| Componente | Specifiche minime |
|---|---|
| Buck converter | 12V → 5V, almeno 1A (es. modulo LM2596) |
| Diodo di ricircolo | 1N5408 (3A, 1000V) o equivalente |
| Fusibile + portafusibile | 5A, tipo automobilistico |
| Cavi pompa/batteria | **1.5mm²** (sezione minima per 5A) |
| Cavi segnale (ESP32 ↔ relay) | 0.25mm² (normali cavetti Arduino) |

---

## Schema di collegamento

### 1. Alimentazione — Batteria 12V → Buck converter → ESP32 e modulo relè

| Da | A | Cavo |
|---|---|---|
| Batteria (+) | Fusibile 5A ingresso | 1.5mm² |
| Fusibile 5A uscita | Buck converter IN+ | 1.5mm² |
| Batteria (−) | Buck converter IN− | 1.5mm² |
| Buck converter OUT+ (5V) | ESP32 **VIN** | cavetto Arduino |
| Buck converter OUT+ (5V) | Relay J2 **JD-VCC** | cavetto Arduino |
| Buck converter OUT− (GND) | ESP32 **GND** | cavetto Arduino |
| Buck converter OUT− (GND) | Relay J2 **GND** | cavetto Arduino |

> Rimuovere il jumper tra VCC e JD-VCC sul modulo relè.

### 2. Segnale — Connettore J1 (10 pin): ESP32 → Modulo relè

| J1 pin | Segnale | ESP32 | Cavo |
|---|---|---|---|
| 1 | GND | GND | cavetto Arduino |
| 2 | IN1 | GPIO26 | cavetto Arduino |
| 10 | VCC | 3.3V | cavetto Arduino |

### 3. Potenza — Relè → Pompa

| Da | A | Cavo |
|---|---|---|
| Batteria (+) — dopo fusibile | Relay 1 **COM** | 1.5mm² |
| Relay 1 **NO** | Pompa (+) | 1.5mm² |
| Pompa (−) | Batteria (−) / GND comune | 1.5mm² |

### 4. Diodo di ricircolo (back-EMF)

Montare il diodo **direttamente sui terminali della pompa**, il più vicino possibile:

| Diodo | Collegamento |
|---|---|
| **Catodo** (banda) | Pompa (+) |
| **Anodo** | Pompa (−) |

Il diodo è normalmente spento. Quando il relè apre, il motore spinge corrente in direzione inversa: il diodo la fa circolare in sicurezza invece di scaricarsi sui contatti del relè.

---

## Schema riassuntivo

```
Batteria 12V (+) ──[fuse 5A]──┬──► Buck IN+
                              └──► Relay COM ──► Relay NO ──► Pompa (+)
                                                                  │
                                                              [1N5408]  ← catodo in alto
                                                                  │
Batteria 12V (−) ─────────────┴──► Buck IN− ──────────────── Pompa (−)
                                       │
                              Buck OUT+ (5V) ──► ESP32 VIN
                              Buck OUT+ (5V) ──► Relay JD-VCC
                              Buck OUT− ──────► ESP32 GND
                              Buck OUT− ──────► Relay J2 GND

ESP32 3.3V ──► Relay J1 pin 10 (VCC)
ESP32 GND  ──► Relay J1 pin 1  (GND)
ESP32 GPIO26 ► Relay J1 pin 2  (IN1)
```

---

## Setup da laboratorio (USB + alimentatore da banco)

Configurazione per sviluppo e test: ESP32 collegato via USB al PC, alimentatore da banco a 12V per il circuito di potenza.

**Alimentatore da banco**: impostare **12V**, limitatore di corrente a **1A** come protezione durante i test.

### Ponticello (jumper) sul modulo relè

In laboratorio si usa il **jumper inserito**, che cortocircuita VCC e JD-VCC permettendo di alimentare tutto il modulo da un'unica fonte 5V.

L'isolamento galvanico (jumper rimosso) serve a proteggere l'ESP32 e l'operatore nel caso in cui un guasto sul lato potenza — arco elettrico, contatto bruciato — si propaghi alla logica e da lì all'USB del PC o a chi lavora sul circuito. A 12V DC questo rischio non esiste: la tensione non è pericolosa e un eventuale guasto non può danneggiare l'ESP32. Il jumper inserito è quindi sicuro in laboratorio e semplifica i collegamenti. In produzione a 230V il jumper va rimosso.

### Alimentazione

| Da | A | Note |
|---|---|---|
| USB PC | ESP32 | dati + alimentazione ESP32 |
| ESP32 **5V** (VBUS) | Relay J2 **VCC** | alimenta logica + bobine via jumper |
| ESP32 **GND** | Relay J2 **GND** | |
| Alimentatore (+) | Relay 1 **COM** | 12V circuito pompa |
| Alimentatore (−) | ESP32 **GND** | massa comune obbligatoria |

> Con il jumper inserito J2 VCC alimenta tutto il modulo. Il pin J1-10 (VCC) non va collegato all'ESP32 — è già alimentato internamente via jumper.

### Segnale — J1

| J1 pin | ESP32 |
|---|---|
| 1 (GND) | GND |
| 2 (IN1) | GPIO26 |
| 10 (VCC) | **non collegare** — già alimentato via jumper da J2 |

### Circuito pompa

| Da | A |
|---|---|
| Relay 1 **NO** | Pompa (+) |
| Pompa (−) | Alimentatore (−) / GND comune |
| Diodo 1N5408 catodo | Pompa (+) |
| Diodo 1N5408 anodo | Pompa (−) |

> Il diodo di ricircolo va montato anche in laboratorio — altrimenti i test non sono rappresentativi del comportamento reale.

### Schema riassuntivo lab

```
USB PC ──► ESP32 ──► GPIO26 ──► Relay IN1
              │
              5V (VBUS) ──► Relay JD-VCC
              GND ─────────┬─► Relay GND
                           └─► Alimentatore (−)

Alimentatore (+) 12V ──► Relay COM ──► Relay NO ──► Pompa (+)
                                                         │
                                                     [1N5408]
                                                         │
Alimentatore (−) ────────────────────────────────── Pompa (−)
```

---

## Setup da laboratorio con isolamento galvanico (configurazione production-like)

Identico al setup precedente ma con **jumper rimosso** e alimentazione separata per le bobine del relè. È il modo più fedele per replicare in lab il comportamento del sistema in produzione.

Serve un 5V dedicato per JD-VCC separato dall'ESP32. La soluzione più pratica con un solo alimentatore da banco è aggiungere un piccolo modulo buck 12V→5V.

**Componente aggiuntivo**: modulo buck 12V→5V (es. LM2596), alimentato dallo stesso banco a 12V.

### Ponticello

**Jumper rimosso** — VCC (logica optocoupler) e JD-VCC (bobine relè) sono rail separati e alimentati da fonti distinte.

### Alimentazione

| Da | A | Note |
|---|---|---|
| USB PC | ESP32 | dati + alimentazione ESP32 |
| Alimentatore (+) 12V | Buck IN+ | |
| Alimentatore (−) | Buck IN− | |
| Buck OUT+ (5V) | Relay J2 **JD-VCC** | solo bobine, isolato dalla logica |
| Buck OUT− (GND) | Relay J2 **GND** | |
| Buck OUT− (GND) | ESP32 **GND** | massa comune tra i due domini |
| Alimentatore (+) 12V | Relay 1 **COM** | circuito pompa |
| Alimentatore (−) | ESP32 **GND** | |

### Segnale — J1

| J1 pin | Collegamento |
|---|---|
| 1 (GND) | GND comune |
| 2 (IN1) | ESP32 GPIO26 |
| 10 (VCC) | Buck OUT+ (5V) |

### Circuito pompa

| Da | A |
|---|---|
| Relay 1 **NO** | Pompa (+) |
| Pompa (−) | Alimentatore (−) / GND comune |

> ⚠️ Il diodo di ricircolo 1N5408 è omesso in questa configurazione di test. Da aggiungere obbligatoriamente nell'installazione definitiva per proteggere i contatti del relè dal back-EMF della pompa.

### Schema riassuntivo lab con isolamento

```
USB PC ──► ESP32 ──► GPIO26 ──► Relay J1-IN1
              │
              GND ──────────────────────────────────────┐
                                                        │
Alimentatore (+) 12V ──► Buck IN+                      │
Alimentatore (−) ────► Buck IN− ── Buck OUT− (GND) ────┴─► Relay J2 GND + J1 GND
                         │
                         Buck OUT+ (5V) ──┬─► Relay J2 JD-VCC
                                          └─► Relay J1 VCC (pin 10)

Alimentatore (+) 12V ──► Relay COM ──► Relay NO ──► Pompa (+)
                                                         │
Alimentatore (−) / GND comune ───────────────────── Pompa (−)
```
