# Compromiso de ONU GPON MIPS — botnet DDoS con persistencia en `/var/config`

| Campo | Valor |
| --- | --- |
| Organización | MACTELCO SAS |
| Estado | Compromiso confirmado · análisis en curso |
| Versión | 1.2 — 2026-11-08 (ver [CHANGELOG](CHANGELOG.md)) |
| Equipos declarados | ZX8202DWT (caso A) · ZX8102WT (caso B); fabricante comercial por corroborar |

> Los principales destinos sospechosos se presentan desactivados (`[.]`) para evitar enlaces automáticos. Los valores sin modificar están en [`iocs/`](iocs/).

## TL;DR

- Dos casos de ONU GPON comprometidas. Se observaron procesos maliciosos como **root**, con camuflaje `libssl.so` en los procesos documentados; no se demuestra que todas las muestras estuvieran activas.
- **Persistencia:** el implante modifica `/var/config/run_customized_sdk.sh` en la partición UBIFS de lectura/escritura. Este script se ejecuta en cada arranque en modo GPON, así que **reiniciar o cambiar la contraseña no limpia el equipo**.
- Se observó a `.liso` abriendo conexiones masivas a `104.233.223[.]169:80`, un patrón compatible con DDoS.
- **C2:**
  - `.liso` descubre su C2 mediante TXT de `jxk.imarapi[.]st` vía DoH (Google, Cloudflare, Quad9, AdGuard), con respaldo UDP a `8.8.8.8:53`.
  - `u` usa `143.244.144[.]70:443` con un protocolo propio (X25519 + ChaCha20/Poly1305, **no TLS**).
  - `.ffaaxx` resuelve infraestructura vía ENS `2busydrinkingcodeine.eth`.
- **Análisis estático de `u`:** shell remoto, sustitución del ejecutable, 23 selectores de propagación y 22 entradas de métodos de ataque, incluido un alias. No demuestra órdenes históricamente ejecutadas.
- **Autenticación y contadores de `u`:** el flujo revisado cierra ante etiqueta inválida y renegocia al superar `2^30` en TX o RX. No se certifica la seguridad completa del protocolo.
- **Atribución:** familia y operador no identificados; las coincidencias de código e infraestructura no atribuyen autoría.
- **Vector de entrada (hipótesis):** Telnet con credenciales por defecto, posiblemente desde un TV Box comprometido en la LAN.

**Qué hacer ya:** [Detección](#detección-a-escala-de-red) → [Remediación](#remediación) → [IOC](#indicadores-de-compromiso).

## Casos y muestras

Las fechas son de informe y revisión, no de infección: las marcas de tiempo del equipo están en 1970. Que dos muestras coincidan en dominio, IP o tamaño no demuestra que sean el mismo ejecutable.

| Caso | Equipo | Fecha documentada | Muestras |
| --- | --- | --- | --- |
| A — original | ZX8202DWT · HW 2.0.0.02.1 · SW V15.5.3 | Informe 2026-11-02 | `.ffaaxx`, `.liso`, `.a`, `hmm` |
| B — adicional | ZX8102WT · HW 1.0.0.03.0 · SW V12.2.44 | Ampliación estática 2026-11-08 | `.liso`, `u`, downloader |

| Caso | Muestra | Bytes | MD5 | SHA-256 |
| --- | --- | --- | --- | --- |
| A | `.ffaaxx` (`pid259.bin`) | 652524 | `3158380a9338e21e9db3dea68671dd59` | `0afa465224ae07e025b5bea1ba5b29618c3309754a9d471ac85958600ac66b5f` |
| A | `.liso` (`pid262.bin`) | 631912 | `5c8ce65398a1723adf4e6406f0c34800` | `4063cdb6e7b6427a82ccf160fc3a13e645e3bc87428eb094aa12b6b32e8306e7` |
| A | `.a` (`pid3437.bin`) | 310808 | `965c86dc0c3db91de5b3a2afcfe38742` | `4707d314f078b84b217e0cc9e6a02283f262c71dbafe5389ba9589e03fcb4c39` |
| A | `hmm` (vacío) | 0 | `d41d8cd98f00b204e9800998ecf8427e` | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| B | `.liso` | 631912 | `3d22d131c80d94f3a620575d0c94d8c8` | `0c0342c4f3d1a976e3344d2f1508ea298fc8e7c65a513d080b17eca76f3315ef` |
| B | `u` (empaquetado) | 169704 | `8020256b6b282d64d3aed3f4e617bb88` | `7208f46f1f2f1f8204d91c39570ab54743535a8a4b5d00d1c335c42859e437c4` |
| B | downloader | 1984 | `e83c6103fa160db22e58b084a2381916` | `dd75cf7600b31a2f7f6ad1c62fd661a8855637197c7195b0752a909400c68230` |

- Los hashes del caso A proceden del informe original; sus MD5 se verificaron en el equipo antes de la transferencia (ver [cadena de custodia](docs/metodologia.md#recolección-e-integridad)).
- Los hashes del caso B se calcularon sobre las muestras locales.
- El hash de `hmm` es el de un archivo vacío y no es un indicador por sí solo.

**Identificación pendiente:** el caso B tiene una versión declarada en interfaz (`V12.2.44`) y una compilación observada en consola (`V4.0.0-240520`). No se demuestra que sean el mismo campo. El chipset y las marcas internas no corroboran por sí solos el fabricante comercial. Véase [firmware y cuentas](docs/firmware-cuentas.md).

## Cómo leer la evidencia

Cada sección indica su nivel de evidencia una sola vez, en la línea que la encabeza, y aplica hasta la siguiente sección.

| Etiqueta | Significado | Límite |
| --- | --- | --- |
| 🟦 **Observado** | Registrado en consola durante la recolección | No prueba función ni actividad anterior o posterior |
| 🟩 **Estático** | Bytes, datos descifrados o ruta de código identificados | Describe lo que el programa contiene o intenta, no que se ejecutara |
| 🟨 **Hipótesis** | Compatible con la evidencia | Pendiente de confirmación |
| ⬜ **Informe original** | Resultado del caso A | No revalidado contra los binarios en la ampliación del caso B |

## Hallazgos

### Procesos — caso A

> 🟦 Observado · ⬜ informe original

Todos los procesos corren como root, con capacidades efectivas amplias, nombre `libssl.so` y argumentos en blanco. Los PID no son indicadores estables.

| PID | Ejecutable | Actividad |
| --- | --- | --- |
| 259 | `/var/config/o/.ffaaxx` | Conexiones salientes TCP/443 |
| 262 | `/var/config/xxq/.liso` | Conexión a `178.128.60[.]155:443` |
| 25301 | `/var/config/xxq/.liso` (hijo de 262) | Conexiones masivas a `104.233.223[.]169:80` |
| 3437 | `/var/tmp/.a` (deleted) | Conexión a `143.244.144[.]70:443`; escucha en `0.0.0.0:42363/TCP` |

### Persistencia en modo GPON

> 🟦 Observado · 🟩 Estático (script) · ⬜ informe original

```text
ubi0:ubi_Config /var/config ubifs rw,relatime 0 0
```

Con `PON_MODE=1`, `/etc/runsdk.sh` hace lo siguiente:

1. Copia `/etc/run_customized_sdk.sh` a `/var/config/run_customized_sdk.sh` **si este no existe**.
2. Ejecuta `/var/config/run_customized_sdk.sh`.
3. Continúa con OMCI y la detección PON.

La copia de `/etc` está vacía. La de `/var/config` contiene:

```sh
chmod 777 /var/config/hmm; /var/config/hmm postsec1
chmod 777 /var/config/o/.ffaaxx; /var/config/o/.ffaaxx vodkat
chmod 777 /var/config/xxq/.liso; /var/config/xxq/.liso poon
```

`/var/config/hmm` tenía 0 bytes al recolectarlo; su contenido previo es desconocido.

Esto describe el mecanismo de persistencia. No se hizo una prueba de reinicio durante la recolección.

### Resumen por muestra

| Muestra | Caso | Función | Detalle |
| --- | --- | --- | --- |
| `.liso` | A, B | Actividad de conexiones masivas observada en A; TXT/DoH y XOR verificados en B; TLS 1.3 según informe A | [docs/liso.md](docs/liso.md) |
| `.ffaaxx` | A | Resolución de infraestructura vía ENS (`eth_call`) y DoH; peticiones HTTP | [docs/ffaaxx-a.md](docs/ffaaxx-a.md) |
| `.a` | A | Generador de tráfico HTTP (agotamiento de recursos); ejecutado desde un archivo borrado | [docs/ffaaxx-a.md](docs/ffaaxx-a.md) |
| `u` | B | Control remoto, sustitución del ejecutable, propagación, ADB y 22 entradas de métodos | [docs/u-c2.md](docs/u-c2.md) |
| downloader | B | `GET /mips` a `170.205.28[.]207:80` → escribe `.liso` (no lo ejecuta) | [docs/u-c2.md](docs/u-c2.md#downloader) |

### Vector de acceso

> 🟨 Hipótesis · casos A y B

Telnet con credenciales por defecto es compatible con depositar los binarios y modificar el script persistente. Sin registros de la sesión inicial ni un firmware de referencia, esto no demuestra el vector concreto.

El TV Box comprometido en la misma LAN es una hipótesis de origen; no se ha identificado su IP ni su MAC.

> 🟦 Observado · ⬜ informe original

La sesión Telnet identificada por el operador como propia corresponde a la investigación. No se presenta como origen del atacante; su dirección se omite del paquete público.

## Mapeo MITRE ATT&CK

| Táctica | Técnica | Evidencia |
| --- | --- | --- |
| Initial Access | T1078.001 Default Accounts | 🟨 Telnet con credenciales por defecto |
| Execution | T1059.004 Unix Shell | 🟩 `u` orden 4 (`sh -c`) |
| Persistence | [T1037.004 RC Scripts](https://attack.mitre.org/techniques/T1037/004/) | 🟦 `/var/config/run_customized_sdk.sh` |
| Defense Evasion | T1036.005 Match Legitimate Name | 🟦 proceso `libssl.so` |
| Defense Evasion | T1070.004 File Deletion, mapeo provisional | 🟦 `/var/tmp/.a` aparece eliminado; no demuestra quién lo borró ni su finalidad |
| Defense Evasion | T1027 Obfuscated Files or Information | 🟩 tabla de configuración cifrada; `u` empaquetado (LZMA) |
| Discovery | T1082 System Information Discovery | 🟩 `u` orden 5 |
| Lateral Movement | T1210 Exploitation of Remote Services | 🟩 `u` orden 18 (23 selectores; éxito no demostrado) |
| Credential Access | T1110.001 Password Guessing | 🟩 `u` método 27 y selector `ssh_brute` |
| Command and Control | T1071.004 DNS | 🟩 TXT de `jxk.imarapi[.]st` |
| Command and Control | [T1572 Protocol Tunneling](https://attack.mitre.org/techniques/T1572/), en la resolución DoH | 🟩 DNS encapsulado en HTTPS; no implica órdenes C2 dentro de DNS |
| Command and Control | T1568 Dynamic Resolution | 🟩 TXT (`.liso`), ENS (`.ffaaxx`) |
| Command and Control | T1573 Encrypted Channel | 🟩 TLS 1.3 (`.liso`); X25519/ChaCha20 (`u`) |
| Command and Control | T1105 Ingress Tool Transfer | 🟩 downloader; `u` orden 3 |
| Impact | T1498 Network DoS / T1499 Endpoint DoS | 🟦 conexiones masivas; 🟩 tabla de métodos |

El mapeo ATT&CK es una interpretación analítica por conducta, no atribución a un grupo. Se distinguen capacidades estáticas e impacto observado; la finalidad de borrar `.a` sigue sin confirmarse.

## Detección a escala de red

Cómo encontrar otras ONU afectadas. Detalle y consultas en [docs/deteccion-remediacion.md](docs/deteccion-remediacion.md).

| Señal | Fuente | Por qué |
| --- | --- | --- |
| ONU → `8.8.8.8`, `8.8.4.4`, `1.1.1.1`, `1.0.0.1`, `9.9.9.9`, `149.112.112.112`, `94.140.14.14`, `94.140.15.15` en **TCP/443** | NetFlow/sFlow (Akvorado) | Señal contextual si el origen es la gestión de la ONU; también puede ser tráfico legítimo del abonado |
| Consulta **TXT** `jxk.imarapi.st` | Sensor DNS en tránsito; logs del ISP solo si recibe la consulta | El respaldo va a 8.8.8.8, no necesariamente al resolvedor del ISP |
| ONU → `143.244.144.70`, `178.128.60.155`, `185.173.39.100`, `170.205.28.207` | NetFlow / Suricata | C2 y descarga conocidos |
| ONU → `87.58.159.3`, `43.157.198.137`, `159.223.66.236` en TCP/443 | NetFlow | Conexiones observadas en el caso A (función no clasificada) |
| ONU con TCP/42363 abierto | Escaneo interno | Escucha de `.a` |
| ONU → TCP/5555 de terceros | NetFlow | Operaciones ADB de `u` (orden 19) |
| Ráfagas salientes desde ONU (SYN, UDP, GRE, ICMP, HTTP) | NetFlow | Métodos DoS de `u` |
| Telnet/23, mapeos externos como TCP/65000, o gestión web alcanzable desde la WAN | Escaneo externo | Vector de entrada hipotético |

Esta edición contiene únicamente documentación, indicadores y extractos de análisis. El desarrollo de herramientas y reglas de detección queda para una fase posterior; véase [validación](docs/validacion.md).

## Remediación

Resumen; los criterios y procedimientos están en [docs/deteccion-remediacion.md](docs/deteccion-remediacion.md#7-recuperación).

1. **Contener fuera de la ONU:** cuarentena o política de salida con destinos de gestión autorizados; considerar IPv4 e IPv6 y separar tráfico de la ONU del abonado. Las listas de IP son una medida complementaria y temporal.
2. **Cortar las sesiones existentes** en el equipo de red donde se aplican los filtros; impedir conexiones nuevas no necesariamente termina una sesión ya establecida.
3. **Conservar evidencia** antes de limpiar: procesos, sockets, binarios, script persistente y hashes. No registrar credenciales en el repositorio público.
4. **Recuperar desde una referencia confiable:** firmware/configuración compatibles, verificados con el proveedor real del equipo. La limpieza manual es contención provisional; no demuestra que se hayan eliminado todas las modificaciones de root.
5. **Restringir gestión:** acceso solo desde la red autorizada, credenciales únicas, Telnet deshabilitado cuando sea viable y revisión de mapeos WAN/LAN.
6. **Verificar recuperación:** comparar con el estado esperado del modelo, vigilar reinfección y revisar los equipos de la LAN bajo la hipótesis aún no confirmada del TV Box.

**DNS:** una RPZ para `jxk.imarapi.st` solo afecta consultas que atraviesen el resolvedor controlado. `.liso` incorpora DoH y DNS externo; una política sobre el origen afectado debe cubrir ambos. Los resolvedores públicos son servicios legítimos y no se consideran C2.

**Reset de fábrica:** no se verificó que limpie `/var/config`. Probarlo únicamente en una unidad de laboratorio con recuperación y aprovisionamiento disponibles.

## Indicadores de compromiso

Versión legible por máquina en [`iocs/iocs.csv`](iocs/iocs.csv) y [`iocs/hashes.csv`](iocs/hashes.csv). Las conexiones observadas son históricas; no se comprobó su titularidad ni su vigencia.

### Destinos y nombres referenciados — función diferenciada

| Indicador | Muestra | Función | Evidencia |
| --- | --- | --- | --- |
| `jxk.imarapi[.]st` | `.liso` (A, B) | Descubrimiento de C2 por TXT | 🟩 A ⬜, B |
| `2busydrinkingcodeine.eth` | `.ffaaxx` (A) | Nombre ENS; la cadena contrato → C2 está pendiente | 🟩 ⬜ |
| `178.128.60[.]155:443` | `.liso` (A) | C2 histórico según informe original | 🟦 ⬜ |
| `185.173.39[.]100:443` | `.liso` (B) | Conexión saliente atribuida al proceso; protocolo de esa sesión no descifrado | 🟦 consola del caso B |
| `143.244.144[.]70:443` | `.a` (A), `u` (B) | Conexión observada (A); C2 predeterminado incorporado (B). Compartirla no demuestra que las muestras sean idénticas | 🟦 ⬜ / 🟩 |
| `170.205.28[.]207:80` `/mips` | downloader (B) | Servidor de descarga | 🟩 |
| `proxy.netloxproxies[.]com` | `u` (B) | Ruta alternativa de ADB; no es el C2 principal | 🟩 |
| `87.58.159[.]3:443`, `43.157.198[.]137:443` | `.ffaaxx` (A) | Sin clasificar (C2, RPC o DoH) | 🟦 ⬜ |
| `159.223.66[.]236:443` | sin atribuir (A) | Función y proceso sin determinar | 🟦 ⬜ |

### Víctima y local

| Indicador | Función | Evidencia |
| --- | --- | --- |
| `104.233.223[.]169:80` | Destino de conexiones masivas; objetivo DDoS (🟨). No es C2 | 🟦 ⬜ |
| `0.0.0.0:42363/TCP` | Escucha local de `.a`; función pendiente | 🟦 ⬜ |

### Servicios legítimos usados por las muestras

No son infraestructura del atacante. Sirven para detectar comportamiento anómalo, pero **no deben bloquearse globalmente**.

| Servicio | Uso |
| --- | --- |
| `dns.google`, `cloudflare-dns.com`, `dns.quad9.net`, `dns.adguard-dns.com` (8 IP, TCP/443) | Resolvedores DoH de `.liso` (B) |
| `8.8.8.8:53/UDP` | Respaldo DNS de `.liso` (B) |
| `public-eth.nownodes.io`, `eth.drpc.org` | RPC de Ethereum de `.ffaaxx` (A ⬜) |

### Host

| Indicador | Muestra | Evidencia |
| --- | --- | --- |
| `/var/config/run_customized_sdk.sh` (con lanzamientos maliciosos) | persistencia | 🟦 ⬜ |
| `/var/config/o/.ffaaxx` | `.ffaaxx` | 🟦 ⬜ |
| `/var/config/xxq/.liso` | `.liso` | 🟦 ⬜ |
| `/var/config/hmm` | `hmm` | 🟦 ⬜ |
| `/var/tmp/.a` (deleted) | `.a` | 🟦 ⬜ |
| `.liso` en el directorio de trabajo | downloader | 🟩 |
| `.akataru-dead` | `u` (orden 16) | 🟩 |
| Socket abstracto `@fdrunkkkk@@@@5` | `.liso` | 🟩 A ⬜, B |
| Socket abstracto `@drunkasfk` | `.ffaaxx` | 🟩 ⬜ |
| Proceso `libssl.so` con argumentos en blanco, ejecutándose desde `/var/config` o `/var/tmp` | `.liso`, `.ffaaxx` | 🟦 ⬜ |
| Argumentos de lanzamiento `postsec1`, `vodkat`, `poon` | script | 🟦 ⬜ |

## Línea de tiempo

| Fecha | Evento |
| --- | --- |
| Desconocida | Infección (marcas de tiempo en 1970) |
| No documentada | Detección del caso A; no se infiere del reloj de la ONU |
| No documentada | Recolección del caso A (consola y archivo citado en informe original) |
| 2026-11-02 | Informe original del caso A |
| No documentada con precisión | Recolección autorizada del caso B; hashes en origen/destino según informe local |
| 2026-11-08 | Ampliación estática del caso B |

## Coordinación y notificaciones

No se enviaron notificaciones durante esta revisión. No se presenta ningún proveedor como responsable del ataque. Antes de un aviso, corroborar titularidad histórica, fechas UTC y papel del destino; redactar datos de abonados.

| Posible destinatario | Motivo / condición | Estado documentado |
| --- | --- | --- |
| Proveedor de alojamiento del C2 o descarga | Evidencia fechada que vincule el destino, muestra y actividad | No consta notificación |
| Proveedores de destinos sin clasificar | Primero determinar función; no denunciarlos automáticamente como C2 | Investigación pendiente |
| Registrador/registro del dominio de descubrimiento | Hallazgo de configuración por muestra y fechas disponibles | No consta notificación |
| Equipo nacional de respuesta aplicable | Coordinación del incidente con el ISP | No consta notificación |
| Fabricante/distribuidor real del equipo | Confirmar modelo, firmware y mecanismo de recuperación | Identidad comercial pendiente de corroboración |
| Titular del destino de conexiones masivas | Compartir evidencia sin afirmar impacto no medido | No consta notificación |

## Alcance y limitaciones

- **Análisis de muestras:** no se ejecutaron ni emularon las muestras. La recolección sí utilizó gestión autorizada del equipo. Las verificaciones locales no consultaron DNS, DoH, nodos Ethereum ni C2; la consulta de documentación pública no es comunicación con sus indicadores.
- **Material disponible:** no hay PCAP, volcado de memoria, firmware de referencia ni registros de la sesión inicial.
- **Muestras del caso A:** `.ffaaxx`, `pid259.bin` y `onu-evidencia.tar` no estaban disponibles para la ampliación del caso B.
- **Sin extrapolación:** los detalles de `.liso` del caso B no se trasladan automáticamente al binario del caso A ni a `.ffaaxx`.

Metodología, herramientas y reproducibilidad: [docs/metodologia.md](docs/metodologia.md).

## Estado de los pendientes

El seguimiento con evidencia necesaria para cerrar cada punto está en [docs/pendientes.md](docs/pendientes.md).

| Estado | Resultado |
| --- | --- |
| Completado en esta revisión | MD5 y SHA-1 de las tres muestras B, además de SHA-256; versiones de herramientas; mapeo de segmentos ELF |
| Completado en el recorrido revisado de `u` | Comprobaciones iniciales, cierre ante error de etiqueta, contadores TX/RX y umbral de reconexión `2^30` |
| Completado como documentación | Evidencia estática publicable, matriz de afirmaciones, indicadores por función y plan de contención |
| Pendiente técnico | Auditoría integral de X25519/autenticación/concurrencia; catálogo de órdenes de `.liso`; cadena ENS de `.ffaaxx`; función TCP/42363 de `.a` |
| Pendiente de evidencia | Muestras A originales; firmware limpio compatible; origen de `rzfiber` y `admin` duplicado; fechas de recolección; PCAP y alcance real |
| Pendiente operativo | Desarrollar y validar reglas con los motores objetivo y un corpus limpio; comprobar recuperación/reset en laboratorio |

La familia, el operador y el número total de dispositivos de la botnet siguen **sin determinar**. No se visitaron los C2 ni se intentó enumerar su flota para completar esos puntos.

## Estructura del repositorio

```text
.
├── README.md
├── CHANGELOG.md
├── MANIFEST-SHA256.txt
├── docs/
│   ├── liso.md
│   ├── u-c2.md
│   ├── ffaaxx-a.md
│   ├── deteccion-remediacion.md
│   ├── firmware-cuentas.md
│   ├── metodologia.md
│   ├── pendientes.md
│   ├── validacion.md
│   ├── fuentes.md
│   └── evidencia/                 extractos estáticos y resultados documentados
└── iocs/
    ├── iocs.csv
    └── hashes.csv
```

Esta entrega es **solo documentación y datos de análisis**: no incluye herramientas, scripts, reglas ejecutables, clientes C2 ni muestras de malware.
