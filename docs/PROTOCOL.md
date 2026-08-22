# Protocolo msg v1

Contrato de wire para la malla P2P. Complementa [`PLAN.md`](PLAN.md).

## Transporte

TCP. Cada mensaje de aplicación es un frame length-prefixed:

```text
u32_be(length) ‖ payload
```

`length` = tamaño de `payload` (sin contar los 4 octetos).  
`MAX_FRAME` = 65536.

## Constantes

| Nombre | Valor |
|--------|-------|
| `MAGIC` | ASCII `MSG1` (4 B) |
| `VERSION` | `1` |
| `DEFAULT_PORT` | `7700` |
| `DEFAULT_HOPS` | `8` |
| `PUBKEY_LEN` | `32` |
| `SIG_LEN` | `64` |
| `NONCE_LEN` | `16` |
| `MSG_ID_LEN` | `32` |

## Hello (primer frame de cada enlace)

En claro, firmado. Ambos extremos envían un Hello tras el `accept`/`connect` (orden: **listener first**, luego dialer).

```text
magic        4   "MSG1"
version      1   u8 = 2
pubkey      32   Ed25519 public key
listen_port  2   u16 BE
nick_len     2   u16 BE
nick         N   UTF-8
x_pubkey    32   X25519 public key (long-term, HKDF from Ed25519 seed)
nonce       16   CSPRNG
sig         64   Ed25519(seed, signed_preimage)
```

`signed_preimage` = todo lo anterior **excepto** `sig`.

## Envelope (frames posteriores)

```text
kind         1   u8
hops         1   u8   (no entra en la firma; se decrementa al reenviar)
msg_id      32   sha256(origin_pub ‖ seq_be64 ‖ kind ‖ body)
origin_pub  32
seq          8   u64 BE (monótono por origin)
ts_ms        8   i64 BE (epoch ms; informativo)
body_len     4   u32 BE
body         B
sig         64   Ed25519(origin_seed, signed_preimage)
```

`signed_preimage` =

```text
kind ‖ msg_id ‖ origin_pub ‖ seq ‖ ts_ms ‖ body
```

Al reenviar se copia el envelope y solo se escribe `hops - 1` en el campo hops; `sig` no cambia.

### Kinds

| kind | Nombre | body |
|------|--------|------|
| `0x01` | Presence | `nick_len u16 ‖ nick ‖ status u8` (0=online, 1=away) |
| `0x02` | Chat | `chan_len u16 ‖ chan_utf8 ‖ flags u8 ‖ payload` |
| `0x03` | Part | `chan_len u16 ‖ chan_utf8` |
| `0x04` | PeerExchange | `count u16` + N × (`host_len u16 ‖ host ‖ port u16 ‖ pubkey 32`) |
| `0x05` | Ping | `token 8` (echo en Pong) |
| `0x06` | Pong | `token 8` |
| `0x07` | Dm | `dest_ed25519 32 ‖ eph_x25519 32 ‖ ciphertext` |

`flags` en Chat:

| bit | Significado |
|-----|-------------|
| `0x01` | `ENCRYPTED` — `payload` es ciphertext ChaCha20-Poly1305 |
| `0x00` | plaintext UTF-8 (canal público) |

## Chat cifrado (invite)

Invite (string OOB), hex lowercase de **32 bytes** = `salt_16 ‖ secret_16` (64 hex chars; UI puede agrupar con guiones).

```text
key = hmac_sha256(sha256(secret), salt ‖ "msg-chan-v1")
nonce = seq_u64_be ‖ 0x00×4
aad   = channel_utf8 ‖ origin_pub ‖ seq_u64_be
ciphertext = chacha20poly1305_seal(key, nonce, aad, plaintext_utf8)
```

Sin invite → `flags=0`, `payload` = texto UTF-8 en claro (sigue firmado a nivel envelope).

## DM cifrado (`/msg`)

Sealed box por mensaje (efímera X25519 + HKDF + ChaCha20-Poly1305):

1. `eph_sk = random_bytes(32)`, `eph_pk = x25519_public_key(eph_sk)`
2. `dh = x25519_shared_secret(eph_sk, dest_x25519_pub)`
3. `key‖nonce = hkdf_sha256(eph_pk‖dest_x_pk, dh, "msg-dm-v1", 44)` (32+12)
4. `aad = dest_ed25519 ‖ origin_ed25519 ‖ seq_be64`
5. Body: `dest_ed25519 ‖ eph_pk ‖ chacha20poly1305_seal(…)`

La identidad X25519 de largo plazo se deriva del seed Ed25519:
`hkdf_sha256("", seed, "msg-identity-x25519-v1", 32)`.

El envelope DM se gossipea; solo el destinatario (match de `dest_ed25519`) descifra y muestra.

## Gossip

1. Verificar `sig` y recalcular `msg_id`; mismatch → drop.
2. Si `msg_id` en cache LRU → drop.
3. Insertar en cache; entregar a UI si aplica (Chat de canal joined, etc.).
4. Si `hops > 0`, reenviar a todos los vecinos excepto el de entrada con `hops - 1`.

Todos los peers reenvían (overlay), estén o no en el canal.

## Presencia / who

Vista **local**: Presence recibidos + nicks del Hello de vecinos directos. No hay roster global.

## PeerExchange y descubrimiento

Tras cada `LinkUp`, el nodo envía al nuevo vecino un `PeerExchange` (hops=0) con:

- su propio `advertise:port` si `--advertise HOST` está activo;
- todos los vecinos con dirección dialable conocida (no `inbound:…`).

El receptor marca hints desconocidos y hace `TryDial`. Las direcciones exitosas se guardan en `config/peers` (una por línea) y se recargan al arrancar junto a `--peer`.

Sin `--advertise`, un peer que solo acepta conexiones no puede publicarse a terceros (el aceptante no conoce el host del inbound).

## Fuera de v1 en el wire

- Compresión, TLS de enlace, relay frames
- Resume / ACK de entrega
