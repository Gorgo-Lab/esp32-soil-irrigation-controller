# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Struttura del repository

- **`controller_pompa.yaml`** (root) — flagship. Unico file da flashare per l'uso reale.
- **`DOCS/wiring.yaml`** (+ `.html`/`.png` generati) — wiring corrispondente al flagship.
- **`bench/`** — varianti di sviluppo/test, tutte basate su relè meccanico (non MOSFET) per comodità di banco: nessuna pompa/alimentatore da collegare, riscontro immediato (scatto del relè) mentre si itera sulla logica. Non rappresentano l'hardware del flagship. Include la logica a sensori di umidità del terreno (1 e 3 sensori), non ancora portata sul MOSFET.
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

Il firmware del flagship è definito in `controller_pompa.yaml` (ESPHome YAML, no C++ scritto a mano).

- **Target**: ESP32 (esp32dev, Arduino framework)
- **Switch di potenza**: driver MOSFET (XY-MOS), GPIO27, active-high (`inverted: false`), `restore_mode: ALWAYS_OFF` — non un relè meccanico (quello è usato solo nelle varianti di sviluppo in `bench/`)
- **Cycle**: on boot → `pump_on_phase` script runs indefinitely, alternating ON/OFF via mutual script recursion
- **Timing**: controlled by `substitutions` at the top of the file (`pump_on_time`, `pump_off_time`)
- **Modalità test**: jumper GPIO4→GND attiva timing rapido (5s/5s) per test da banco; floating = timing normale (5min/10min)

To change timing, edit the `substitutions` block — no logic changes needed elsewhere.

## TODO aperti (non risolti dal riordino repo)

- Portare la logica a sensori di umidità (validata in `bench/soil_*sensor.yaml` su relè) sul MOSFET del flagship — non è un copia-incolla: la polarità del trigger cambia (relè low-level vs MOSFET active-high).
- Ridisegnare la catena di alimentazione da batteria reale (fusibile + buck converter) per il ramo MOSFET — `DOCS/wiring.yaml` documenta solo ESP32+MOSFET+pompa su alimentatore da banco, non l'alimentazione a batteria da campo (vedi `archive/WIRING.md` per il riferimento concettuale, era per il relè).
