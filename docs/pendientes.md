# Pendientes, prioridad y evidencia de cierre

[← README](../README.md)

## Resuelto en la revisión 1.2

| Punto | Evidencia / alcance |
| --- | --- |
| MD5 y SHA-1 del caso B | Recalculados localmente; `iocs/hashes.csv`. No se inventan SHA-1 de muestras A ausentes |
| Herramientas y mapeos ELF | Versiones locales y segmentos en `metodologia.md` |
| Autenticación y contadores de u | Recorrido de error, inicio TX/RX y umbral `2^30`; informes y bytes en `docs/evidencia/`. No cierra una auditoría criptográfica formal |
| Clasificación de indicadores | C2, descarga, conexión sin clasificar, víctima posible y servicios legítimos diferenciados; sin suponer vigencia actual |
| Preparación del paquete público | Enlaces, CSV, manifiesto y ausencia de muestras; no sustituye control de custodia ni pruebas de producción |

## Trabajo pendiente

Prioridades: P1 afecta contención/recuperación o fiabilidad de detección; P2 completa reconstrucción y alcance; P3 profundiza relación de campañas. Son tareas propuestas, **no realizadas**. La asignación personal y los plazos corresponden al operador.

| ID | Prioridad | Pendiente | Evidencia necesaria para cerrarlo |
| --- | --- | --- | --- |
| P-01 | P1 | Corroborar fabricante, modelo y campos de versión | Etiqueta/inventario de origen confiable, proveedor y reconciliación V12.2.44 frente a V4.0.0-240520; no basta la consola comprometida |
| P-02 | P1 | Firmware y recuperación confiables | Imagen compatible, procedencia y checksum/firma; prueba de recuperación y aprovisionamiento en laboratorio |
| P-03 | P1 | Origen de rzfiber, admin duplicado y otros UID 0 | Generador de cuentas de imagen independiente y comparación con unidad limpia; sin publicar credenciales |
| P-04 | P1 | Efecto real de reset y limpieza de /var/config | Ensayo controlado por modelo/versión y modo de reset, con estado previo/posterior y plan de recuperación |
| P-05 | P1 | Desarrollar y validar reglas de detección | Diseño de reglas YARA/Suricata, compilación con versiones registradas y evaluación con PCAP sintéticos positivos/negativos y corpus limpio compatible; no incluidas en esta entrega |
| P-06 | P1 | Alcance en la red del ISP | Flujos/logs con ventana y cobertura; atribución a abonado antes de NAT; separar sospechosos y confirmados, no equipararlo al tamaño total de la botnet |
| P-07 | P2 | Revalidar caso A y completar custodia | Archivo original, sus hashes, muestras y fechas/responsables documentados; no inferir datos faltantes |
| P-08 | P2 | Cadena ENS de .ffaaxx | Muestra coincidente y referencias ENS → contrato/selector → respuesta → parser → conexión; primero análisis local, no consulta automática de nodos |
| P-09 | P2 | Catálogo de órdenes de .liso | Despachador/manejadores y condiciones ligados al SHA-256, sin trasladar órdenes de u |
| P-10 | P2 | Auditoría integral de autenticación/X25519 de u | Vectores y equivalencia de aritmética, validación de secretos degenerados, confirmación de claves y todas las ramas de error |
| P-11 | P2 | Generación aleatoria y concurrencia de u | Calidad del fallback de /dev/urandom y análisis de intercalados/reconexiones/contadores; no basta el modelo matemático |
| P-12 | P2 | Órdenes históricas y rotación efectiva | PCAP/telemetría ya capturados y, si existe legítimamente, material de sesión; tiempos y TXT históricos. No reconectar a C2 para producir evidencia |
| P-13 | P2 | TCP/42363 de .a y descargador A | Muestra A, referencias de bind/listen/accept y rutas de descarga; no extrapolar desde u |
| P-14 | P2 | Vector inicial y TV Box | Logs independientes, relación temporal, IP/MAC y evidencia de acceso; Telnet/ADB o cadenas GPON no lo prueban |
| P-15 | P2 | Impacto DDoS | Tasas/ventanas verificables y evidencia del destino; distinguir conexiones masivas de daño medido |
| P-16 | P3 | Familia/operador y relación de muestras | Coincidencias de código, identidad criptográfica, configuración y cronología con procedencia; un país de hosting o un nombre no atribuye actor |
| P-17 | P2 | Coordinación con terceros | Avisos preparados con evidencia mínima y titularidad histórica corroborada; no consta envío |

No se incorporan consultas DNS/ENS, conexiones al C2, escaneos externos o carga de muestras a terceros como tareas automáticas de publicación.
