# Compromiso de ONU GPON con persistencia y actividad DDoS

Investigación: **MACTELCO SAS** · Clasificación: confidencial · Estado: compromiso confirmado, análisis en curso

## Resumen

MACTELCO identificó una ONU GPON ZX8202DWT comprometida, con tres ejecutables maliciosos activos como root. Dos de ellos persisten en la partición UBIFS `/var/config` y se lanzan desde el script de personalización que el firmware ejecuta al arrancar en modo GPON.

El binario `.liso` genera conexiones TCP masivas hacia 104.233.223.169:80, patrón compatible con participación en un DDoS. Su canal de control se reconstruyó por análisis estático: dominio cifrado `jxk.imarapi.st`, descubrimiento de la IP de control vía registros DNS TXT y sesión TLS 1.3 sobre TCP/443.

Un segundo binario, `.ffaaxx`, resuelve infraestructura a través de Ethereum (JSON-RPC `eth_call` sobre el nombre ENS `2busydrinkingcodeine.eth`) y DNS sobre HTTPS. El tercero, `.a`, se ejecuta desde un archivo ya eliminado y contiene plantillas de inundación HTTP.

| Dato | Valor |
| --- | --- |
| Equipo | ONU GPON de cliente residencial |
| Modelo / firmware | ZX8202DWT · HW 2.0.0.02.1 · SW V15.5.3 |
| Arquitectura de las muestras | ELF MIPS 32 bits, big-endian, sin símbolos |
| Fuentes | Salidas de consola del operador y onu-evidencia.tar |
| Fecha del informe | 2 de noviembre de 2026 |

## Procesos y ejecutables

Los procesos se ejecutan como root con capacidades efectivas amplias y se camuflan con el nombre `libssl.so` y argumentos en blanco. Los PID corresponden a las capturas y no son indicadores estables.

| PID | Ejecutable | Observado |
| --- | --- | --- |
| 259 | `/var/config/o/.ffaaxx` | Conexiones salientes a TCP/443 |
| 262 | `/var/config/xxq/.liso` | Conexión a 178.128.60.155:443 |
| 25301 | `/var/config/xxq/.liso` (hijo de 262) | Conexiones masivas a 104.233.223.169:80 |
| 3437 | `/var/tmp/.a` (deleted) | Conexión a 143.244.144.70:443; escucha en 0.0.0.0:42363/TCP |

## Persistencia en modo GPON

El atacante modificó un script legítimo de personalización almacenado en la partición persistente, sin necesidad de tocar el firmware de solo lectura. La infección sobrevive a reinicios y a cambios de contraseña.

```
ubi0:ubi_Config /var/config ubifs rw,relatime 0 0
```

Con `PON_MODE=1`, `/etc/runsdk.sh`:

1. Copia `/etc/run_customized_sdk.sh` a `/var/config/run_customized_sdk.sh` si este no existe.
2. Ejecuta `/var/config/run_customized_sdk.sh`.
3. Continúa con OMCI y la detección PON.

La copia en `/etc` está vacía. La de `/var/config` contiene:

```
chmod 777 /var/config/hmm; /var/config/hmm postsec1
chmod 777 /var/config/o/.ffaaxx; /var/config/o/.ffaaxx vodkat
chmod 777 /var/config/xxq/.liso; /var/config/xxq/.liso poon
```

`/var/config/hmm` pesa 0 bytes al momento de la recolección; su contenido previo es desconocido.

## Análisis de las muestras

Los binarios se recuperaron desde `/proc/PID/exe` y se analizaron de forma estática: cadenas, estructura ELF, desensamblado MIPS y reproducción aislada de la rutina que oculta su configuración. No se ejecutaron ni se contactaron sus destinos.

### .liso — bot DDoS

- Proceso hijo (PID 25301) que abre conexiones masivas a 104.233.223.169:80.
- Descifra el dominio `jxk.imarapi.st` y consulta sus registros TXT.
- Interpreta los TXT como IPv4, selecciona una y conecta por TCP/443.
- Negocia TLS con la suite `TLS13-AES256-GCM-SHA384`, envía un registro inicial y procesa órdenes binarias.
- Socket UNIX abstracto: `@fdrunkkkk@@@@5`.

### .ffaaxx — resolución vía Ethereum

- Construye peticiones HTTP con cookies, user-agents de navegador y cabeceras variables.
- Incluye cliente TLS y DNS sobre HTTPS.
- Envía JSON-RPC `eth_call` contra proveedores públicos de Ethereum para resolver el nombre ENS `2busydrinkingcodeine.eth`.
- Socket UNIX abstracto: `@drunkasfk`.

El uso de `eth_call` corresponde a lectura de un contrato, no a minería ni a robo de criptomonedas.

### .a — generador de tráfico

- Se ejecuta desde `/var/tmp/.a`, archivo ya eliminado.
- Conecta a 143.244.144.70:443 y escucha en 0.0.0.0:42363/TCP.
- Plantillas HTTP con parámetros aleatorios, `X-Forwarded-For`, `HTTP CONNECT` y una petición con `Content-Length: 2147483647`, orientadas a agotar recursos del destino.
- Contiene cadenas de SSH y WireGuard, posiblemente para imitar protocolos.

## Canal de control de .liso

El C2 no está codificado en el binario: se obtiene en tiempo de ejecución a partir de registros TXT, lo que permite rotar servidores sin actualizar las muestras. La conexión observada fue a 178.128.60.155:443.

1. Descifrado del dominio `jxk.imarapi.st`.
2. Consulta DNS TXT.
3. Parseo de las respuestas como IPv4 y selección de una.
4. Conexión TCP/443.
5. Handshake TLS.
6. Intercambio binario de órdenes.

| Dirección | Función |
| --- | --- |
| 0x4067a4–0x406804 | Obtención del destino mediante resolución |
| 0x406850–0x4068c0 | Preparación del puerto 443 y conexión |
| 0x406da8–0x406e2c | Establecimiento del canal TLS |
| 0x408210 | Procesamiento de respuestas TXT |

Los binarios persistentes incluyen peticiones `POST /dns-query` con `Content-Type: application/dns-message`. Con DoH, las consultas al dominio no aparecen en los registros del DNS local del operador.

## Vector de acceso

La ZX8202DWT con software V15.5.3 sale de fábrica con Telnet habilitado y credenciales por defecto. Cualquier equipo en la LAN del cliente puede autenticarse como administrador, depositar los binarios y modificar el script persistente; este es el vector de compromiso. El dispositivo de origen apunta a un TV Box comprometido en la misma red, aún sin IP/MAC identificada.

La sesión Telnet desde 38.191.216.1 presente en los registros corresponde a la investigación, no al acceso del atacante.

## Indicadores de compromiso

### Red

| Indicador | Contexto |
| --- | --- |
| `jxk.imarapi.st` | Dominio de descubrimiento de .liso (consultas TXT) |
| `2busydrinkingcodeine.eth` | Nombre ENS consultado por .ffaaxx |
| 178.128.60.155:443 | C2 observado de .liso |
| 143.244.144.70:443 | Conexión observada de .a |
| 87.58.159.3:443 | Conexión de .ffaaxx (PID 259) |
| 43.157.198.137:443 | Conexión de .ffaaxx (PID 259) |
| 159.223.66.236:443 | Conexión sin proceso atribuido |
| 104.233.223.169:80 | Objetivo del tráfico DDoS, no C2 |
| 0.0.0.0:42363/TCP | Puerto en escucha de .a |

### Host

| Indicador | Contexto |
| --- | --- |
| `/var/config/o/.ffaaxx` | Binario persistente |
| `/var/config/xxq/.liso` | Binario persistente |
| `/var/config/hmm` | Referenciado por el script; 0 bytes |
| `/var/tmp/.a` | Binario eliminado en ejecución |
| `/var/config/run_customized_sdk.sh` | Script legítimo modificado para persistencia |
| `@drunkasfk` | Socket UNIX abstracto de .ffaaxx |
| `@fdrunkkkk@@@@5` | Socket UNIX abstracto de .liso |
| `libssl.so` | Nombre de proceso falso usado por .ffaaxx y .liso |

Los binarios también referencian servicios públicos legítimos (dns.google, cloudflare-dns.com, dns.quad9.net, dns.adguard-dns.com, public-eth.nownodes.io, eth.drpc.org), usados como resolvedores DoH y nodos RPC.

## Hashes

| Muestra | Tamaño (bytes) | MD5 | SHA-256 |
| --- | --- | --- | --- |
| .ffaaxx (pid259.bin) | 652524 | `3158380a9338e21e9db3dea68671dd59` | `0afa465224ae07e025b5bea1ba5b29618c3309754a9d471ac85958600ac66b5f` |
| .liso (pid262.bin) | 631912 | `5c8ce65398a1723adf4e6406f0c34800` | `4063cdb6e7b6427a82ccf160fc3a13e645e3bc87428eb094aa12b6b32e8306e7` |
| .a (pid3437.bin) | 310808 | `965c86dc0c3db91de5b3a2afcfe38742` | `4707d314f078b84b217e0cc9e6a02283f262c71dbafe5389ba9589e03fcb4c39` |
| hmm (hmm.bin) | 0 | `d41d8cd98f00b204e9800998ecf8427e` | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |

Los MD5 coinciden con los calculados en el equipo antes de la transferencia.

## Alcance y limitaciones

El análisis se basa en evidencia estática y capturas de consola. No hay PCAP, volcado de memoria, imagen de firmware de referencia ni registros de la sesión inicial, y las marcas de tiempo de 1970 impiden fechar la infección.

Queda sin determinar:

- IP/MAC del TV Box de origen.
- Familia del malware y actor.
- Número de equipos afectados.
- Tasa de paquetes e impacto real sobre 104.233.223.169.
- Catálogo completo de órdenes de .liso.
- Cadena completa ENS → contrato → C2 de .ffaaxx.
- Función del puerto TCP 42363 de .a.
- Los binarios descargadores, no incluidos en la evidencia.