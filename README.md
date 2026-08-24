# msg

Chat **peer-to-peer** inspirado en IRC, sin servidor central. Cada proceso es un nodo de una malla TCP: canales `#…`, nicks, gossip firmado (Ed25519), cifrado opcional de canal (invite → ChaCha20-Poly1305) y DMs sealed-box (X25519).

Escrito en [raylang](https://github.com/roberto-ayala/raylang). Versión **0.1** — MVP de malla listo (fases 0–8 en [`docs/PLAN.md`](docs/PLAN.md)). Wire: [`docs/PROTOCOL.md`](docs/PROTOCOL.md).

## Estado actual

Implementado y usable en LAN / IP alcanzable:

| Pieza | Qué hace |
|-------|----------|
| Identidad | Seed Ed25519 + X25519 derivada (HKDF); nick en `~/.config/msg/` |
| Mesh | `listen` + dial a bootstraps; Hello firmado (v2 con X25519); Ping/Pong |
| Gossip | Flood de envelopes firmados, dedup por `msg_id`, hops |
| Canales | `/join` público (solo firma) o cifrado con invite OOB |
| DMs | `/msg NICK` sealed box; requiere haber visto el nick (Hello/Presence) |
| Descubrimiento | PeerExchange + `--advertise`; peers exitosos en `config/peers` |
| UI | REPL por líneas (`>`), no TUI todavía |

Fuera de v1: NAT traversal / relay, discovery UDP LAN, historial persistente, TUI.

## Uso rápido

```sh
# Cada peer necesita su propio directorio de identidad (si no, comparten la misma clave
# y el otro lado descarta el chat como “eco propio”).
MSG_HOME=/tmp/msg-alice ray run src/main.ray --nick alice --listen 127.0.0.1:7700 --advertise 127.0.0.1
MSG_HOME=/tmp/msg-bob   ray run src/main.ray --nick bob   --listen 127.0.0.1:7701 --advertise 127.0.0.1 --peer 127.0.0.1:7700
```

En **ambos**:

```text
/join #lab
hola malla
```

Comprueba `peer up: …` y `/peers` antes de escribir. Sin `/join` en el receptor, el mensaje no se muestra (sí se reenvía en overlay).

Canal cifrado:

```text
/invite #secret
# comparte el invite OOB, el otro hace:
/join #secret <invite>
```

DM (tras ver el nick en la malla):

```text
/msg bob hola privado
```

## Comandos

| Comando | Efecto |
|---------|--------|
| `/nick NAME` | Cambia el alias y anuncia presencia |
| `/join #chan [invite]` | Entra (sin invite = público firmado) |
| `/part [#chan]` | Sale |
| `/peers` | Vecinos directos |
| `/who [#chan]` | Vista local de nicks |
| `/invite #chan` | Crea invite + entra cifrado |
| `/msg NICK text` | DM cifrado (X25519 sealed box) |
| `/help` | Ayuda |
| `/quit` | Sale |
| texto sin `/` | Mensaje al canal activo |

## Identidad y discovery

Por defecto en `~/.config/msg/` (`identity.seed`, `nick`, `peers`). Override: `--config DIR` o `MSG_HOME`.

| Flag | Rol |
|------|-----|
| `--listen HOST:PORT` | Bind (default `0.0.0.0:7700`) |
| `--peer HOST:PORT` | Bootstrap (repetible; se fusiona con `peers`) |
| `--nick NAME` | Alias (persistido) |
| `--advertise HOST` | Host publicado en PeerExchange (necesario para que terceros te redescubran) |

Sin `--advertise`, un peer que solo acepta conexiones no puede publicarse a la malla (el aceptante no conoce el host del inbound).

## Layout

```text
src/
  main.ray       CLI
  identity.ray   seed / nick / X25519
  framing.ray    frames u32 BE
  protocol.ray   Hello + Envelope
  crypto_chan.ray / crypto_dm.ray
  mesh.ray       hub, dial/accept, gossip delivery
  gossip.ray     dedup LRU
  channels.ray   join/part + clave
  store.ray      peers persistidos
  commands.ray   parser IRC-like
  ui.ray         prompt + líneas
```

## Tests

```sh
ray test src/commands.ray
ray test src/crypto_dm.ray
# (y el resto de módulos con @test)
```
