# Audit de compatibilité Ubuntu 26.04 — init-vps

Date : 29/09/2026. Script audité : `init-vps.sh` de `main` (7380349), identique à
la release **v2026.09.15.4** hormis les lignes `SCRIPT_VERSION` / `INIT_VPS_REPO`.

Méthode :

- **lecture du code** : recensement de chaque appel concerné, avec son numéro de ligne ;
- **mesures sur un vrai serveur** : Hetzner CX23, image `ubuntu-26.04`, projet de test.
  Chaque appel de coreutils a été exécuté avec uutils **et** avec GNU (`gnu*`, installé
  en parallèle sur 26.04), avec les entrées réelles du script ;
- **installation de référence** de la release actuelle (étape 2), sur deux 26.04 et un
  témoin 24.04.

Légende : ✅ compatible · ⚠️ à vérifier ou à corriger · ❌ incompatible.

## Synthèse

| Sujet | Verdict | À faire |
|---|---|---|
| Coreutils Rust (uutils 0.8.0) | ✅ pour tous les appels du script | Aucun changement requis. Un bug **existant sur 24.04** est corrigé par uutils (voir `reboot_postpone`). |
| sudo-rs 0.2.13 | ⚠️ | `timestamp_type` **refusé** : l'étape 3.4 ne peut garder que `timestamp_timeout`. L'horodatage est par terminal. |
| Hypothèses « 24.04 » | ✅ | Toutes vérifiées au runtime. |
| Dépôt Docker `resolute` | ✅ | Publié. Le repli sur `noble` n'est pas utilisé. |
| fail2ban / OpenSSH / UFW | ✅ | OpenSSH 10 : liste `KexAlgorithms` sans post-quantique (non spécifique à 26.04). |
| Installation de référence (étape 2) | ✅ sur 26.04 | Code 0 sur les deux rôles. Trois défauts **communs à 24.04** sont à corriger à l'étape 3.5. |

**Verdict provisoire : rien ne bloque la 26.04 dans la release actuelle.** Le go définitif
attend l'étape 4 (redémarrage, timers, liaison Dokploy).

## 1. Coreutils en Rust (uutils)

Sur 26.04, `/usr/bin/date` → `../lib/cargo/bin/coreutils/date`, `date (uutils coreutils) 0.8.0`,
paquets `rust-coreutils 0.8.0-0ubuntu3` + `coreutils-from-uutils`. GNU reste installé
(`gnu-coreutils 9.7`, binaires préfixés `gnu`). `grep` (GNU 3.12), `sed` (GNU 4.9),
`awk` (gawk 5.3.2) et `find` (GNU findutils 4.10) ne sont **pas** remplacés : les
`grep -P`, `sed -E`/`sed -i` et le motif `0,/…/` ne sont pas concernés.

| Appel (lignes) | Entrée réelle | Résultat mesuré | Verdict |
|---|---|---|---|
| `date +%s`, `+%Y%m%d%H%M%S`, `'+%F %T'`, `-Iseconds` (191, 1598, 2132, 4132…) | — | identique | ✅ |
| `date -d "@epoch" '+%d/%m à %H:%M'` (1451, 2738, 2945, 4345) | époque | identique | ✅ |
| `date -d "$started" +%s` (2505) | `StartedAt` Docker, `2026-09-29T12:34:56.123456789Z` ; `0001-01-01T00:00:00Z` | identique | ✅ |
| `date -d "$started" '+%F %T'` (2601) | `ActiveEnterTimestamp` systemd, `Tue 2026-09-29 14:00:00 CEST` | identique | ✅ (voir note 1) |
| `date -d "$(date -d "+${d} day" +%F) ${t}"` (2956, `next_reboot_window`) | `2026-09-30 04:00` | identique, y compris au changement d'heure | ✅ |
| `date -d "$(date -d "@$when" '+%F %H:%M') +1 day"` (2990, `reboot_postpone`) | `2026-09-29 04:00 +1 day` | **uutils : 04:00 — GNU : 05:00 CEST** | ✅ sur 26.04, **❌ sur 24.04** (note 2) |
| `numfmt --to=iec --suffix=o` (2404) | 536870912, 1610612736, 0, `max` | `512Mo`, `1.5Go`, `0o` ; `max` → erreur (repli `\|\| echo`) comme GNU | ✅ |
| `sort -V` (1468, 1579, 3843, 4179) | `2026.09.15.9` / `.10`, `0.0.0-dev`, noyaux `7.0.0-30` / `7.0.0-4` / `6.17…` | identique | ✅ |
| `sort -u`, `sort -un` (767, 2275, 3391) | — | identique | ✅ |
| `stat -c %Y` (2735) | — | identique | ✅ |
| `mktemp`, `mktemp -d`, modèles `.vps-helper.XXXXXX` (1235, 1416, 1518, 1616, 2023, 2074, 2139, 3748, 4158) | — | nom conforme, modes 600 / 700 | ✅ |
| `df -P -x tmpfs -x devtmpfs -x overlay -x squashfs -x efivarfs` (2728), `df -h /` (1430) | — | mêmes colonnes et lignes | ✅ |
| `cut -d' ' -f1-3`, `cut -d: -f6` (1404, 1428, 1749…) | — | identique | ✅ |
| `tr -s '[:space:]'`, `tr -d '[:space:]'`, `tr '\n' ','`, `tr -d '0-9'` (481, 1306, 1442, 1569, 2846) | — | identique | ✅ |
| `tr -d '\000-\010\013\014\016-\037'` (3827, vps-notify) | contrôles + ESC | identique | ✅ |
| `install -d -m 700 -o -g`, `install -m 0755 -d` (689, 1796, 2906, 3433, 3653) | — | modes et propriétaires corrects | ✅ |
| `paste -sd' '` / `-sd','` (1253, 2701, 2940, 3239) | — | identique | ✅ |
| `comm -23 fichier <(…)` (3045) | substitution de processus | identique | ✅ |
| `sha256sum -c --status`, `sha256sum \| cut` (2088, 2846) | fichier bon / altéré | codes 0 / 1 identiques | ✅ |
| `cp -a` (194, 1702, 1802…) | mode, propriétaire, mtime | conservés | ✅ |
| `mv -f`, `chmod a+r`, `chmod -x glob`, `touch`, `dirname`, `uniq -c`, `wc -l`, `head`/`tail -n`, `env bash`, `id -u` | — | identique | ✅ |
| `readlink` | — | aucun usage dans le script | — |

**Note 1 — fuseau écrit dans la date.** uutils **ignore** une abréviation `UTC`/`GMT`
quand le `TZ` du processus est différent : `TZ=Europe/Paris date -d "… 14:00:00 UTC"` donne
14:00 local, contre 16:00 avec GNU. `CEST` et `CET` sont correctement lus. Sans effet
ici : `systemctl show` écrit l'horodatage dans le fuseau du système, celui-là même que
lit `date`. À garder en tête pour toute future date en UTC explicite : préférer
`@epoch` ou le format ISO avec décalage (`+02:00`), qui est lu correctement.

**Note 2 — bug existant de `reboot_postpone`, sur 24.04.** GNU lit `04:00 +1 day`
comme « 04:00 **en UTC+1**, plus un jour » : le `+1` est pris pour un décalage de fuseau
(manuel GNU coreutils, *Time of day items* : l'heure peut être suivie d'une correction de
fuseau). Résultat mesuré à Paris : report à **05:00 CEST** en été (04:00 CET en hiver,
par coïncidence) ; sur un serveur en UTC, report à 03:00. uutils donne 04:00. À corriger
en 3.5 sans dépendre de ce comportement, par exemple
`date -d "$(date -d "@$when" +%F) +1 day" +%F` suivi de l'heure : c'est la forme déjà
utilisée, correctement, par `next_reboot_window`.

## 2. sudo-rs

Sur 26.04 : `sudo` → alternative `/usr/lib/cargo/bin/sudo` (`sudo-rs 0.2.13-0ubuntu1.2`),
`visudo` → `visudo-rs 0.2.13`. Le sudo classique (`1.9.17p2`) est installé mais n'est pas
l'alternative active. `/etc/sudoers` contient `@includedir /etc/sudoers.d`.

Mesures (`visudo -cf` sur un fichier temporaire) :

| Contenu | `visudo -cf` |
|---|---|
| `dokploy ALL=(ALL) NOPASSWD:ALL` | ✅ `parsed OK`, code 0 |
| `Defaults:admin timestamp_timeout=30` | ✅ code 0 |
| `Defaults:admin timestamp_timeout=30, timestamp_type=global` | ❌ `unknown setting: 'timestamp_type'`, code 1 |
| `Defaults timestamp_type=global` | ❌ code 1 |
| erreur de syntaxe (`NOPASSWORD:`, texte quelconque) | ❌ code 1, ligne et colonne indiquées |

- `visudo -c` et `visudo -cf FICHIER` existent, avec les mêmes codes de retour que le
  visudo classique (`visudo -h` : `-c, --check`, `-f, --file`).
- À l'exécution, un fichier déposé avec un réglage inconnu **ne bloque pas** sudo-rs :
  l'erreur est affichée et la commande s'exécute. On ne s'en sert pas comme filet :
  la règle reste « aucun fichier installé sans `visudo -cf` réussi ».
- Un fichier de `sudoers.d` dont le nom contient un point est **ignoré** (mesuré). Un
  temporaire `.90-dokploy.XXXXXX` dans le même répertoire est donc sans effet avant son
  renommage.
- Un fichier en 0644 est accepté (mesuré) ; on installe quand même en 0440.
- **Horodatage** : `sudoers-rs(5)`, section *SUDOERS OPTIONS*, liste 16 réglages, dont
  `timestamp_timeout`, mais pas `timestamp_type`. La même page indique :
  « sudo-rs uses a separate record for each terminal, which means that a user's login
  sessions are authenticated separately ». Pour l'étape 3.4, on garde donc
  `timestamp_timeout=30` seul : « une seule saisie pour deux commandes » ne vaut que
  **dans le même terminal**. `timestamp_type=global` resterait valide sur 24.04
  (sudo 1.9.15) : écrire le fichier selon ce que `visudo -cf` accepte, et non selon la
  version d'Ubuntu.
- Avertissement pendant le dist-upgrade (les deux serveurs 26.04) :
  `update-alternatives: warning: forcing reinstallation of alternative /usr/lib/cargo/bin/sudo because link group sudo is broken`.
  C'est une réparation automatique par le paquet : sudo-rs reste actif après coup
  (`update-alternatives --query sudo` : `Status: auto`). Rien à faire.

## 3. Hypothèses « Ubuntu 24.04 » du code et de CLAUDE.md

| Hypothèse (emplacement) | 26.04 mesuré | Verdict |
|---|---|---|
| sshd activé par socket, port généré au `daemon-reload` (l. 794, `ssh_restart`) | `ssh.socket` enabled, `ssh.service` disabled, générateur `sshd-socket-generator` présent. Phases 1/2 réussies, `ss` voit le bon port. | ✅ |
| `/run/sshd` pour `sshd -t` (`test_sshd_config`) | 26.04 : tmpfiles `d /run/sshd 0755` (`openssh-server.conf`), toujours présent. 24.04 : seulement `RuntimeDirectory=sshd` de `ssh.service`. | ✅ 26.04 — **❌ 24.04**, voir § 6 |
| `pam_motd` avec `noupdate` (l. 1497) | `/etc/pam.d/sshd` : `pam_motd.so motd=/run/motd.dynamic` + `pam_motd.so noupdate`, comme sur 24.04. `run-parts --test` ne liste que `00-studiokyne`. | ✅ |
| fail2ban en nftables (l. 2615) | `jail.d/defaults-debian.conf` : `banaction = nftables`. `recidive` : `backend = auto` + `logpath` bien posés. | ✅ |
| needrestart en mode `a` avant l'étape 1 (`configure_needrestart`) | needrestart 3.11 ; `$nrconf{restart} = 'a'` écrit, aucun menu pendant apt. | ✅ |
| unattended-upgrades, `${distro_codename}-security` | 2.12 ; `check` : « Dernière exécution sans erreur ». | ✅ |
| sysctl par défaut (liste fermée de `99-network-perf.conf`) | `somaxconn 4096`, `fs.file-max` 2^63−1, `tcp_tw_reuse 2`, `kptr_restrict 1`, `dmesg_restrict 1`, `ptrace_scope 1`, `unprivileged_bpf_disabled 2`, `protected_regular 2`, soit les mêmes valeurs que sur 24.04. `tcp_rfc1337 0` par défaut. | ✅ liste inchangée |
| Écrasement de `log_martians` par UFW | `/etc/ufw/sysctl.conf` aligné (`all/log_martians=1`), runtime = 1. **Non encore vérifié après un redémarrage** (étape 4). | ⚠️ à confirmer |
| cgroup v2 + driver systemd (l. 2426) | `cgroup2fs`, `docker info` : `systemd 2`. | ✅ |
| iptables-nft, `DOCKER-USER` | iptables 1.8.11 (nft), Docker 29.8.1 « Firewall Backend: iptables ». 9 règles v4 + 7 v6 posées. | ✅ |
| cloud-init hotplug (exception Hetzner) | cloud-init 26.1, comme sur 24.04 : l'exception s'applique. Pas d'échec observé pendant l'installation. | ✅ |
| python3 pour vps-notify | 3.14.3. Envoi non testé (pas de webhook de test). | ⚠️ à tester à l'étape 4 |

Bruit de journal nouveau avec systemd 259 (26.04) :
`systemd-networkd: Foreign process 'dockerd' changed sysctl '/proc/sys/net/ipv6/conf/eth0/disable_ipv6' from '0' to '1'`,
à chaque création de conteneur. Il s'agit de l'`eth0` **interne** de chaque conteneur :
`nsenter … sysctl net.ipv6.conf.eth0.disable_ipv6` vaut 1 dans les trois conteneurs, alors
que l'`eth0` de l'hôte reste à 0, avec IPv6 fonctionnelle en entrée (réponse de Traefik
sur l'adresse v6) comme en sortie (`curl -6` → 200). **Sans effet, à ne pas « corriger ».**

## 4. Dépôt Docker

`https://download.docker.com/linux/ubuntu/dists/resolute/Release` → 200 ; paquets
`docker-ce 5:29.8.1-1~ubuntu.26.04~resolute` publiés. `ensure_docker` teste l'URL
`Release` du codename courant, la trouve et utilise `resolute` : le repli sur `noble`
n'est **pas** déclenché (`docker.list` : `… ubuntu resolute stable`). Le repli reste
sain comme filet (même ligne de dépôt, paquets `noble` installables), mais il n'est plus
nécessaire pour 26.04. Le commentaire d'`ensure_docker`, qui cite justement « 26.04
resolute » comme exemple de dépôt absent, est donc devenu faux.

## 5. Versions des paquets

| Paquet | 24.04 (témoin) | 26.04 | Changement notable pour nous |
|---|---|---|---|
| fail2ban | 1.0.2 | 1.1.0 | Python 3.14 : service actif, jails `sshd` + `recidive` chargées. Même `defaults-debian.conf`. |
| openssh-server | 9.6p1 | 10.2p1 | Socket + générateur : inchangé. `/run/sshd` fourni par tmpfiles (§ 3). OpenSSH 10 négocie par défaut `mlkem768x25519-sha256` (post-quantique) ; notre `KexAlgorithms` l'exclut (§ 6). |
| ufw | 0.36.2-6 | 0.36.2-9 | Aucun. |
| needrestart | 3.6 | 3.11 | Aucun pour `restart = 'a'`. |
| unattended-upgrades | 2.9.1 | 2.12 | Aucun observé. |
| systemd | 255 | 259 | Bruit networkd (§ 3). |
| iptables | 1.8.10 | 1.8.11 | Aucun. |
| docker-ce | 29.8.1 | 29.8.1 | Même version, dépôt natif. |
| sudo actif | sudo 1.9.15 | **sudo-rs 0.2.13** | § 2. |
| noyau | 6.8 | 7.0 | Aucun observé ; cgroup v2 identique. |
| cloud-init | 26.1 | 26.1 | Aucun. |

## 6. Étape 2 — installation de référence de la release actuelle

Serveurs du projet Hetzner de test : réseau `test-prive` (10.0.0.0/16, eu-central),
CX23 nbg1, clé `lumia-tests`.

| Serveur | Image | Rôle | Hostname | IP privée |
|---|---|---|---|---|
| `test-manager` | 26.04 | 1 | `vps-internal-nbg1-9` | 10.0.0.2 |
| `test-remote` | 26.04 | 2 | `vps-client-nbg1-9` | 10.0.0.3 |
| `test-ref2404` (témoin) | 24.04 | 1 | `vps-internal-nbg1-8` | 10.0.0.4 |

Déroulé : release v2026.09.15.4 téléchargée (somme SHA-256 vérifiée), **script non
modifié**, piloté par `expect` sur chaque serveur. Réponses : port 22, Europe/Paris, swap
par défaut (4 Go), pas de webhook, redémarrage automatique à 04:00 (manager) / 04:30
(remote). Le script s'arrête sur la question de verrouillage SSH jusqu'à ce que la
connexion `admin` par clé et `sudo` aient été testés depuis un autre poste.

### Résultats

| | test-manager (26.04) | test-remote (26.04) | témoin 24.04 |
|---|---|---|---|
| Code de sortie | 0 | 0 | **1** au premier passage, 0 à la relance |
| `vps-helper check` | 17 PASS, **1 FAIL**, 1 WARN | 15 PASS, 0 FAIL, 1 WARN | 17 PASS, **1 FAIL**, 1 WARN |
| FAIL | Port 3000/tcp publié sur toutes les interfaces | — | idem manager |
| WARN | Redémarrage requis (noyau 7.0.0-34) | idem | idem (noyau 6.8) |
| Port 3000 depuis Internet | **injoignable** | — | **injoignable** |

Avertissements propres à 26.04 dans les logs :

1. `update-alternatives: … link group sudo is broken` (§ 2) — bénin.
2. `Failed to get properties: Transport endpoint is not connected` /
   `Failed to connect to system scope bus via local transport`, émis par le trigger
   `systemd` pendant le dist-upgrade, pendant le redémarrage de D-Bus. Bénin :
   aucune unit en échec ensuite (`check` : PASS).
3. Bruit networkd sur `disable_ipv6` (§ 3) — bénin.

**Aucune erreur d'init-vps propre à 26.04.**

### Défauts révélés, communs à 24.04 et 26.04 (à traiter en 3.5)

1. **Panneau Dokploy injoignable après une installation neuve.** `step_ufw_base` ouvre
   3000/tcp dans UFW et le résumé affiche `http://IP:3000`. Mais
   `step_docker_user_firewall` a posé `DOCKER-USER` avant Dokploy : DROP sur eth0 pour
   tout sauf 80/443. Or le trafic vers un port publié ne passe pas par UFW (cf.
   CLAUDE.md) : la règle `allow 3000` est sans effet. Mesuré : `curl
   http://IP:3000` depuis Internet → timeout, la règle 8 (DROP) compte exactement les
   paquets de l'essai ; `curl 127.0.0.1:3000` sur le serveur → 307. Même résultat sur
   24.04. Le `check` en FAIL « exposé si DOCKER-USER ne le filtre pas » est donc un faux
   positif sur ce point, et l'information « Port 3000 : ouvert » est inexacte. Ce
   défaut touche aussi l'étape 4.4 (tunnel SSH) : `ssh -L 3000:127.0.0.1:3000` fonctionne
   (le 307 local le montre), c'est d'ailleurs aujourd'hui le seul moyen d'atteindre le
   panneau.
2. **Premier lancement sur 24.04 : `sshd -t` échoue** (`Missing privilege separation
   directory: /run/sshd`). Avec l'activation par socket, `/run/sshd` n'existe que tant
   que `ssh.service` tourne, et needrestart venait de redémarrer des services. Le script
   s'arrête proprement (ancienne configuration gardée) ; une relance passe. Correctif
   simple : `install -d -m 0755 /run/sshd` avant `sshd -t`. Sans effet sur 26.04, où
   tmpfiles crée le répertoire.
3. **`KexAlgorithms` sans post-quantique.** `99-hardening.conf` impose
   `curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512`. Tout
   client OpenSSH ≥ 10 affiche alors `connection is not using a post-quantum key exchange
   algorithm` (voir https://openssh.com/pq.html). Ajouter en tête
   `mlkem768x25519-sha256` (OpenSSH ≥ 9.9, donc 26.04) et
   `sntrup761x25519-sha512@openssh.com` (≥ 8.5, donc 24.04). La liste doit rester
   valide sur les deux versions : `sshd -t` le vérifiera.
4. **`reboot_postpone`** : décalage d'une heure sur 24.04 (§ 1, note 2).
5. **`vps-helper check` renvoie 0 même en cas de FAIL** : mineur. À décider (code 1 ?),
   sans casser `vps-check.timer`, dont l'`ExecStart` devrait alors porter un `-`.

### Reste à vérifier à l'étape 4

- `log_martians` et l'ensemble des sysctl **après redémarrage** (écrasement par UFW au boot).
- Cycle complet du redémarrage automatique, envoi vps-notify (python3 3.14).
- Liaison Dokploy : le manager installe **Dokploy v0.30.8**, postérieure à la v0.29.0
  qui accepte un utilisateur non-root.
