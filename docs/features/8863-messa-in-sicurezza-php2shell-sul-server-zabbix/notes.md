> Ticket: oc:8863

# Note — Messa in sicurezza PHP2Shell sul server zabbix

> Repo pubblico: esito, versioni, esposizione e verifica di compromissione sono nel ticket oc:8863.

## Divergenze dal piano, task per task

### Task 3: verifica degli aggiornamenti automatici

Il piano verificava la sospensione con `unattended-upgrade --dry-run`. Lanciato a mano, però,
`unattended-upgrade` non legge `APT::Periodic::Unattended-Upgrade` e mostra comunque i pacchetti che
installerebbe: la verifica non dimostrava nulla. Il valore che conta è quello letto dal servizio
notturno (`/usr/lib/apt/apt.systemd.daily`), controllato con
`apt-config dump | grep APT::Periodic::Unattended-Upgrade` → `"0"`.

### Task 5: valori di partenza del php.ini

`expose_php` era già `Off` nel `php.ini` di Apache. `session.upload_progress.enabled` era solo
commentato (`;session.upload_progress.enabled = On`), quindi attivo per default: la sostituzione
prevista dal piano non lo trovava. È stata aggiunta una riga esplicita
`session.upload_progress.enabled = Off` sotto quella commentata, poi `systemctl reload apache2`.

### Task 6: versione mostrata da phpversion()

Il piano prevedeva che `phpversion()` mostrasse solo `8.1.2`; dal modulo Apache mostra invece la
versione completa del pacchetto (`8.1.2-1ubuntu2.28`), che rende la verifica via web più diretta.

## Bug trovati

- Sorgenti apt non adatte all'architettura: il server è arm64, ma le sorgenti puntavano a
  `mirror.hetzner.com/ubuntu/…`, che pubblica solo amd64. Corrette su
  `mirror.hetzner.com/ubuntu-ports/…`. Senza questa correzione la patch non era installabile.

## Decisioni

- Documentazione nel repo `devops`, pubblico, senza dati sensibili (scrum del 26/10, scelta del
  dev); i dettagli restano nel ticket.
- Snapshot Hetzner prima dell'intervento, aumentando di uno il limite di snapshot del progetto;
  cancellato a fine lavoro (scrum del 26/10). Prima della decisione allo scrum lo snapshot era
  bloccato dal limite e si era scelto di procedere con la sola copia a mano delle evidenze: scelta
  superata lo stesso giorno.
- Aggiornamenti automatici sospesi prima di correggere le sorgenti, per evitare che il server
  installasse da solo, di notte, gli aggiornamenti accumulati (scelta del dev, 26/10).
- Apache fermo dalla raccolta delle evidenze fino alla patch, come su oc:8843: si ferma solo il
  frontend, la raccolta dati di Zabbix continua.
- Login di verifica attraverso un tunnel SSH, perché il frontend è solo in HTTP.
- Installazione con `NEEDRESTART_MODE=l` e `--force-confold`: nessun servizio riavviato in automatico,
  `php.ini` esistente conservato.
- Durante `apt update` e la simulazione di `unattended-upgrade` Zabbix ha segnalato per circa 14
  minuti un carico alto dei processi «discoverer», chiuso da solo: il server ha 2 vCPU.

## Follow-up

- Le attività lasciate aperte sono elencate nel ticket oc:8863, sezione «Resta aperto».
