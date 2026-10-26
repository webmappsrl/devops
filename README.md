# devops

Documentazione del server `zabbix` del progetto Hetzner **devops** di Webmapp, su cui girano
servizi che non hanno un repository nostro:

- **Zabbix** — monitoraggio dell'infrastruttura (server, frontend web PHP su Apache, agent,
  database PostgreSQL), installato dai pacchetti ufficiali Zabbix;
- **PostHog** — analytics.

Qui si registrano le scelte fatte su questi servizi, legate ai ticket Orchestrator che le hanno
prodotte, in `docs/features/<ID>-<titolo>/`.

Il repository è clonato sul server in `/root/devops`.

Questo repository è **pubblico**: indirizzi, versioni vulnerabili, porte esposte ed esiti di
verifiche di sicurezza non vanno scritti qui, ma nei ticket.
