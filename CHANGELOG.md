# Changelog

## 1.2 — 2026-11-08 · edición documental

- Entrega limitada a documentación y datos de análisis; se retiran scripts y reglas del paquete de publicación. Su desarrollo queda pendiente.

- Integra autenticación y contadores de `u`, manejo de etiquetas inválidas, umbral `2^30` y límites de las conclusiones.
- Completa MD5/SHA-1 del caso B, versiones locales, segmentos ELF y procedencia de evidencia.
- Añade la conexión observada B a `185.173.39.100:443`, sin afirmar descifrado de la sesión.
- Añade pendientes con prioridad y criterio de cierre; incorpora cuentas privilegiadas, versiones de firmware discordantes y fabricante por corroborar.
- Retira marcado de distribución no solicitado y atribución automática a ZTE.
- Corrige DNS externo frente a logs del ISP, tráfico de gestión frente a tránsito/NAT, Minecraft TCP y alcance de la limpieza.
- Separa destinos sin clasificar de C2 y bloqueos automáticos; contención aguas arriba y recuperación verificable.
- Documenta límites de detección: nombre DNS frente a QTYPE, representación de sockets abstractos y separación entre rasgos locales y caso A no revalidado. No incluye reglas ni afirma compilación de motores.
- Añade extractos estáticos seleccionados, manifiesto y validación documental. No se incluyen herramientas, muestras, PCAP, clientes C2 ni consola cruda.


## 1.1 — 2026-11-08

### Cambiado

- README reestructurado: TL;DR, MITRE ATT&CK, detección, remediación, línea de tiempo, notificaciones y pendientes.
- Análisis técnico movido a `docs/`.
- Tabla única de hashes: se eliminan los SHA-256 duplicados.
- IOC de red desactivados (`[.]`) en el README; los valores sin modificar están en `iocs/`.
- Etiquetas de evidencia indicadas una vez por sección.

### Añadido

- Caso B (ZX8102WT), con tres muestras:
  - `.liso`: tabla de configuración descifrada, flujo TXT → candidato → IPv4, decodificador (XOR `6b 9e 3f 2a`), DoH con 8 resolvedores y respaldo UDP a `8.8.8.8:53`.
  - `u`: canal X25519/ChaCha20 propio (no TLS), órdenes 0, 2–5, 16–19 y 22 métodos.
  - downloader: `GET /mips` → `.liso`.
- `iocs/iocs.csv`, `iocs/hashes.csv`, reglas YARA y Suricata.

### Corregido

- Los candidatos TXT de `.liso` son IPv4 ofuscadas en hexadecimal, no IPv4 decimales.
- El puerto 443 de `u` no implica TLS/HTTPS.
- En `u` se distinguen el ID de orden y el ID de método.
- El método 33 no se atribuye a Rapid Reset (no hay RST_STREAM).

## 1.0 — 2026-11-02

- Informe original del caso A (ZX8202DWT): `.ffaaxx`, `.liso`, `.a`, `hmm`, persistencia en `/var/config`.
