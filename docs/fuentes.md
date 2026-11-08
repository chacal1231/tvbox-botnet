# Fuentes y matriz de afirmaciones

[← README](../README.md)

## Evidencia de la investigación

| Afirmación | Caso / procedencia | Evidencia publicable | Límite |
| --- | --- | --- | --- |
| Procesos A, 178.128.60.155, TCP/42363 y script persistente | Informe original A | README y `ffaaxx-a.md` preservan la procedencia | Binarios/archivo A no disponibles para revalidación |
| Tabla, dominio y parser TXT | Muestra `.liso` B | [Informe](evidencia/informe-configuracion-txt-liso.txt), [datos](evidencia/configuracion-y-txt-liso.json), [VA](evidencia/evidencia-configuracion-txt-liso.txt) | Reconstrucción estática, sin TXT real |
| Conexión B a 185.173.39.100:443 | Consola B: evidencia-02/05 y ampliación local | Resumen en `liso.md` e IOC | Consola cruda privada; sin PCAP ni descifrado |
| Cifrado/intercambio de u | Segmentos reconstruidos B | [Intercambio](evidencia/intercambio-y-alternativas-c2-u.txt), [cifrado](evidencia/puerto-y-cifrado-u.txt) | No interoperabilidad; precisa leer la ampliación de autenticación |
| Errores, sesión y contadores | Muestra u B | [Informe](evidencia/autenticacion-y-contadores-u.txt), [bytes](evidencia/evidencia-autenticacion-contadores-u.txt), [comprobaciones](evidencia/autenticacion-contadores-u.json) | 38 instrucciones y 11 reglas, no auditoría total |
| Métodos/propagación | Muestra u B | [Métodos](evidencia/metodos-orden-0-u.csv), [selectores](evidencia/modulos-propagacion-orden-18-u.csv), [HTTP/TLS/SSH](evidencia/ampliacion-http-tls-ssh-u.txt) | Capacidades, no explotación histórica demostrada |
| Cuentas y firmware | Consola B e informe local de cuentas/modelo | Resumen redactado en `firmware-cuentas.md` | Referencia oficial limpia no obtenida; sin seriales/contraseñas publicados |

La publicación de extractos no sustituye el depósito privado de evidencia. Los hashes A siguen siendo valores declarados en el informe original. Las correcciones más recientes prevalecen sobre apartados que describen un pendiente de una revisión anterior.

## Documentación externa

Estas fuentes explican herramientas, estándares o metodología; no atribuyen las muestras a una familia ni confirman su actividad.

- [MITRE ATT&CK T1037.004 — RC Scripts](https://attack.mitre.org/techniques/T1037/004/): referencia para el mapeo de persistencia por script de arranque.
- [MITRE ATT&CK T1568 — Dynamic Resolution](https://attack.mitre.org/techniques/T1568/): marco para interpretar selección dinámica de destinos.
- [MITRE ATT&CK T1572 — Protocol Tunneling](https://attack.mitre.org/techniques/T1572/): incluye el uso de DoH en resolución; no demuestra órdenes dentro de DNS en este caso.
- [Suricata 7.0.15, payload keywords](https://docs.suricata.io/en/suricata-7.0.15/rules/payload-keywords.html): `content`, `bsize` y límites de buffers.
- [Suricata, DNS keywords](https://docs.suricata.io/en/latest/rules/dns-keywords.html): diferencias entre nombre y tipo de consulta; verificar versión del motor objetivo.
- [YARA, sintaxis de reglas](https://yara.readthedocs.io/en/latest/writingrules.html) y [módulo hash](https://yara.readthedocs.io/en/v4.2.0/modules/hash.html): referencia de condiciones, no certificado de que las reglas se compilaran aquí.
- [UPX, propiedades LZMA](https://github.com/upx/upx/blob/devel/src/compress/compress_lzma.cpp): interpretación de propiedades del desempaquetado estático.
- [CISA, incident response playbook](https://www.cisa.gov/sites/default/files/publications/Cybersecurity_Incident_Vulnerability_Response_Playbooks_508C.pdf): aislamiento y filtrado como contención.

Consultadas para la edición del 2026-11-08. No se consultaron indicadores ni se enviaron notificaciones a partir de estos enlaces.
