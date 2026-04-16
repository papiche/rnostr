# Migration strfry → rnostr

Plan de migration pour remplacer strfry (C++) par rnostr (Rust) sur le port 7777
de l'écosystème UPlanet / Astroport.ONE.

**État actuel :** strfry est le relay de production (port 7777).
**Objectif :** rnostr remplace strfry sur le même port, sans modifier les scripts Astroport.ONE ni NIP-101.

---

## Inventaire des usages de strfry

### A. DAEMON relay (port 7777)

Lancé par `NIP-101/start_strfry-relay.sh` :
```bash
cd ~/.zen/strfry && ./strfry relay &
```
Connexions WebSocket depuis `UPassport/services/nostr.py` et `backfill_constellation.sh`
sur `ws://127.0.0.1:7777`.

### B. CLI binaire — `~/.zen/strfry/strfry`

~50 scripts Astroport.ONE appellent le binaire strfry directement depuis `~/.zen/strfry/` :

```bash
# SCAN — lecture DB locale (filtre NIP-01 JSON)
./strfry scan '{"kinds": [0], "authors": ["$hex"], "limit": 1}'
./strfry scan --count '{}'   # compte total événements

# IMPORT — insertion NDJSON dans la DB
echo "$event_json" | ./strfry import
echo "$event_json" | ./strfry import --no-verify  # skip vérif signature

# DELETE — suppression par filtre
echo '{"ids": ["$id"]}' | ./strfry delete
./strfry delete --filter='{"ids": ["$id"]}'
```

Fichiers utilisant le CLI (liste non exhaustive) :
- `RUNTIME/NOSTR.UMAP.refresh.sh` — scan profils, activité, likes, contrats
- `RUNTIME/NODE.refresh.sh` — amisOfAmis.txt maintenance
- `RUNTIME/ECONOMY.broadcast.sh` — import kind 30850 (fallback)
- `NIP-101/backfill_constellation.sh` — scan count, import N²
- `tools/nostr_get_events.sh` — CLI générique multifiltre
- `tools/nostr_RESTORE_TW.sh` — import events restauration
- `tools/dashboard.TUBE.manager.sh` — delete events

### C. Write policy plugin (NIP-101)

strfry pipe chaque événement entrant vers un script bash :
```
event JSON → stdin → all_but_blacklist.sh → stdout → {"id":"…","action":"accept|reject|shadowReject"}
```

**Entry points :**
- `NIP-101/relay.writePolicy.plugin/all_but_blacklist.sh` — mode par défaut (tout accepter sauf blacklist)
- `NIP-101/relay.writePolicy.plugin/process.sh` — mode strict (MULTIPASS holders seulement)

**13 filtres par kind** dans `NIP-101/relay.writePolicy.plugin/filter/` :

| Kind | Fichier | Logique |
|------|---------|---------|
| 0 | `0.sh` | Log & accept (profils) |
| 1 | `1.sh` | Classify, rate-limit visiteurs (3/24h), detect #secret, cleanup TTL 48h, appelle `strfry scan` + `strfry delete` |
| 7 | `7.sh` | Valide paiements ẐEN, crowdfunding, votes assets |
| 21, 22 | `{21,22}.sh` | Accept (vidéos) |
| 1984 | `1984.sh` | Log signalements d'abus |
| 9735 | `9735.sh` | Traite ZAP receipts |
| 22242 | `22242.sh` | NIP-42 AUTH — crée marqueurs `~/.zen/game/nostr/$email/.nip42_auth_$pubkey` |
| 30023 | `30023.sh` | Accept articles long-form |
| 30078 | `30078.sh` | Accept app data |
| 30303 | `30303.sh` | Accept bons TrocZen |
| 30500 | `30500.sh` | Valide permit definitions (Oracle) |
| 30904 | `30904.sh` | Valide campagnes crowdfunding |

### D. Fichiers de données `~/.zen/strfry/`

| Fichier | Accès | Rôle | Mis à jour par |
|---------|-------|------|----------------|
| `strfry.conf` | RO | Config relay (hardware-tuned par setup.sh) | NIP-101/setup.sh |
| `strfry-db/` | RW | LMDB database (data.mdb, lock.mdb) | daemon |
| `amisOfAmis.txt` | RW | Whitelist friends-of-friends | NODE.refresh, NOSTRCARD.refresh, UPLANET.refresh, NOSTR.UMAP.refresh |
| `blacklist.txt` | RW | Pubkeys rejetées | UPlanet_IA_Responder.sh |
| `.pid` | RW | PID daemon | start/stop scripts |
| `constellation-backfill.{log,pid,lock}` | RW | État backfill N² | backfill_constellation.sh |

### E. Backfill constellation (NIP-101)

`backfill_constellation.sh` synchronise les événements entre relays via :
1. Tunnel IPFS P2P → port 9999 (slug `"strfry"`, généré par DRAGON_p2p_ssh.sh)
2. WebSocket REQ sur ce tunnel
3. Import batch : `./strfry import [--no-verify] < events.ndjson`

> **Note :** `strfry router` (negentropy) n'est **pas utilisé** en production —
> la synchro passe exclusivement par le tunnel IPFS P2P + WebSocket REQ.
> `strfry router` n'est donc **pas à porter**.

---

## Ce que rnostr fournit déjà

| Fonctionnalité | Commande rnostr | Status |
|---|---|---|
| Daemon relay WebSocket | `rnostr relay -c config.toml` | ✓ |
| Import NDJSON | `rnostr import <path> -` | ✓ |
| Export/scan par filtre | `rnostr export <path> -f 'filter'` | ✓ (syntaxe différente) |
| Delete par filtre | `rnostr delete <path> -f 'filter'` | ✓ (syntaxe différente) |
| NIP-42 Auth (whitelist/blacklist) | Extension `auth` | ✓ (statique TOML) |
| Prometheus metrics | Extension `metrics` | ✓ |
| Rate limiting | Extension `rate_limiter` | ✓ |
| NIP-45 count | Extension `count` | ✓ |
| NIP-50 search | Extension `search` | ✓ |
| Hot reload config | `--watch` | ✓ |
| LMDB storage | `nostr-db` crate | ✓ |
| NIPs 01,02,04,09,11,15,16,20,22,25,26,28,33,40,42,45,50,70 | — | ✓ |

---

## Gaps à combler

### GAP 1 — Write policy bash bridge (BLOQUANT)

**Problème :** rnostr a un système d'extensions Rust mais **aucun mécanisme d'appel de scripts bash**.
Les 13 filtres NIP-101 ne peuvent pas être branchés directement.

**Solution retenue : extension Rust "bash-policy"** qui spawne le script configuré :

```rust
// extensions/src/bash_policy.rs
use std::process::{Command, Stdio};
use std::io::Write;

impl Extension for BashPolicy {
    fn message(&self, msg: ClientMessage, session: &mut Session, _ctx: &mut ...) 
        -> ExtensionMessageResult 
    {
        if let ClientMessage::Event(event) = &msg {
            let event_json = serde_json::to_string(event)?;
            let mut child = Command::new(&self.script_path)
                .stdin(Stdio::piped())
                .stdout(Stdio::piped())
                .spawn()?;
            child.stdin.as_mut().unwrap().write_all(event_json.as_bytes())?;
            let output = child.wait_with_output()?;
            // parser {"id":"…","action":"accept|reject|shadowReject"}
            // → accept : Continue(msg)
            // → reject/shadowReject : Stop(OutgoingMessage::notice("blocked"))
        }
        ExtensionMessageResult::Continue(msg)
    }
}
```

Config TOML :
```toml
[bash_policy]
enabled = true
script = "/home/$USER/.zen/workspace/NIP-101/relay.writePolicy.plugin/all_but_blacklist.sh"
timeout_ms = 500
```

Avantage : **zéro modification des scripts NIP-101**.
Inconvénient : overhead par événement (spawn bash). Acceptable pour le volume UPlanet.

**Durée estimée : 2-3 jours.**

---

### GAP 2 — Wrappers CLI de compatibilité (BLOQUANT)

Les scripts Astroport font `cd ~/.zen/strfry && ./strfry scan|import|delete`.
La syntaxe rnostr est différente et requiert le path DB explicitement.

**Différences de syntaxe :**

| Appel strfry (actuel) | Appel rnostr équivalent |
|---|---|
| `./strfry scan 'filter'` | `rnostr export ./events -f 'filter'` |
| `./strfry scan --count '{}'` | *(flag `--count` absent)* |
| `echo e \| ./strfry import` | `echo e \| rnostr import ./events -` |
| `./strfry import --no-verify` | *(flag `--no-verify` absent)* |
| `echo f \| ./strfry delete` | `rnostr delete ./events -f 'filter'` |

**Solution : wrapper shell `~/.zen/strfry/strfry`** qui intercept les appels et traduit :

```bash
#!/bin/bash
# ~/.zen/strfry/strfry — wrapper de compatibilité strfry → rnostr
DB_PATH="$(dirname "$0")/events"
CMD="$1"; shift

case "$CMD" in
    scan)
        FILTER="${1:-{}}"
        if [[ "$FILTER" == "--count" ]]; then
            rnostr export "$DB_PATH" -f "${2:-{}}" | wc -l
        else
            rnostr export "$DB_PATH" -f "$FILTER"
        fi
        ;;
    import)
        # pass --no-verify if present (needs rnostr support, see GAP 2b)
        rnostr import "$DB_PATH" - "$@"
        ;;
    delete)
        FILTER="{}"
        while [[ $# -gt 0 ]]; do
            case "$1" in
                --filter=*) FILTER="${1#--filter=}" ;;
                *) FILTER="$1" ;;
            esac
            shift
        done
        rnostr delete "$DB_PATH" -f "$FILTER"
        ;;
    relay)
        rnostr relay -c "$(dirname "$0")/rnostr.conf" "$@"
        ;;
esac
```

**Modifications Rust nécessaires en plus du wrapper :**
- **`--count` sur `rnostr export`** : ajouter flag booléen → affiche count au lieu des events (trivial)
- **`--no-verify` sur `rnostr import`** : skip vérification signature (`event.verify()`) (trivial)

**Durée estimée : 1 jour** (wrapper + 2 flags Rust).

---

### GAP 3 — amisOfAmis.txt / blacklist.txt dynamiques (MOYEN)

rnostr Auth extension supporte `pubkey_whitelist`/`pubkey_blacklist` dans le TOML,
mais ces listes sont **statiques** (hot reload du fichier TOML complet).

Les fichiers `amisOfAmis.txt` et `blacklist.txt` sont mis à jour en continu par
5 scripts différents (NODE.refresh, NOSTRCARD.refresh, UPLANET.refresh…).

**Si GAP 1 est résolu avec le bridge bash :** ce point est automatiquement résolu
car les scripts bash lisent eux-mêmes ces fichiers. **Rien à faire.**

**Si migration native Rust souhaitée :** étendre l'extension `auth` pour lire des
fichiers texte (un pubkey par ligne) avec inotify/polling :
```toml
[auth.event.permission]
pubkey_whitelist_file = "/home/$USER/.zen/strfry/amisOfAmis.txt"
pubkey_blacklist_file  = "/home/$USER/.zen/strfry/blacklist.txt"
reload_interval_secs   = 60
```
**Durée estimée : 1-2 jours** (si migration native, sinon 0).

---

### GAP 4 — NIP-42 marqueurs fichiers (MOYEN)

`filter/22242.sh` crée des fichiers marqueurs lors d'une AUTH réussie :
```bash
~/.zen/game/nostr/$email/.nip42_auth_$pubkey
```
Ces fichiers sont consultés par d'autres scripts pour confirmer l'authenticité.

rnostr gère NIP-42 nativement mais n'expose pas l'état d'auth au système de fichiers.

**Solution :** extension Rust légère qui, lors d'un AUTH réussi, cherche l'email
associé à la pubkey dans `~/.zen/game/nostr/*/HEX` et crée le fichier marqueur.

**Si GAP 1 est résolu avec le bridge bash :** le script `22242.sh` existant gère
déjà ce comportement. **Rien à faire.**

**Durée estimée : 1 jour** (si migration native).

---

### GAP 5 — Path DB et migration des données (PRÉALABLE)

strfry DB : `~/.zen/strfry/strfry-db/data.mdb`
rnostr DB : configurable via `[data] path = "…"` → crée un sous-répertoire `events/`

**Configuration rnostr.conf :**
```toml
[data]
path = "/home/$USER/.zen/strfry"
# rnostr crée ~/.zen/strfry/events/ automatiquement
```

**Migration des données existantes :**
```bash
# 1. Exporter toute la DB strfry
cd ~/.zen/strfry && ./strfry export > /tmp/strfry_export.ndjson

# 2. Importer dans rnostr
rnostr import ~/.zen/strfry/events - < /tmp/strfry_export.ndjson

# 3. Vérifier le compte
./strfry scan --count '{}'              # avant
rnostr export ~/.zen/strfry/events | wc -l  # après
```

**Durée estimée : 1 heure.**

---

### GAP 6 — Slug IPFS tunnel DRAGON (COMPATIBILITÉ)

`DRAGON_p2p_ssh.sh` publie le relay NOSTR sous le slug `"strfry"` :
```bash
generate_p2p_service 7777 "strfry" "Nostr Relay" 9999
# → génère x_strfry.sh et canal IPFS /x/strfry-$IPFSNODEID
```

`backfill_constellation.sh` cherche **littéralement** `x_strfry.sh` (ligne 352).

**Ce slug NE DOIT PAS changer**, même après migration vers rnostr.
Le commentaire dans DRAGON l'explique déjà. **Rien à faire.**

---

## Plan d'exécution recommandé

```
Jour 0  : GAP 5 — Config path DB rnostr.conf, script de migration données
Jour 1  : GAP 2 — Wrapper shell strfry + flags --count et --no-verify dans rnostr
Jours 2-4 : GAP 1 — Extension Rust bash-policy (spawn + stdin/stdout + timeout)
Jour 5  : Tests d'intégration (backfill, filtres kind 1 et 7, NIP-42)
Basculement : remplacer service systemd strfry → rnostr (même port 7777)
```

GAP 3 et 4 sont **résolus automatiquement** si le bridge bash (GAP 1) est retenu.

**Effort total estimé : 5-7 jours** pour une migration fonctionnelle complète.

---

## Ce qui ne change pas

- Les scripts NIP-101 (`all_but_blacklist.sh`, `process.sh`, filtres `filter/*.sh`)
- `backfill_constellation.sh` (synchro via tunnel IPFS P2P + WebSocket REQ)
- `amisOfAmis.txt` et `blacklist.txt` (chemins, format, scripts qui les alimentent)
- Slug DRAGON `"strfry"` → `x_strfry.sh` (figé pour compatibilité backfill)
- Port 7777 (UFW, NPM, subdomain `relay.DOMAIN`)
- Port 9999 (port local du tunnel IPFS P2P pour backfill)
- `strfry router` — non utilisé, non à porter
