# `u` — canal de control, órdenes y métodos · downloader

[← README](../README.md)

> 🟩 Estático · caso B (ZX8102WT). Todo lo descrito aquí es capacidad reconstruida del código. No se capturó ninguna sesión de control: no hay evidencia de que estas órdenes se recibieran o ejecutaran en la ONU, ni de que se usaran contra víctimas.

| Muestra | Bytes | SHA-256 |
| --- | --- | --- |
| `u` (empaquetado) | 169704 | `7208f46f1f2f1f8204d91c39570ab54743535a8a4b5d00d1c335c42859e437c4` |

**Desempaquetado:** se descomprimieron los bloques LZMA sin ejecutar la muestra y se reconstruyeron los segmentos cargables. No se afirma haber restaurado todos los metadatos del ELF original. Las VA de este documento corresponden a esos segmentos reconstruidos (ver [metodología](metodologia.md)).

## Canal

- **Destino predeterminado:** `143.244.144.70:443/TCP`.
  - El puerto sale de descifrar la entrada 1 de la tabla (`01bb`). Leer esos bytes sin descifrar habría dado un puerto incorrecto.
  - Un argumento de inicio permite sustituir el destino por una IPv4 o IPv6 literal.
  - No se confirmó ninguna rotación por dominios ni por ENS.
- **Transporte:** TCP no bloqueante con `select`, comprobación de errores y reconexión.
- **Framing:** prefijo de longitud de 2 bytes en orden de red, seguido del cuerpo. Según el estado del bucle, el cuerpo llega con una etiqueta de 16 bytes que se verifica y descifra, o llega directamente. No se ha demostrado que un tercero pueda elegir la rama o saltarse la negociación.
- **Criptografía:** X25519 con una identidad pública incorporada, ChaCha20 y Poly1305 en un **formato propio**.
  - No es TLS/X.509; combina ChaCha20 y Poly1305, pero el formato autenticado difiere del AEAD estándar RFC 8439.
  - **El puerto 443 no implica HTTPS.**
  - La clave que descifra las constantes no permite descifrar sesiones capturadas.
- **Despacho:** el primer byte del cuerpo indica la orden.
  - Las cadenas `HTTPFLOOD`, `BOTKILLER` y las plantillas IRC no se identificaron como órdenes textuales aceptadas por este despachador.
  - Los registros cuyo primer byte es 15 reciben un tratamiento previo distinto de la orden 15; su significado sigue pendiente.

## Autenticación inicial, contadores y errores

> 🟩 Estático · caso B. [Informe ampliado](evidencia/autenticacion-y-contadores-u.txt), [instrucciones](evidencia/evidencia-autenticacion-contadores-u.txt) y [comprobaciones](evidencia/autenticacion-contadores-u.json).

### Comprobaciones de establecimiento

El llamador exige el éxito de `0x1c6e0` antes de enviar identificación. Las rutas revisadas abortan ante fallo de transporte, de etiqueta, de longitud recuperada o de comparación de los 32 bytes de identidad esperada. La comparación acumula XOR/OR sobre el bloque. No se observó un fallback desde negociación fallida a despacho sin cifrar.

**Límite de autenticación:** el bloque inicial se verifica bajo el secreto efímero. Comparar una identidad pública no prueba por sí solo posesión de la privada estática. El segundo secreto participa en la clave final, pero la negociación retorna antes del primer mensaje entrante protegido con ella. No se certifica autenticación mutua formal ni resistencia completa a intermediarios.

No se encontró rechazo explícito del resultado compartido todo-cero entre `0x1bc00` y la derivación revisada. Esa función tiene un retorno normal que comunica cero; comprobar ese código de retorno no equivale a validar el secreto. Es una comprobación no encontrada, **no una demostración de explotación o recuperación de claves**.

### Estado del canal

| Etapa | TX | RX | Evidencia |
| --- | --- | --- | --- |
| Registro inicial de identificación | Usa 0 | Todavía no recibe órdenes de aplicación en este paso | `0x41cc–0x41e0` |
| Identificación enviada correctamente | Siguiente valor 1 | Siguiente valor 0 | `0x4244–0x4268`; se activa el indicador de sesión |
| Envío ordinario | Usa valor anterior y almacena TX+1 | Independiente | `0x5fdc–0x6008`; dirección 0 |
| Recepción protegida | Independiente | Usa valor anterior y almacena RX+1 antes de verificar | `0x4644–0x4674`; dirección 1 |
| Etiqueta inválida | Se abandona la conexión | No se retrocede para probar otros valores | `0x4684` → liberación → `0x4084` → cierre/reintento |
| TX o RX > `0x40000000` | Cierre y reinicio | Cierre y reinicio | `0x32bc–0x330c`; nuevo establecimiento |

El nonce contiene un byte de dirección, siete ceros y un contador de mensaje de 32 bits big-endian. Es distinto del contador de bloques de ChaCha20. La fragmentación TCP no consume números adicionales; el prefijo de longitud cero vuelve al bucle sin consumir RX.

La verificación de los 16 bytes de Poly1305 precede al descifrado y al despachador. Una etiqueta inválida provoca cierre, no ejecución. La longitud de cuerpo aceptada en el bucle es de 1 a 4096 bytes, más la etiqueta cuando está protegido; el prefijo no se incluye como AAD en las rutinas inspeccionadas.

**Repetición:** el siguiente contador esperado cambia el nonce, por lo que repetir un registro anterior no equivale a un mensaje nuevo válido dentro de la sesión revisada. No es una prueba formal entre sesiones: depende de renovar correctamente las claves. El umbral permite exactamente `2^30` y cierra al superarlo; está en el bucle principal, no en cada cifrado. La ruta de envío tiene sincronización, pero todos los intercalados de hilos no se han verificado.

### Qué continúa pendiente

- Equivalencia integral de la aritmética X25519 y tratamiento de entradas degeneradas.
- Impredecibilidad del generador alternativo cuando falla `/dev/urandom`.
- Auditoría del protocolo de autenticación completo y carreras de estado/reconexión.
- PCAP y secretos de sesión autorizados para contrastar el modelo con tráfico histórico.

Se verificaron 38 instrucciones contra los bytes y 11 reglas de estado. Las 72 pruebas matemáticas de mensajes con claves artificiales verifican el **modelo propio**, no una traza MIPS ni interoperabilidad real. No se creó un cliente operativo del C2 para este informe.

## Órdenes del C2

Los nombres describen el efecto reconstruido; no son los nombres del protocolo.

| ID | Capacidad | Referencias | Límites |
| --- | --- | --- | --- |
| 0 | Lanza un trabajo de ataque con duración, método, objetivos y opciones | `0x47cc` → parser `0x147a8` → motor `0xa2c0` | Eficacia no probada; métodos [abajo](#métodos-de-la-orden-0) |
| 2 | Responde con 6 bytes calculados por una rutina de capacidad | `0x14b04`, respuesta `0x82` | Significado de los campos pendiente |
| 3 | Descarga una carga y reemplaza el proceso del malware | `0x64b4` → `0x748c` → descarga `0x6c38` → ejecución `0x7200` | BLAKE2s-256 opcional, con el valor esperado enviado por el controlador. Admite un servidor de descarga distinto del C2. No es actualización de firmware |
| 4 | Shell remoto (`sh -c` en un proceso hijo) | `0x4a24`; `0x4b18–0x4b38` | Con los permisos de la ejecución de `u`; no se demostró una sesión activa de `u` como root |
| 5 | Reconocimiento: UID/EUID, RAM, CPU, sistema de archivos, socket raw, posible medición UDP local | `0x194ec`, `0x60d4`, respuesta `0x87` | No se ha demostrado que mida ancho de banda real de Internet |
| 16 | Autodesactivación si el segundo byte es 16: para tareas, crea `.akataru-dead` y sale con código 77 | `0x4b48` | No se demostró eliminación completa de persistencia |
| 17 | Detiene los trabajos de ataque | `0x4cec` → `0xa224` | No desinstala |
| 18 | Descubrimiento y propagación: iniciar, detener, consultar | `0x2bda4`, estado `0x88` | 23 selectores; no hay evidencia de infecciones remotas exitosas |
| 19 | Operaciones por ADB: iniciar, detener, consultar | `0x2ffc4`, estado `0x89` | Puerto 5555, directo o por proxy. Requiere que ADB sea accesible y autorizado |
| 1, 6–15 | Descartadas por la tabla general | — | No confundir con el tratamiento previo del byte 15 |

### Selectores de propagación (orden 18)

`hikvision` · `boa` · `tplink` · `realtek` · `gpon` · `netgear` · `zyxel` · `jaws` · `vacron` · `dasan` · `tenda` · `dlink` · `telesquare` · `alcatel` · `mebrtu` · `ssh_brute` · `eir` · `huawei` · `dlinkupnp` · `gpon443` · `zte` · `linksys` · `asus`

Son etiquetas internas del binario. No acreditan un modelo vulnerable concreto, un CVE ni el vector que infectó esta ONU. Que aparezcan `gpon`, `gpon443`, `zte`, `huawei`, `dasan` y `alcatel` sí es relevante para el inventario de CPE del ISP.

### ADB (orden 19)

- Envía órdenes al servicio shell de ADB y examina la salida.
- Los textos sobre instalación Android **no** prueban que exista un APK fijo ni que se instalara alguno.
- `proxy.netloxproxies.com` pertenece a la ruta alternativa (por proxy). No es el C2 principal y no se consultó.

## Métodos de la orden 0

> El **ID de orden** y el **ID de método** son tablas distintas: la orden 18 es propagación, el método 18 es HTTP GET.

Hay 22 entradas, una de ellas alias (17 → 6). Que un método esté registrado no prueba que sea distinto, interoperable o eficaz.

| ID | Capacidad | Confianza / límites |
| --- | --- | --- |
| 0 | UDP con cabeceras IPv4/UDP propias (raw) | Alta |
| 3 | TCP SYN configurable; perfiles linux/windows/android; reintentos | Alta |
| 4 | TCP ACK con carga configurable; flags parametrizables | Alta |
| 5 | TCP: establece la conexión y luego envía tráfico construido | Alta para TCP; media para la etiqueta "STOMP". No secuestra conexiones ajenas |
| 6 | GRE (47), tipo 0x0800, con IPv4/UDP interior | Alta |
| 7 | GRE con Ethernet/IP/UDP interior (+14 bytes respecto al método 6) | Alta (estructura) |
| 9 | UDP con sockets de datagramas | Alta |
| 15 | TCP SYN de 60 bytes con opciones y variación de campos | Alta |
| 17 | Alias del método 6 (`0x12ef0` → `0x126c8`) | Alta |
| 18 | HTTP/1.1 GET; TLS directo; motor compartido | Alta. Sin segunda capa TLS observada en la rama CONNECT revisada |
| 22 | ICMP Echo Request (tipo/código 8/0) | Alta |
| 24 | Tres ramas: TCP sostenido, HTTP con cabeceras incompletas (opcionalmente sobre TLS), SSH hasta KEXINIT | Alta |
| 26 | QUIC Initial sintético, UDP, 1200 bytes | Alta para la estructura; validez del handshake no demostrada |
| 27 | Intentos de autenticación SSH por contraseña; reporta credenciales al obtener éxito (mensaje `0x83`) | Alta. No abre shell; es distinto de `ssh_brute` (orden 18) |
| 28 | Control OpenVPN sobre UDP/1194, cabecera `0x38`; variante raw o datagrama | Alta |
| 30 | HTTP/1.1 POST; cuerpo recibido o aleatorio; TLS directo | Alta. Sin segunda capa TLS observada en la rama CONNECT revisada |
| 32 | HTTP/1.1 con perfiles de navegador y cookies fabricadas; TLS directo | Alta. No demuestra evasión de WAF |
| 33 | HTTP/2 sobre TLS: SETTINGS + HEADERS (END_STREAM/END_HEADERS), flujos impares, reconexión | Alta. No se observó RST_STREAM en el trabajador revisado; no se atribuye Rapid Reset |
| 34 | WireGuard: iniciaciones de 148 bytes, UDP/51820, Noise/mac1 | Alta. No revela claves de VPN legítimas |
| 35 | Minecraft: handshake/login, VarInt, TCP/25565, nombres aleatorios | Alta (estructura) |
| 37 | Tipo FiveM/Quake (`ff ff ff ff` + connect) sobre **TCP**/30120 | Alta para bytes y transporte; media para atribuir la familia del protocolo |
| 38 | ClientHello TLS sintético repetido sobre TCP | Alta. No espera ServerHello; es distinto del cliente TLS 1.3 compartido |

Las capacidades reales dependen de permisos, configuración y conectividad. Que el parser tenga una opción de falsificar el origen no demuestra que la falsificación sea posible: depende además de que la red lo permita (BCP 38).

## Downloader

| Bytes | SHA-256 |
| --- | --- |
| 1984 | `dd75cf7600b31a2f7f6ad1c62fd661a8855637197c7195b0752a909400c68230` |

- Conecta a **`170.205.28.207:80`** y envía **`GET /mips HTTP/1.0`**.
- Escribe el cuerpo de la respuesta en **`.liso`**, en el directorio de trabajo.
- No se observó ejecución del archivo descargado en el flujo analizado.
- No se observó validación criptográfica de la descarga en el flujo analizado.
- Es infraestructura incorporada en el código: no se realizó ninguna conexión durante el análisis.

Relación con el resto: el nombre de salida coincide con `.liso`, pero no se ha demostrado que lo descargado sea la muestra `.liso` analizada. El descargador del caso A no está disponible.

## Relación con las otras muestras

- `u` y `.a` comparten `143.244.144.70:443`. Eso **no** demuestra que sean la misma muestra.
- El despachador y los métodos de `u` **no** se trasladan a `.liso`, `.a` ni `.ffaaxx`.

## Evidencia complementaria

- [Intercambio inicial](evidencia/intercambio-y-alternativas-c2-u.txt).
- [Tabla y cifrado de mensajes](evidencia/puerto-y-cifrado-u.txt).
- [Métodos con VA y límites](evidencia/metodos-orden-0-u.csv).
- [Selectores de propagación](evidencia/modulos-propagacion-orden-18-u.csv).
- [Ampliación HTTP/TLS/SSH](evidencia/ampliacion-http-tls-ssh-u.txt).
