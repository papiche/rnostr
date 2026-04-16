# CLAUDE.md — rnostr

Relay NOSTR haute-performance écrit en Rust, basé sur LMDB.
Alternative à strfry pour les stations UPlanet nécessitant plus de performance.
License: MIT/Apache. Version: voir Cargo.toml.

## Concept

rnostr est un relay NOSTR haute-performance inspiré de strfry :
- Stockage **LMDB** (base clé-valeur ultra-rapide)
- **Hot reload** : rechargement de config sans redémarrage
- **Extensible** : système d'extensions (Prometheus, Auth NIP-42, rate limiter)
- Architecture **library** : `relay/` est un crate réutilisable pour créer des relays custom

## Structure

```
rnostr/
├── src/                  ← Binaire rnostr principal
├── relay/                ← Crate library (relay NOSTR générique)
├── extensions/           ← Extensions (metrics, auth, rate_limiter, count, search)
├── kv/                   ← Wrapper LMDB
├── db/                   ← Gestion base de données NOSTR
├── Cargo.toml            ← Dépendances workspace
├── rnostr.example.toml   ← Configuration de référence
└── docker-compose.yml    ← Déploiement Docker
```

## NIPs supportés

NIP-01, 02, 04, 09, 11, 12, 15, 16, 20, 22, 25, 26, 28, 33, 40, 42, 45, 50, 70

## Ports

| Port | Usage |
|------|-------|
| **7777** | WebSocket NOSTR public (via NPM → `relay.DOMAIN`) |
| **8888** | Port interne (metrics Prometheus / admin) — localhost uniquement, bloqué par UFW |

## Commandes

```bash
cd rnostr
cargo build --release
./target/release/rnostr relay -c ./rnostr.example.toml --watch   # Hot reload config

# Docker
docker-compose up -d

# Créer un relay custom (voir relay/README.md)
```

## Configuration (rnostr.example.toml)

```toml
[information]
name = "rnostr"
description = "..."

[data]
path = "./data"              # Chemin LMDB
db_query_timeout = "100ms"  # Timeout requêtes

[network]
host = "0.0.0.0"
port = 7777   # Port NOSTR relay UPlanet

[limitation]
max_message_length = 524288  # 512K
max_subscriptions = 20
max_filters = 10
```

## Extensions

| Extension | Description | NIP |
|-----------|-------------|-----|
| `metrics` | Exposition Prometheus | — |
| `auth` | Whitelist/blacklist IP + pubkey | NIP-42 |
| `rate_limiter` | Limitation de débit | — |
| `count` | Comptage résultats (expérimental) | NIP-45 |
| `search` | Recherche par mots-clés (expérimental) | NIP-50 |

## Différences avec strfry (NIP-101)

| Aspect | rnostr | strfry (NIP-101) |
|--------|--------|------------------|
| Langage | Rust | C++ |
| Filtres write-policy | Extensions Rust | Scripts Bash par kind |
| Intégration MULTIPASS | Non (standard) | Oui (UPlanet-specific) |
| Hot reload | Oui | Non |
| Extensibilité | Crate library | Plugin externe |
| Usage UPlanet | Expérimental/alternatif | Production (7777) |

## Utilisation dans UPlanet

rnostr est la **migration planifiée** pour remplacer strfry sur le port **7777**.

**Situation actuelle :**
- strfry est le relay de **production** (port 7777)
- strfry binary est aussi utilisé pour les opérations DB locales (`strfry scan`, `strfry import`, `strfry delete`) dans de nombreux scripts Astroport.ONE
- `backfill_constellation.sh` cherche explicitement `x_strfry.sh` (généré par DRAGON) — ce nom est figé

**Migration vers rnostr :**
Quand rnostr prendra le port 7777, plusieurs points de compatibilité sont à traiter :
1. Le slug DRAGON reste `"strfry"` pour que `backfill_constellation.sh` trouve `x_strfry.sh`
2. Les scripts utilisant `~/.zen/strfry/strfry scan|import|delete` devront migrer vers les outils rnostr
3. `amisOfAmis.txt` et `blacklist.txt` dans `~/.zen/strfry/` : gérer via extensions Auth de rnostr

## Développement d'extensions custom

Voir `relay/README.md` — le crate `nostr-relay` expose un mécanisme d'extension
pour intercepter les messages utilisateurs et implémenter un traitement custom.
