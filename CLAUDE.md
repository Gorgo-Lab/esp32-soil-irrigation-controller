# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Struttura del repository

- **`controller_pompa.yaml`** (root) — flagship. Unico file da flashare per l'uso reale.
- **`DOCS/wiring.yaml`** (+ `.html`/`.png` generati) — wiring corrispondente al flagship.
- **`bench/`** — varianti di sviluppo/test, tutte basate su relè meccanico (non MOSFET) per comodità di banco: nessuna pompa/alimentatore da collegare, riscontro immediato (scatto del relè) mentre si itera sulla logica. Non rappresentano l'hardware del flagship. `soil_1sensor.yaml` è stato portato sul flagship (vedi sotto); `soil_3sensor.yaml` resta solo su relè, da valutare se promuovere in futuro.
- **`archive/`** — materiale storico/superato (vecchio wiring basato su relè, datasheet del modulo relè Elegoo). Non cancellato, tenuto come riferimento.
- Vedi `README.md` per la mappa completa dei file e le istruzioni d'uso/sviluppo.

## Commands

```bash
# Flash firmware + open serial log
pipenv run esphome run controller_pompa.yaml

# Serial log only (device already flashed)
pipenv run esphome logs controller_pompa.yaml

# Validate config without flashing
pipenv run esphome config controller_pompa.yaml

# Compile without flashing (verifica che il C++ generato compili)
pipenv run esphome compile controller_pompa.yaml

# Explicit serial port (if not auto-detected)
pipenv run esphome run controller_pompa.yaml --device /dev/ttyUSB0   # Linux
pipenv run esphome run controller_pompa.yaml --device COM3            # Windows
```

## Architecture

Il firmware del flagship è definito in `controller_pompa.yaml` (ESPHome YAML, no C++ scritto a mano). Irrigazione guidata da 1x sensore capacitivo di umidità del terreno (soglia con isteresi), non più da timer cieco.

- **Target**: ESP32 (esp32dev, Arduino framework)
- **Switch di potenza**: driver MOSFET (XY-MOS), id `pump_mosfet`, GPIO27, active-high (`inverted: false`), `restore_mode: ALWAYS_OFF` — non un relè meccanico (quello è usato solo nelle varianti di sviluppo in `bench/`)
- **Sensore**: capacitivo, ADC su GPIO32, alimentato a ciclo (duty cycle) via GPIO25 diretto — acceso solo per la finestra di lettura (~2.2s), non in continuo, per preservarlo da corrosione/elettrolisi
- **Logica**: `binary_sensor.analog_threshold` con isteresi (soglie in `substitutions`) guida `pump_mosfet` via `on_press`/`on_release`; cooldown minimo tra accensioni e cutoff di sicurezza a tempo massimo indipendenti dal sensore
- **Timing**: controllato da `substitutions` in testa al file (`read_interval_*`, `cooldown_*`, `max_on_*`)
- **Modalità test**: jumper GPIO4→GND attiva timing rapido (lettura 5s, cooldown 5s, cutoff 30s) per test da banco; floating = timing normale (lettura 10min, cooldown 5min, cutoff 5min)

Per cambiare i tempi, modifica il blocco `substitutions` — nessuna modifica alla logica altrove.

## TODO aperti (non risolti dal riordino repo)

- **Validare su hardware reale** il porting sensori→MOSFET: finora solo `esphome config`/`compile` (compila, non garantisce comportamento fisico corretto — es. polarità trigger, timing reale del MOSFET). Da testare fisicamente prima di fidarsene in campo.
- Valutare se promuovere `bench/soil_3sensor.yaml` (multi-zona, OR su 3 sensori) sul flagship — solo se l'esperienza con 1 sensore mostra che serve monitorare punti diversi.
- Ridisegnare la catena di alimentazione da batteria reale (fusibile + buck converter) per il ramo MOSFET — `DOCS/wiring.yaml` documenta solo ESP32+MOSFET+pompa+sensore su alimentatore da banco, non l'alimentazione a batteria da campo (vedi `archive/WIRING.md` per il riferimento concettuale, era per il relè).
- Calibrare le soglie di umidità (`soil_dry_threshold`/`soil_wet_threshold`) sul sensore fisico effettivamente installato — i valori attuali sono validati su un'unità di test, non garantiti identici su un altro esemplare.
