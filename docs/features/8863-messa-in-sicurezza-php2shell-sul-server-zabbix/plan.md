> Ticket: oc:8863

# Messa in sicurezza PHP2Shell sul server zabbix — piano

> **Per chi esegue:** lavoro operativo via SSH (alias `zabbix`, utente root), non c'è codice da
> scrivere né una suite di test: ogni task è una sequenza di comandi con la sua verifica. Si
> esegue un task alla volta, nell'ordine, fermandosi su ogni verifica che non torna.

**Obiettivo:** il PHP usato da Apache per il frontend Zabbix passa a `8.1.2-1ubuntu2.28` (correzione
Ubuntu di PHP2Shell), con il monitoraggio che continua a funzionare.

**Approccio:** ordine vincolante del tag php2shell — snapshot, evidenze fuori dalla macchina,
patch. Aggiornamento mirato dei soli pacchetti PHP da `jammy-security` per arm64, dopo aver
sospeso gli aggiornamenti automatici e corretto le sorgenti apt.

**Strumenti:** console Hetzner (dev), `ssh`/`scp`, `apt-get`, `dpkg`, `apache2ctl`, `systemctl`,
`psql`.

**Spec:** [overview.md](overview.md)

## Vincoli globali

- Repo pubblico: in questo repo e nei messaggi di commit nessun IP, nessuna versione vulnerabile,
  nessuna porta, nessun esito della verifica di compromissione. Quelli vanno nel ticket oc:8863.
- Evidenze sul Mac del dev in `~/Documents/evidenze-php2shell/8863-zabbix/`, mai nel repo.
- Pacchetti attesi, e solo questi (simulazione del 26/10): `libapache2-mod-php8.1 php8.1-bcmath
  php8.1-cli php8.1-common php8.1-gd php8.1-ldap php8.1-mbstring php8.1-opcache php8.1-pgsql
  php8.1-readline php8.1-xml`, tutti a `8.1.2-1ubuntu2.28`.
- Mirror arm64: `https://mirror.hetzner.com/ubuntu-ports/packages` e
  `https://mirror.hetzner.com/ubuntu-ports/security`.
- Nessun comando che scrive sul server senza l'ok esplicito del dev al task che lo contiene.
- Nessun commit o push automatico: i commit sono istruzioni per il dev.

## Punti da guardare in review

- Apache che non riparte dopo l'aggiornamento (`Address already in use`, oc:8834): il frontend
  resta giù mentre `zabbix-server` continua a raccogliere. Coperto nel Task 5.
- Un prompt di `ucf`/`dpkg` sul `php.ini` durante l'installazione: deve tenere il file attuale.
  Coperto nel Task 4 con `--force-confold`.
- `needrestart` che riavvia servizi non previsti: modalità solo elenco. Coperto nel Task 4.
- Aggiornamenti automatici che ripartono dopo la correzione delle sorgenti: coperto nel Task 3,
  con la verifica che il prossimo giro di `unattended-upgrades` non installi nulla.
- La pagina di prova dimenticata nella document root: coperto nel Task 6 con la verifica che
  risponda 404.

---

### Task 1: Snapshot Hetzner (dev)

- [ ] **Passo 1:** dalla console Hetzner, progetto devops, server `zabbix` → Snapshots → crea
  snapshot `zabbix-pre-8863`. Se compare «Snapshot limit reached», chiedi l'aumento di uno dalla
  pagina dei limiti (decisione dello scrum del 26/10) e riprova.
- [ ] **Verifica:** lo snapshot compare nell'elenco come disponibile. L'ID va nel ticket, non qui.

### Task 2: Livello di partenza, stop di Apache, evidenze fuori dalla macchina

**Produce:** `~/Documents/evidenze-php2shell/8863-zabbix/evidenze-8863.tgz` sul Mac, con
`SHA256SUMS` verificato; `/root/evidenze-8863/` sul server.

- [ ] **Passo 1 — livello di partenza degli errori PHP:**
  `grep -c "PHP " /var/log/apache2/error.log` e il numero per ciascuna rotazione non compressa.
  Annotalo: serve al Task 7.
- [ ] **Passo 2 — stop di Apache** (precedente oc:8843): `systemctl stop apache2`.
  Verifica: `ss -ltn | grep ':80 '` vuoto; `systemctl is-active zabbix-server` → `active`.
- [ ] **Passo 3 — raccolta** in `/root/evidenze-8863/`: `/var/log/apache2/` intera, `journalctl
  --since 2026-10-10` in un file, `ps auxwwf`, `ss -tanp`, crontab di tutti gli utenti e
  `/etc/cron*`, `/etc/passwd`, `/etc/group`, `/etc/shadow` (solo date: `chage -l` per utente),
  `authorized_keys` di root e degli utenti, `/etc/apache2`, `/etc/php`, `/etc/zabbix`,
  `/usr/share/zabbix`, `dpkg -l`, `find / -xdev -newermt 2026-10-20 -type f` (escludendo `/proc`,
  `/sys`, `/var/lib/postgresql`). Poi `tar czf /root/evidenze-8863.tgz -C /root evidenze-8863` e
  `sha256sum`.
- [ ] **Passo 4 — copia sul Mac:** `scp zabbix:/root/evidenze-8863.tgz` nella cartella del
  vincolo globale, `shasum -a 256` sul Mac.
- [ ] **Verifica:** l'hash calcolato sul Mac coincide con quello calcolato sul server.

### Task 3: Aggiornamenti automatici sospesi e sorgenti apt corrette

> ⚠️ L'implementazione ha deviato da questo task: [notes.md](notes.md#task-3-verifica-degli-aggiornamenti-automatici)

**Consuma:** Apache fermo dal Task 2.

- [ ] **Passo 1:** copia `/etc/apt/apt.conf.d/20auto-upgrades` in `20auto-upgrades.pre-8863`
  (fuori da `apt.conf.d`, in `/root/`), poi imposta `APT::Periodic::Unattended-Upgrade "0";`.
  Lascia `Update-Package-Lists "1"`.
- [ ] **Passo 2:** copia `/etc/apt/sources.list` in `/root/sources.list.pre-8863`, poi sostituisci
  `mirror.hetzner.com/ubuntu/packages` → `mirror.hetzner.com/ubuntu-ports/packages` e
  `mirror.hetzner.com/ubuntu/security` → `mirror.hetzner.com/ubuntu-ports/security`. Le righe di
  `repo.zabbix.com` restano come sono.
- [ ] **Passo 3:** `apt-get update`.
  Verifica: nessun `404` o `Err` nell'output; `apt-cache policy libapache2-mod-php8.1` mostra
  `Candidate: 8.1.2-1ubuntu2.28`.
- [ ] **Passo 4:** `unattended-upgrade --dry-run --debug 2>&1 | tail`.
  Verifica: compare che gli aggiornamenti automatici sono disabilitati, oppure nessun pacchetto
  da installare.

### Task 4: Aggiornamento dei soli pacchetti PHP

- [ ] **Passo 1 — simulazione:** `apt-get -s install --only-upgrade <pacchetti attesi>`.
  Verifica: `11 upgraded, 0 newly installed, 0 to remove`, e l'elenco coincide con quello del
  vincolo globale. Se compare altro, fermati e porta l'elenco al dev.
- [ ] **Passo 2 — installazione** (ok del dev sull'output del passo 1):
  `NEEDRESTART_MODE=l apt-get install --only-upgrade -o Dpkg::Options::=--force-confold <pacchetti
  attesi>`.
- [ ] **Verifica:** `dpkg -l <pacchetti attesi> | awk '/^ii/{print $2, $3}'` → tutti
  `8.1.2-1ubuntu2.28`; `systemctl is-active postgresql zabbix-server zabbix-agent` → tre `active`.

### Task 5: php.ini e ripartenza di Apache

> ⚠️ L'implementazione ha deviato da questo task: [notes.md](notes.md#task-5-valori-di-partenza-del-phpini)

- [ ] **Passo 1:** in `/etc/php/8.1/apache2/php.ini` imposta `session.upload_progress.enabled = Off`
  ed `expose_php = Off`.
  Verifica: `diff` con la copia nelle evidenze mostra solo queste due righe.
- [ ] **Passo 2:** `apache2ctl configtest` → `Syntax OK`; poi `systemctl start apache2`.
- [ ] **Verifica:** `systemctl is-active apache2` → `active`; `ss -ltnp | grep ':80 '` mostra
  processi `apache2`; nessun processo `apache2` avviato prima dello start
  (`ps -o pid,lstart -C apache2`); `grep -i "address already in use" /var/log/apache2/error.log`
  vuoto dopo lo start.

### Task 6: Verifica della correzione

> ⚠️ L'implementazione ha deviato da questo task: [notes.md](notes.md#task-6-versione-mostrata-da-phpversion)

- [ ] **Passo 1:** `zcat /usr/share/doc/libapache2-mod-php8.1/changelog.Debian.gz | head -20`
  contiene `CVE-2026-109618` e `CVE-2026-109619`.
- [ ] **Passo 2:** crea `/var/www/html/<nome casuale>.php` con
  `<?php echo PHP_SAPI, ' ', phpversion(), ' ', ini_get('expose_php'), ' ',
  ini_get('session.upload_progress.enabled');`, interrogalo con `curl -s
  http://localhost/<nome>.php` dal server stesso, poi cancellalo.
  Verifica: la risposta inizia con `apache2handler 8.1.2`; le due direttive risultano spente; dopo
  la cancellazione lo stesso `curl` risponde 404.
  Nota: `phpversion()` mostra `8.1.2` anche dopo la patch; la prova della versione è nel passo 1
  e nel Task 4.

### Task 7: Verifica del monitoraggio

- [ ] **Passo 1 (dev):** `ssh -L 8080:localhost:80 zabbix`, poi `http://localhost:8080/zabbix/`
  nel browser, login. Verifica: la dashboard mostra «Zabbix server is running: Yes» e i grafici
  dell'ultima ora sono continui fino a ora.
- [ ] **Passo 2:** `sudo -u postgres psql zabbix -Atc "select to_timestamp(max(clock)) from
  history"` (da `/tmp`). Verifica: valore degli ultimi minuti.
- [ ] **Passo 3:** conteggio degli errori `PHP ` in `error.log` dopo lo start di Apache.
  Verifica: nessun errore nuovo rispetto al livello di partenza del Task 2.

### Task 8: Verifica di compromissione

Sulle evidenze copiate nel Mac, non sul server.

- [ ] **Passo 1:** nei log di accesso dal 10/10, richieste con `allow_url_include`,
  `auto_prepend_file` o `%AD`: conteggio per codice di risposta e dimensione della risposta delle
  200, confrontata con una GET normale su `/`.
- [ ] **Passo 2:** file nuovi o modificati dal 20/10 (dalla `find` del Task 2), crontab, utenti e
  chiavi: tutto ciò che non si spiega con attività nota.
- [ ] **Verifica:** se emerge qualcosa di non spiegato, fermati e portalo al dev prima del Task 9.
  L'esito va solo nel ticket.

### Task 9: Chiusura

- [ ] **Passo 1:** nel ticket oc:8863 (`prepend`): esito, ID dello snapshot, verifiche, esito della
  verifica di compromissione, e la riga «aggiornamenti automatici spenti in attesa
  dell'aggiornamento completo controllato del sistema».
- [ ] **Passo 2 (dev, dopo l'ok):** cancellazione dello snapshot `zabbix-pre-8863` dalla console
  Hetzner (decisione dello scrum del 26/10).
- [ ] **Passo 3:** `notes.md` di questa cartella, senza dati sensibili.
- [ ] **Passo 4 — legame repo ↔ server (scrum del 26/10):**
  - `README.md` nella root del repo: a quale server corrisponde (progetto Hetzner devops, server
    `zabbix`), quali servizi ci girano (Zabbix, PostHog), dove sta il clone sul server
    (`/root/devops`). Nessun IP.
  - `CLAUDE.md` nella root del repo, con in cima le regole: repo pubblico, nessun dato sensibile
    (IP, versioni vulnerabili, porte, esiti di verifiche di sicurezza: vanno nei ticket); dopo ogni
    merge su `main`, `git -C /root/devops pull` sul server, a carico di chi ha modificato la
    documentazione.
- [ ] **Passo 5 — commit (istruzione per il dev, non automatica):** sul branch
  `feature/oc-8863-messa-in-sicurezza-php2shell-sul-server-zabbix`,
  `docs(oc:8863): overview, piano e note della messa in sicurezza PHP2Shell di zabbix`, poi PR e
  merge su `main`.
- [ ] **Passo 6 — clone sul server, dopo il merge:** `git clone
  https://github.com/webmappsrl/devops.git /root/devops` (HTTPS, nessuna credenziale sul server;
  fuori da `/var/www/html`, che Apache pubblica).
  Verifica: `git -C /root/devops log -1 --oneline` mostra il commit di questo ticket e
  `git -C /root/devops rev-parse --abbrev-ref HEAD` → `main`.
