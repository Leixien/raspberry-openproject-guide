# Guida passo-passo “per principianti” per installare OpenProject su Raspberry Pi

Questa guida è pensata per chi **non ha mai usato un Raspberry Pi** e vuole trasformarlo in un piccolo server di project‑management con OpenProject.  Seguendo i passaggi descritti qui potrai accendere il tuo Pi, installare il sistema operativo, preparare Docker, avviare OpenProject e rendere il tutto accessibile in modo sicuro ai colleghi tramite VPN.

> **Hardware consigliato:** Raspberry Pi 4 con almeno **4 GB di RAM**, scheda micro‑SD da **32 GB** o più grande, alimentatore 5 V 3 A e cavo Ethernet.  Un computer con lettore SD e Internet serve per preparare la scheda.

## 1 – Installare Raspberry Pi OS

### 1.1 Preparare la micro‑SD con Raspberry Pi Imager

1. Scarica e installa **Raspberry Pi Imager** dal sito ufficiale sul tuo PC.
2. Inserisci la micro‑SD nel PC e apri l’imager.
3. Clicca **Choose OS** e seleziona **Raspberry Pi OS (64‑bit)**.
4. Apri le **impostazioni avanzate** (icona ⚙️) e compila la tabella:

   | Impostazione        | Valore                      | Descrizione                                  |
   |--------------------|----------------------------|----------------------------------------------|
   | **Hostname**       | `project`                  | Nome di rete del Raspberry                  |
   | **Username**       | `admin-user`               | Utente con cui accederai via SSH            |
   | **Password**       | scegli una passphrase      | Usa almeno 12 caratteri                      |
   | **Enable SSH**     | ✔️ Password authentication | Abilita l’accesso remoto                     |
   | **Locale**         | Fuso orario Europe/Rome, tastiera IT | Imposta lingua e tastiera italiane |

5. Torna alla schermata principale e clicca **Write**.  Attendi la scrittura e poi espelli la micro‑SD.

### 1.2 Primo avvio

* Inserisci la micro‑SD nel Raspberry, collega il cavo Ethernet al router e alimenta il Pi.  Dopo circa un minuto il sistema è pronto.
* Dal tuo computer apri un terminale e connettiti via SSH:

  ```bash
  ssh admin-user@project.local
  ```

* Accetta la chiave e inserisci la password scelta.  Lavorerai quasi sempre via SSH.

## 2 – Aggiornare e configurare il Raspberry

### 2.1 Aggiornare i pacchetti

Nel terminale del Raspberry esegui:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
```

Questo aggiorna tutti i pacchetti e rimuove i file non più necessari.

### 2.2 Impostare un IP fisso

Per evitare che l’indirizzo cambi ad ogni riavvio, apri il pannello del tuo router (sezione DHCP) e riserva l’indirizzo `192.168.1.100` per il Raspberry.  Un IP fisso è indispensabile per il port‑forwarding e la VPN【115596516407720†L104-L115】.

### 2.3 Aumentare la memoria swap (opzionale)

Con 4 GB di RAM è consigliabile creare una swap di 2 GB per evitare arresti improvvisi.  Esegui:

```bash
sudo dphys-swapfile swapoff
sudo sed -i 's/^CONF_SWAPSIZE=.*/CONF_SWAPSIZE=2048/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

## 3 – Installare Docker e Docker Compose

OpenProject è distribuito come container.  Installerai Docker Engine e il plugin Compose.

### 3.1 Aggiungere il repository Docker

```bash
sudo apt install ca-certificates curl gnupg lsb-release -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```

### 3.2 Installare Docker e Compose

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
sudo usermod -aG docker admin-user
newgrp docker   # ricarica i gruppi nella sessione corrente
docker run hello-world
```

Il comando `hello-world` deve stampare un messaggio di test: se compare, l’installazione funziona.  Aggiungendo `admin-user` al gruppo **docker** non dovrai digitare `sudo` ad ogni comando.

## 4 – Preparare l’ambiente per OpenProject

### 4.1 Creare la cartella del progetto e le variabili di configurazione

```bash
mkdir -p ~/project-server
cd ~/project-server

# scegli una password per Postgres e genera una secret key
export OP_DB_PASSWORD='ProjectDb0123'
export OP_SECRET_KEY=$(openssl rand -hex 32)

cat <<EOF > .env
OP_DB_PASSWORD=${OP_DB_PASSWORD}
OP_SECRET_KEY=${OP_SECRET_KEY}
EOF
```

Il file `.env` terrà le credenziali del database e la chiave segreta.

## 5 – Scrivere il file `docker-compose.yml`

Userai due container: uno per PostgreSQL e uno per OpenProject.  Segui la struttura raccomandata dalla documentazione【703879020181655†L1446-L1451】.

```yaml
services:
  projectdb:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: openproject
      POSTGRES_PASSWORD: ${OP_DB_PASSWORD}
      POSTGRES_DB: openproject
    volumes:
      - ./db-data:/var/lib/postgresql/data

  project-box:
    image: openproject/openproject:16
    depends_on:
      - projectdb
    restart: always
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: "postgres://openproject:${OP_DB_PASSWORD}@projectdb:5432/openproject"
      SECRET_KEY_BASE: ${OP_SECRET_KEY}
      OPENPROJECT_HOST__NAME: "project.local:8080"
      OPENPROJECT_HTTPS: "false"
    volumes:
      - ./op-assets:/var/openproject/assets
```

**Salva** il file come `docker-compose.yml` nella cartella `~/project-server`.

## 6 – Avviare OpenProject

Lancia i container in background:

```bash
cd ~/project-server
docker compose up -d
```

Il primo avvio può durare 10‑15 minuti.  Per monitorare lo stato usa:

```bash
docker compose logs -f project-box
```

Quando appare `OpenProject has been started`, apri il browser sul tuo computer e visita **http://project.local:8080**.  Fai login con **admin / admin** e cambia la password.

## 7 – Configurare l’accesso remoto

### 7.1 Registrare un dominio DuckDNS

1. Crea un account su [DuckDNS](https://www.duckdns.org/) e genera il dominio, ad esempio `project-server.duckdns.org`.
2. Annota il token.  Configura un cron job per aggiornare automaticamente l’IPv4 e disattivare l’IPv6 (che può causare errori su WireGuard).  Sul Pi:

   ```bash
   sudo tee /etc/cron.d/duckdns >/dev/null <<'EOT'
   */5 * * * * root curl -s "https://duckdns.org/update?domains=project-server&token=<YOUR_DUCKDNS_TOKEN>&ip=&ipv6=" >/dev/null
   EOT
   sudo run-parts /etc/cron.d
   ```

   Sostituisci `<YOUR_DUCKDNS_TOKEN>` con il token fornito da DuckDNS.  L’opzione `ipv6=` vuota garantisce che non venga creato il record AAAA.

### 7.2 Installare PiVPN e WireGuard

Esegui sul Raspberry:

```bash
curl -L https://install.pivpn.io | bash
```

Durante il wizard:

1. Seleziona **WireGuard** come protocollo.
2. Scegli un DNS provider, ad esempio **Quad9**.
3. Per «Public IP or DNS» seleziona **DNS Entry** e inserisci `project-server.duckdns.org`.
4. Porta: lascia **51820/UDP** (ricorda di aprirla sul router in port‑forwarding verso `192.168.1.100`).
5. Conferma le opzioni rimanenti e attendi la fine della configurazione.

### 7.3 Creare profili client

Per ogni dispositivo che dovrà usare la VPN:

```bash
pivpn add
```

Inserisci un nome (ad esempio `iphone`), premi Invio per la durata del certificato e attendi la generazione.  PiVPN ti dirà dove ha salvato il file `.conf` (es. `/home/admin-user/configs/iphone.conf`) e potrai generare un QR‑code con:

```bash
pivpn -qr iphone
```

Scansiona il QR con l’app **WireGuard** su iOS/Android o trasferisci il file `.conf` sul dispositivo.  Dopo l’importazione appare un tunnel pronto da attivare.

### 7.4 Configurare un tunnel completo (full‑tunnel)

Per instradare tutto il traffico del dispositivo attraverso la VPN (utile su reti pubbliche):

1. **Abilita l’inoltro IP** sul Raspberry:

   ```bash
   sudo tee /etc/sysctl.d/99-wireguard.conf >/dev/null <<'EOF'
   net.ipv4.ip_forward=1
   net.ipv6.conf.all.forwarding=1
   EOF
   sudo sysctl --system
   ```

2. **Aggiungi le regole NAT** in `wg0.conf`:

   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```

   Dentro la sezione `[Interface]` inserisci queste righe se non ci sono:

   ```ini
   PostUp   = iptables -t nat -A POSTROUTING -s 10.196.234.0/24 -o eth0 -j MASQUERADE
   PostDown = iptables -t nat -D POSTROUTING -s 10.196.234.0/24 -o eth0 -j MASQUERADE
   ```

   Salva e chiudi.

3. **Riavvia WireGuard**:

   ```bash
   sudo systemctl restart wg-quick@wg0
   ```

4. **Modifica il profilo sul dispositivo**: in `[Interface]` aggiungi `MTU = 1280` e verifica che in `[Peer]` ci siano:

   ```ini
   AllowedIPs = 0.0.0.0/0, ::/0
   PersistentKeepalive = 25
   ```

Ora tutte le app (anche WhatsApp e Safari) funzioneranno mentre la VPN è attiva.

Se preferisci un **tunnel parziale (split‑tunnel)** per usare la VPN solo per accedere a OpenProject, modifica la riga `AllowedIPs` in:

```ini
AllowedIPs = 10.196.234.0/24, 192.168.1.0/24
```

Così il dispositivo userà Internet normalmente e invierà al Raspberry solo il traffico destinato a OpenProject o ad altri host nella tua LAN.

## 8 – Best practice e manutenzione

* **Aggiorna regolarmente** con `sudo apt update && sudo apt full-upgrade -y`.  Per aggiornare OpenProject: `docker compose pull` seguito da `docker compose up -d`.
* **Backup** dei dati di OpenProject:

  ```bash
  docker compose exec project-box openproject run rake backups:create
  ```

  Troverai un archivio `.tar.gz` nella cartella `backups/`.  Copialo periodicamente su un altro disco.
* **Non esporre direttamente la porta 8080 su Internet**.  Accedi a OpenProject tramite VPN oppure configura un reverse‑proxy HTTPS.
* **Documentati**: la guida ufficiale raccomanda di separare i container (database e app) per rendere il servizio scalabile【703879020181655†L1446-L1451】.  Se l’IP pubblico del tuo provider cambia spesso, un servizio come DuckDNS con port‑forwarding è fondamentale【115596516407720†L104-L115】.

## 9 – Conclusione

Hai trasformato il Raspberry Pi in un server di project‑management robusto e sicuro.  Da principiante hai imparato a: installare il sistema operativo, configurare Docker, avviare OpenProject, registrare un dominio dinamico, installare una VPN con WireGuard e, se necessario, instradare tutto il traffico attraverso il Pi.  Con questa base potrai aggiungere monitoraggio, backup automatici e altre integrazioni.  Buon lavoro!
