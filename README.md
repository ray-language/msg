# msg

Chat **peer-to-peer** inspirado en IRC, sin servidor central. Cada proceso es un nodo de una malla TCP: canales `#…`, nicks, gossip firmado (Ed25519) y cifrado opcional de canal (invite → ChaCha20-Poly1305).

Escrito en [raylang](https://github.com/roberto-ayala/raylang). Diseño: [`docs/PLAN.md`](docs/PLAN.md), wire: [`docs/PROTOCOL.md`](docs/PROTOCOL.md).

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

Comprueba `peer up: …` y `/peers` antes de escribir. Sin `/join` en el receptor, el mensaje no se muestra.

Canal cifrado:

```text
/invite #secret
# comparte el invite OOB, el otro hace:
/join #secret <invite>
```

## Comandos

| Comando | Efecto |
|---------|--------|
| `/nick NAME` | Cambia el alias |
| `/join #chan [invite]` | Entra (sin invite = público firmado) |
| `/part [#chan]` | Sale |
| `/peers` | Vecinos directos |
| `/who` | Vista local de nicks |
| `/invite #chan` | Crea invite + entra cifrado |
| `/msg NICK text` | DM cifrado (X25519 sealed box) |
| `/help` | Ayuda |
| `/quit` | Sale |

## Identidad

Por defecto en `~/.config/msg/` (`identity.seed`, `nick`, `peers`). Override: `--config DIR` o `MSG_HOME`.  
`--advertise HOST` publica tu dirección en PeerExchange (necesario para que terceros te redescubran).

## Límites v1

- Solo P2P directo (LAN / IP alcanzable); sin NAT traversal ni relay.
- REPL por líneas (aún no TUI completa).
- DMs requieren haber visto al nick (Hello/Presence) para conocer su X25519.
