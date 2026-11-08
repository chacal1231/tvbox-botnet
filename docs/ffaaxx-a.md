# `.ffaaxx` y `.a` — caso A

[← README](../README.md)

> 🟩 Estático / 🟦 Observado · ⬜ informe original. Ninguna de estas muestras se revalidó en la ampliación del 2026-11-08: no estaban disponibles `.ffaaxx`, `pid259.bin` ni `onu-evidencia.tar`.

## `.ffaaxx` — resolución vía Ethereum

| Bytes | MD5 | SHA-256 |
| --- | --- | --- |
| 652524 | `3158380a9338e21e9db3dea68671dd59` | `0afa465224ae07e025b5bea1ba5b29618c3309754a9d471ac85958600ac66b5f` |

- **Persistencia:** se ejecuta desde `/var/config/o/.ffaaxx` con el argumento `vodkat` (PID 259).
- **Peticiones HTTP:** cookies, user-agents de navegador y cabeceras variables.
- **Cliente TLS** y **DNS sobre HTTPS**.
- **Resolución ENS:** envía JSON-RPC `eth_call` a proveedores públicos (`public-eth.nownodes.io`, `eth.drpc.org`) para resolver **`2busydrinkingcodeine.eth`**.
- **Socket UNIX abstracto:** `@drunkasfk`.
- **Conexiones observadas:** `87.58.159.3:443` y `43.157.198.137:443`. Con la evidencia disponible no se puede distinguir si son nodos RPC, resolvedores o C2.

`eth_call` es una consulta de solo lectura a un contrato: no publica transacciones. **Su uso no implica minería, transferencias ni robo de criptomonedas.**

### Pendiente

- [ ] Contrato concreto, función consultada y formato de la respuesta.
- [ ] Conversión de la respuesta a dirección C2.
- [ ] Algoritmo de selección y rotación.
- [ ] Localizar evidencia histórica del registro ENS ya capturada. Una consulta a un explorador es una actividad de red diferente, no realizada en esta revisión.

No se atribuyen a `.ffaaxx` ni el XOR `6b 9e 3f 2a` ni la selección `R mod N` comprobados en `.liso`.

## `.a` — generador de tráfico

| Bytes | MD5 | SHA-256 |
| --- | --- | --- |
| 310808 | `965c86dc0c3db91de5b3a2afcfe38742` | `4707d314f078b84b217e0cc9e6a02283f262c71dbafe5389ba9589e03fcb4c39` |

- Se ejecuta desde `/var/tmp/.a`, un archivo ya **eliminado**. El binario solo se pudo recuperar desde `/proc/3437/exe`.
- Conecta a `143.244.144.70:443` y escucha en `0.0.0.0:42363/TCP`.
- Contiene plantillas HTTP orientadas a agotar recursos del destino:
  - parámetros aleatorios;
  - `X-Forwarded-For`;
  - `HTTP CONNECT`;
  - `Content-Length: 2147483647`.
- Contiene cadenas de SSH y WireGuard. 🟨 Podría usarlas para imitar esos protocolos; su presencia sola no lo demuestra.

### Pendiente

- [ ] Función del puerto TCP 42363.
- [ ] Relación con `u`. Comparten destino, pero eso no demuestra identidad. Las cadenas de SSH/WireGuard son indicios débiles; comparar código y protocolo antes de relacionar variantes. No se trasladan los métodos de `u` a `.a`.
