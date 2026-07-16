# ELEGOO 8-Channel 5V Relay Module

https://eu.elegoo.com/en-es/blogs/arduino-projects/elegoo-dc-5v-relay-module-tutorial?gad_source=1&gad_campaignid=20846060146&gbraid=0AAAAAqAZSFxIElatxyYIUy7vLB3ExWvIe

## Cos'è

Il modulo è una scheda con 8 relè elettromeccanici controllabili da un microcontrollore (ESP32, Arduino, ecc.). Permette di pilotare carichi ad alta tensione/corrente (230V AC, motori, pompe) usando i segnali logici a 3.3V o 5V del microcontrollore, senza alcun collegamento elettrico diretto tra le due parti.

L'isolamento è garantito da **optoisolatori**: il segnale di controllo accende un LED interno all'optoisolatore, che a sua volta attiva un fototransistor sul lato relè — i due circuiti non si toccano mai fisicamente.

## Relè montati: Songle SRD-05VDC-SL-C

Ogni canale monta un relè Songle SRD, bobina 5V. Ogni relè ha tre morsetti di uscita:

| Morsetto | Significato | Stato a riposo |
|----------|-------------|----------------|
| **COM**  | Comune (polo mobile) | — |
| **NO**   | Normally Open | aperto (circuito interrotto) |
| **NC**   | Normally Closed | chiuso (circuito in conduzione) |

Quando il relè viene attivato, COM si sposta da NC a NO. Per controllare un carico (es. pompa) che deve essere spento a riposo, si cablano COM e NO.

### Specifiche elettriche dei contatti

| Parametro | Valore |
|-----------|--------|
| Carico resistivo max | 10A @ 125VAC — 10A @ 28VDC — 7A @ 240VAC |
| Carico induttivo max | 3A @ 120VAC — 3A @ 28VDC |
| Tensione massima | 250VAC / 110VDC |
| Vita elettrica | 100.000 operazioni a carico nominale |
| Vita meccanica | 10.000.000 operazioni a vuoto |

Per carichi induttivi (motori, pompe) la corrente ammessa è circa un terzo rispetto ai carichi resistivi — dimensionare di conseguenza.

## Come funziona il controllo

Il segnale IN del microcontrollore entra nell'optoisolatore attraverso una resistenza di limitazione (~102Ω). L'optoisolatore pilota la base di un transistor NPN (8050) che alimenta la bobina del relè. Un diodo flyback (1N4148) protegge il transistor dallo spike di tensione generato dalla bobina al momento dello spegnimento.

Il modulo supporta due modalità di trigger, selezionabili tramite jumper sulla scheda:

- **High-level trigger**: il relè si attiva con IN = HIGH (logica positiva)
- **Low-level trigger**: il relè si attiva con IN = LOW (active-low, logica invertita)

Questo progetto usa **low-level trigger** (`inverted: true` nel YAML ESPHome su GPIO26).

## Alimentazione: VCC e JD-VCC

Il modulo ha due rail di alimentazione separati, collegati tramite un jumper:

- **VCC**: alimenta la logica degli optoisolatori (3.3V o 5V dal microcontrollore)
- **JD-VCC**: alimenta le bobine dei relè (richiede 5V, 60mA per canale attivo)

**Con il jumper presente** (configurazione default): VCC e JD-VCC sono cortocircuitati — un'unica fonte alimenta tutto. È la configurazione più semplice ma riduce l'isolamento galvanico.

**Senza jumper**: le bobine vengono alimentate da una sorgente esterna dedicata collegata a JD-VCC e GND. In questo caso il GND del microcontrollore va collegato al GND del modulo relè per mantenere il riferimento comune. Questa configurazione garantisce isolamento galvanico completo tra ESP32 e il circuito di potenza — consigliata quando si pilotano carichi a 230VAC.

## Connettore di controllo (J1)

| Pin | Segnale |
|-----|---------|
| 1   | GND     |
| 2–9 | IN1–IN8 |
| 10  | VCC     |

Il connettore laterale (J2, 3 pin) espone GND, VCC e JD-VCC per la gestione separata dell'alimentazione delle bobine.
