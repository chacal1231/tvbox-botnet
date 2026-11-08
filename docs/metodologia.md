# Metodología y trazabilidad

[← README](../README.md)

## Alcance

Las muestras se leen como datos: estructura ELF, bytes, cadenas, desensamblado y descompresión. No se ejecutan ni emulan. Las pruebas propias usan claves sintéticas y redes de documentación. La revisión de este paquete no consulta DNS/DoH, ENS, nodos Ethereum ni servidores C2; sí consulta documentación pública de herramientas. No se envían muestras a servicios de terceros.

Los resultados del caso A provienen de su informe original, no de una nueva validación de sus binarios. La consola de un equipo comprometido aporta observaciones, no una referencia íntegra de firmware.

## Recolección e integridad

| Caso | Procedencia | Qué pudo verificarse |
| --- | --- | --- |
| A | Informe original: consola y extracción desde `/proc/PID/exe`; archivo `onu-evidencia.tar` citado | Se conservan sus hashes declarados. Archivo y muestras originales no disponibles para esta revisión; no se inventa una fecha o responsable de custodia |
| B | Sesión de gestión autorizada; extracción de downloader, `.liso` y `/var/tmp/u`; tratamiento de escapes Telnet y CR/NUL | Informe local documenta comparación de MD5 con origen. MD5, SHA-1 y SHA-256 recalculados sobre los archivos locales en esta revisión |

Fechas precisas, responsables nominales, ubicación del depósito original y SHA-256 del archivo A no están documentados aquí. Los hashes acreditan igualdad de bytes comparados; no acreditan por sí solos quién recolectó la evidencia ni una cadena de custodia completa.

## Herramientas verificadas localmente

| Uso | Herramienta | Versión verificada |
| --- | --- | --- |
| Scripts, hashes, descompresión | Python, `hashlib`, `lzma`, `struct` | Python 3.12.14 |
| Desensamblado MIPS32 big-endian | Capstone | 5.0.9 |
| ELF | pyelftools | 0.33 |
| Modelos criptográficos independientes | PyCryptodomex | 3.24.2 |
| Validación YARA por motor | No disponible | No realizada |
| Validación Suricata por motor | No disponible | No realizada |

Estas son versiones de las herramientas locales comprobadas, no versiones supuestas de un análisis anterior. Los resultados matemáticos no son trazas del ejecutable.

## Segmentos y direcciones

Fórmula general para bytes respaldados por archivo: **offset = p_offset + (VA − p_vaddr)**, dentro del segmento correspondiente. BSS no tiene necesariamente bytes en el archivo.

| Muestra | p_vaddr | p_offset | p_filesz |
| --- | --- | --- | --- |
| `.liso` B, primer PT_LOAD | `0x400000` | `0x0` | `0x93f40` |
| `.liso` B, segundo PT_LOAD | `0x4d4000` | `0x94000` | `0x61d4` |
| `u` reconstruido, primer PT_LOAD | `0x0` | `0x0` | `0x68000` |
| `u` reconstruido, segundo PT_LOAD | `0x78094` | `0x68094` | `0xfc0` |

La VA `.liso` `0x489340` equivale a offset `0x89340` por pertenecer al primer segmento. No se aplica esa resta al segundo segmento. En `u`, las direcciones publicadas se refieren a segmentos reconstruidos, nunca a offsets del archivo empaquetado.

## Desempaquetado de u

Se interpretaron cabeceras de bloques LZMA de método 14: offsets `0xac`, `0x124` y `0x2791f`, con salidas de 244, 425740 y 4032 bytes. Los primeros dos producen `0x68000` bytes; el tercero se coloca en offset `0x68094`, preservando el hueco. Resultado: 430164 bytes, SHA-256 `6c124574dc34dd4d06f930c51ba1548c980a775aa2130941fce4d7b9c7081ac2`.

No es una restauración íntegra de las secciones/metadatos originales. La salida se usa como datos para desensamblar; no debe ejecutarse. La interpretación de propiedades LZMA se contrastó con el [código de UPX](https://github.com/upx/upx/blob/devel/src/compress/compress_lzma.cpp).

## Comprobaciones publicadas

- [Configuración y TXT de `.liso`](evidencia/informe-configuracion-txt-liso.txt): diez entradas coherentes y vuelta al ciphertext; no prueba por sí sola equivalencia integral de la rutina.
- [Autenticación y contadores](evidencia/autenticacion-contadores-u.json): 38 instrucciones contrastadas con bytes y 11 reglas de estado.
- [Pruebas de cifrado](evidencia/auditoria-cifrado-offline-u.json): 72 comprobaciones del modelo con claves artificiales; no interoperabilidad ni auditoría formal.
- [Intercambio sintético](evidencia/verificacion-intercambio-u.json): consistencia del modelo, no validación integral de X25519 MIPS.
- Revisión local de enlaces, CSV y manifiesto de la distribución. Se documentan los resultados; no se incluye la herramienta utilizada para esas comprobaciones.

Los logs de un intento de conexión aportados posteriormente por el operador solo muestran timeout: no demuestran negociación, rechazo de identidad ni mensajes reales del C2. No se utilizan como prueba de interoperabilidad.

## Custodia pendiente y publicación

Mantener fuera del repositorio muestras, PCAP, consola cruda, seriales, credenciales y configuraciones de abonados. Los extractos publicados son análisis estáticos seleccionados, no toda la evidencia original. No se aconseja subir automáticamente muestras a servicios públicos; cualquier compartición requiere revisar alcance y política del propietario de la evidencia.

El archivo `MANIFEST-SHA256.txt` permite verificar los archivos de este paquete. Su hash no sustituye el de la muestra original ni acredita una firma de fabricante.
