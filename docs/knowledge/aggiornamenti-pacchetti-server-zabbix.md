# Aggiornamenti dei pacchetti sul server zabbix

## Come funziona oggi
- I pacchetti Ubuntu arrivano da `mirror.hetzner.com/ubuntu-ports/packages` e
  `mirror.hetzner.com/ubuntu-ports/security` (il server è arm64: vedi la trappola nel `CLAUDE.md`);
  quelli di Zabbix dal repository ufficiale `repo.zabbix.com` per arm64.
- Il PHP del frontend è quello di sistema (`mod_php`, pacchetti `php8.1-*`): le correzioni di
  sicurezza arrivano come nuove revisioni Ubuntu dal canale `jammy-security`.
- Lo stato degli aggiornamenti automatici e i motivi sono nel ticket oc:8863.
- Un aggiornamento mirato si fa così: simulazione con `apt-get -s install --only-upgrade <pacchetti>`
  per leggere cosa cambia, poi installazione con `NEEDRESTART_MODE=l` (nessun riavvio automatico di
  servizi) e `-o Dpkg::Options::=--force-confold` (configurazioni esistenti conservate).
- La versione realmente in uso da Apache si verifica via web, da localhost, e nel changelog del
  pacchetto (`/usr/share/doc/libapache2-mod-php8.1/changelog.Debian.gz`).

## Perché così
- **Aggiornamento mirato e non completo** (oc:8863): chiudere una vulnerabilità senza toccare
  PostgreSQL e Zabbix; l'aggiornamento completo va fatto a parte, con snapshot fresco e qualcuno
  che guarda.
- **Lo snapshot è l'unico punto di ritorno** (oc:8863, regola in cima al `CLAUDE.md`): le revisioni
  precedenti dei pacchetti non restano scaricabili dai mirror.

## Come ci siamo arrivati
- **Nessun punto di ritorno, solo copia a mano delle evidenze** (oc:8863, superata lo stesso
  giorno): proposta quando lo snapshot era bloccato dal limite del progetto; superata dalla
  decisione di aumentare il limite di uno (scrum del 26/10).
