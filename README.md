# ESP32 Soil Irrigation Controller

Firmware ESPHome per il controllo automatico di una pompa 12V DC, con driver MOSFET e irrigazione basata su un sensore capacitivo di umidità del terreno.

## Struttura del progetto

| Percorso | Cos'è |
|---|---|
| `controller_pompa.yaml` | Questo è il file da flashare per l'uso reale. ESP32 + MOSFET + pompa + 1 sensore di umidità. |
| `DOCS/wiring.yaml` (+ `.html`/`.png`) | Wiring: schemi collegamenti |
| `bench/` | Varianti di sviluppo/test — **non rappresentano l'hardware del progetto principale**. Usano un relè meccanico invece del MOSFET per comodità di banco (riscontro immediato, nessuna pompa/alimentatore da collegare). Vedi sotto. |
| `archive/` | Materiale storico/superato (vecchio wiring a relè, datasheet). Non cancellato, tenuto come riferimento. |

### Contenuto di `bench/`

| File | Cos'è |
|---|---|
| `wiring_lab_test.yaml` | Banco relè + pompa reale, solo timer (nessun sensore) — per isolare problemi hardware puri. |
| `soil_1sensor.yaml` / `wiring_soil_1sensor.yaml` | Logica irrigazione con 1 sensore di umidità capacitivo, su banco relè — origine del porting nel flagship. |
| `soil_3sensor.yaml` / `wiring_soil_3sensor.yaml` | Come sopra ma con 3 sensori (OR: irriga se almeno una zona è secca) — non ancora sul flagship, da valutare se promuovere. |

> ⚠️ Il porting sensore→MOSFET nel flagship è validato solo via `esphome config`/`compile` (compila correttamente), **non ancora su hardware fisico**. Vedi TODO in `CLAUDE.md`.

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

## Utilizzo

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

**Perché il sensore di umidità si accende/spegne ad ogni lettura invece di restare sempre alimentato** (vedi `controller_pompa.yaml` e `bench/soil_*sensor.yaml`): è un duty cycle scelto per preservare nel tempo le piste capacitive del sensore da corrosione/elettrolisi dovute a tensione continua in ambiente umido — **non** è una scelta di risparmio energetico (il consumo del sensore, pochi mA, è trascurabile rispetto a quello della pompa quando gira).

**Cosa succede se il cutoff di sicurezza (`max_on_ms`) spegne la pompa mentre il terreno è ancora "secco"**: la pompa **non riparte da sola**. Il trigger che l'accende scatta solo sulla transizione bagnato→secco, non a livello continuo — se il sensore non è mai sceso sotto la soglia bagnata, quella transizione non si ripresenta. Serve un ciclo completo (il terreno torna bagnato, poi di nuovo secco) perché la pompa riparta. È il comportamento atteso, non un guasto: se il cutoff scatta spesso, è un segnale che le soglie o il sensore vanno controllati, non che manca un "riavvio automatico" da aggiungere.

Per i TODO aperti (validazione fisica del porting, alimentazione a batteria per il ramo MOSFET, calibrazione soglie, eventuale promozione del 3-sensori), vedi `CLAUDE.md`.

---

## Licenza

Questo progetto è distribuito con licenza MIT (vedi `LICENSE`).

> ⚠️ La licenza **non copre** i datasheet e il materiale di terze parti in `archive/SPEC/` (documentazione del modulo relè Elegoo), inclusi solo come riferimento storico — restano di proprietà dei rispettivi detentori dei diritti.
