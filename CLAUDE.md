# devops

Documentazione del server `zabbix` del progetto Hetzner devops (Zabbix e PostHog). Nessun codice:
solo documentazione. Accesso al server: alias ssh `zabbix`, utente root.

## Prima di tutto

- **Il repo è pubblico.** Mai scrivere qui IP, versioni vulnerabili, porte esposte, stato del
  firewall, esiti di verifiche di compromissione, chiavi o credenziali: vanno nel ticket
  Orchestrator. Anche i messaggi di commit restano nella storia pubblica.
- **I comandi che scrivono sul server solo con l'ok esplicito del dev**, e prima di un intervento lo
  snapshot Hetzner.
- **Dopo ogni merge su `main`**, il dev che ha modificato la documentazione aggiorna il clone sul
  server: `ssh zabbix git -C /root/devops pull --ff-only`. Un agente lo esegue solo con il suo ok.

## Regole del repo

- Documentazione in italiano.
- Overview, piano e note di ogni lavoro in `docs/features/<ID>-<titolo-in-kebab-case>/`.

## Conoscenza

| Argomento | Cosa copre | Pagina |
|---|---|---|
| Aggiornamenti dei pacchetti | Da dove arrivano i pacchetti, come si fa un aggiornamento mirato e come si verifica | [docs/knowledge/aggiornamenti-pacchetti-server-zabbix.md](docs/knowledge/aggiornamenti-pacchetti-server-zabbix.md) |

## Trappole

- Il server è arm64: le sorgenti apt devono puntare a `mirror.hetzner.com/ubuntu-ports/…`, non a
  `…/ubuntu/…`, che per arm64 risponde 404 e lascia le liste ferme senza errori evidenti (oc:8863).
- `unattended-upgrade --dry-run` lanciato a mano ignora `APT::Periodic::Unattended-Upgrade`: per
  sapere se gli aggiornamenti automatici sono attivi si guarda `apt-config dump` (oc:8863).
