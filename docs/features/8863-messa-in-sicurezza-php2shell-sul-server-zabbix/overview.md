> Ticket: oc:8863

# Messa in sicurezza PHP2Shell (CVE-2026-109618/109619) sul server zabbix

> Questo repository è pubblico: qui non compaiono indirizzi, versioni vulnerabili, porte esposte
> né esiti della verifica di compromissione. Quei dettagli sono nel ticket oc:8863 su Orchestrator.

## Cosa cambia
Sul server `zabbix` del progetto Hetzner devops (Ubuntu 22.04, arm64), il PHP che Apache usa per
il frontend web di Zabbix (`mod_php`, pacchetti di sistema Ubuntu) viene portato alla versione
`8.1.2-1ubuntu2.28`, che contiene la correzione Ubuntu di PHP2Shell (USN-8948-1/-2, canale
`jammy-security`). Si aggiornano solo i pacchetti `php8.1-*` e `libapache2-mod-php8.1`. Nel
`php.ini` di Apache si spengono `session.upload_progress.enabled` ed `expose_php`. Zabbix,
PostgreSQL e il resto del sistema restano come sono.

## Perché
PHP2Shell permette di eseguire comandi su un server senza credenziali, con exploit pubblico e
attacchi in corso. Il frontend web di Zabbix è l'unico punto in cui questo server usa PHP. Va messo
in sicurezza fra i primi (scrum del 26/10).

## Requisiti
- [ ] Snapshot Hetzner del server prima di qualsiasi modifica, aumentando di uno il limite di
  snapshot del progetto se serve (scrum del 26/10); cancellato a fine lavoro, se tutto è andato bene.
- [ ] Evidenze copiate fuori dalla macchina prima di toccare i pacchetti, con hash sha256
  ricalcolati dopo la copia: log Apache, journal, processi, connessioni, crontab, utenti, chiavi,
  configurazione di Apache, PHP e Zabbix, elenco dei pacchetti. Restano fuori da questo repo.
- [ ] Sorgenti apt allineate al mirror per arm64 (`ubuntu-ports`), con verifica che
  `8.1.2-1ubuntu2.28` sia disponibile prima di installare.
- [ ] Aggiornamenti automatici sospesi (`Unattended-Upgrade "0"`) prima di correggere le sorgenti,
  e segnalati nel ticket come spenti finché non si fa l'aggiornamento completo controllato del
  sistema.
- [ ] Aggiornamento dei soli pacchetti PHP installati; le dipendenze richieste si leggono con una
  simulazione prima di confermare.
- [ ] `session.upload_progress.enabled=off` ed `expose_php=off` nel `php.ini` di Apache, modificato
  dopo l'aggiornamento dei pacchetti e confrontato con la copia.
- [ ] Riavvio di Apache con controllo che non restino processi con la libreria PHP precedente.
- [ ] Verifica della correzione: `dpkg -l`, changelog del pacchetto e versione caricata da Apache,
  controllata con una pagina di prova interrogata solo da localhost e poi cancellata. Mai solo
  `php -v`.
- [ ] Verifica del monitoraggio: login nel frontend fatto dal dev attraverso un tunnel SSH
  (`ssh -L 8080:localhost:80 zabbix`, poi `http://localhost:8080/zabbix/`), perché il frontend è
  solo in HTTP e le credenziali non devono passare in chiaro; dashboard con «Zabbix server is
  running: Yes» e grafici che si aggiornano; `zabbix-server` e `zabbix-agent` attivi; nessun nuovo
  errore PHP rispetto al livello di partenza.
- [ ] Verifica di compromissione sulle evidenze copiate; se emerge qualcosa ci si ferma prima di
  andare avanti.
- [ ] Esito nel ticket oc:8863; in questo repo solo ciò che serve a chi lavorerà sul server domani.
- [ ] Repo legato al server (scrum del 26/10): `README.md` che dice a quale server corrisponde;
  `CLAUDE.md` con le regole del repo pubblico e del `git pull` sul server dopo ogni merge; clone
  in `/root/devops` via HTTPS sul branch `main`, dopo il merge.

## Rischi
- **Liste apt mai aggiornate per arm64:** una volta corrette le sorgenti, il server vede tutti gli
  aggiornamenti accumulati, e con gli aggiornamenti automatici attivi li installerebbe da solo la
  notte dopo. Per questo gli aggiornamenti automatici vengono sospesi prima e l'installazione resta
  mirata ai pacchetti PHP. Il prezzo è che il server resta senza aggiornamenti automatici finché
  qualcuno non fa l'aggiornamento completo: va scritto nel ticket.
- **Riavvio di Apache:** breve interruzione del solo frontend; la raccolta dati di `zabbix-server`
  non dipende da Apache.
- **Ritorno indietro:** i pacchetti PHP precedenti non sono più scaricabili e sarebbero comunque
  vulnerabili; il punto di ritorno è lo snapshot.

## Out of scope
- Aggiornamento completo del sistema.
- Firewall e porte esposte del server.
- Aggiornamento di Zabbix.
- Documentazione di PostHog, che sta sullo stesso server.
- Rotazione dei segreti, che il tag php2shell fa a fine giro.

## Moduli toccati
Server `zabbix` (nessun codice applicativo nostro):
- sorgenti apt;
- pacchetti `php8.1-*` e `libapache2-mod-php8.1`;
- `/etc/php/8.1/apache2/php.ini`;
- servizio `apache2` (riavvio);
- clone del repo in `/root/devops`.

Repo `devops`: questa cartella `docs/features/8863-…/`, `README.md`, `CLAUDE.md`.
