# Controller Pompa ESP32

Firmware ESPHome per il controllo automatico di una pompa 12V DC, con driver MOSFET, timer, e (in sviluppo) irrigazione basata su sensori di umidità del terreno.

## Struttura del progetto

| Percorso | Cos'è |
|---|---|
| `controller_pompa.yaml` | **Flagship** — questo è il file da flashare per l'uso reale. |
| `DOCS/wiring.yaml` (+ `.html`/`.png`) | Wiring corrispondente al flagship (ESP32 + driver MOSFET XY-MOS + pompa). |
| `bench/` | Varianti di sviluppo/test — **non rappresentano l'hardware del flagship**. Usano un relè meccanico invece del MOSFET per comodità di banco (riscontro immediato, nessuna pompa/alimentatore da collegare). Vedi sotto. |
| `archive/` | Materiale storico/superato (vecchio wiring a relè, datasheet). Non cancellato, tenuto come riferimento. |

### Contenuto di `bench/`

| File | Cos'è |
|---|---|
| `wiring_lab_test.yaml` | Banco relè + pompa reale, solo timer (nessun sensore) — per isolare problemi hardware puri. |
| `soil_1sensor.yaml` / `wiring_soil_1sensor.yaml` | Logica irrigazione con 1 sensore di umidità capacitivo, su banco relè. |
| `soil_3sensor.yaml` / `wiring_soil_3sensor.yaml` | Come sopra ma con 3 sensori (OR: irriga se almeno una zona è secca). |

> ⚠️ Nessuna delle varianti in `bench/` è ancora stata portata sul MOSFET del flagship. Vedi TODO in `CLAUDE.md`.

## Installazione ambiente

### Linux

```bash
# 1. dipendenze di sistema
sudo apt update && sudo apt install -y build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev curl \
  libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev

# 2. pyenv
curl https://pyenv.run | bash
# aggiungi a ~/.bashrc o ~/.zshrc:
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
# riapri il terminale, poi verifica:
pyenv --version

# 3. installa Python
pyenv install 3.12.0

# 4. crea la cartella di progetto e imposta Python localmente
mkdir controller-pompa && cd controller-pompa
pyenv local 3.12.0

# 5. pipenv + esphome
pip install pipenv
pipenv install esphome

# 6. permessi porta seriale (richiede logout/login)
sudo usermod -aG dialout $USER
```

### Windows

```powershell
# 1. pyenv-win (PowerShell come amministratore)
Invoke-WebRequest -UseBasicParsing -Uri "https://raw.githubusercontent.com/pyenv-win/pyenv-win/master/pyenv-win/install-pyenv-win.ps1" -OutFile "./install-pyenv-win.ps1"; &"./install-pyenv-win.ps1"
# riapri PowerShell, poi verifica:
pyenv --version

# 2. installa Python
pyenv install 3.12.0

# 3. crea la cartella di progetto e imposta Python localmente
mkdir controller-pompa && cd controller-pompa
pyenv local 3.12.0

# 4. pipenv + esphome
pip install pipenv
pipenv install esphome
```

> **Driver USB**: installa il driver CH340 o CP210x in base al chip USB del tuo ESP32.

---

## Utilizzo (flagship)

### Flash + log (tutto in uno)

```bash
pipenv run esphome run controller_pompa.yaml
```

### Solo log (device già flashato)

```bash
pipenv run esphome logs controller_pompa.yaml
```

### Porta seriale esplicita

Se ESPHome non rileva automaticamente la porta:

```bash
# Linux
pipenv run esphome run controller_pompa.yaml --device /dev/ttyUSB0

# Windows
pipenv run esphome run controller_pompa.yaml --device COM3
```

Per usare una delle varianti in `bench/`, sostituisci il percorso del file, es. `pipenv run esphome run bench/soil_1sensor.yaml`.

---

## Sviluppo

### Validare un config senza flashare

Controlla sintassi, pin, riferimenti tra componenti — non richiede l'ESP32 collegato:

```bash
pipenv run esphome config controller_pompa.yaml
```

### Compilare senza flashare

Verifica che il C++ generato compili davvero — cattura errori che `esphome config` non vede (es. lambda scritte male). Richiede l'ESP32 collegato? No, ma richiede la toolchain platformio (già scaricata al primo utilizzo):

```bash
pipenv run esphome compile controller_pompa.yaml
```

### Rigenerare i diagrammi di wiring (WireViz via Docker)

I file `DOCS/wiring.yaml` e quelli in `bench/wiring_*.yaml` sono sorgenti [WireViz](https://github.com/wireviz/WireViz). Per rigenerare `.html`/`.png` dopo una modifica:

```bash
docker run --rm -v "$(pwd)":/root/src znibb/wireviz:latest "wireviz DOCS/wiring.yaml -f hp"
```

`-f hp` genera sia HTML che PNG. Sostituisci il percorso per rigenerare un altro file (es. `bench/wiring_soil_1sensor.yaml`).

---

## Note di progettazione

**Perché i sensori di umidità si accendono/spengono ad ogni lettura invece di restare sempre alimentati** (vedi `bench/soil_*sensor.yaml`): è un duty cycle scelto per preservare nel tempo le piste capacitive dei sensori da corrosione/elettrolisi dovute a tensione continua in ambiente umido — **non** è una scelta di risparmio energetico (il consumo dei sensori, pochi mA, è trascurabile rispetto a quello della pompa quando gira).

Per i TODO aperti (porting della logica sensori sul MOSFET, alimentazione a batteria per il ramo MOSFET), vedi `CLAUDE.md`.
