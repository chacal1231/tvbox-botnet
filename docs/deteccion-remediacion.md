# Detección, contención y recuperación

[← README](../README.md)

## 1. Identificar el origen correcto

Separar la IP/VLAN de gestión de la ONU, el tráfico originado por el sistema de la ONU y el tráfico que solo atraviesa el equipo desde la LAN. Un flujo hacia un resolvedor público puede ser legítimo y no demuestra infección. En modo router/NAT pueden compartir dirección; después de CGNAT una IP pública puede representar muchos abonados.

Usar interfaces de ingreso, VLAN, inventario de abonados y registros de NAT con tiempo y puertos. Medir por separado equipos sospechosos, equipos confirmados y cobertura del muestreo. NetFlow/sFlow no contiene el texto cifrado de las órdenes ni permite contar la botnet mundial.

## 2. Búsquedas de flujos

Las siguientes condiciones son **pseudoconsultas**, no comandos garantizados para una versión concreta de Akvorado/ClickHouse. Traducir nombres de campos, tipos IP, prefijos, muestreo y ventanas temporales al esquema real.

```text
origen = gestión de ONU identificada
ventana = intervalo UTC de investigación

Señal DoH potencial:
  TCP/443 hacia {8.8.8.8,8.8.4.4,1.1.1.1,1.0.0.1,
                9.9.9.9,149.112.112.112,94.140.14.14,94.140.15.15}

C2 estático de u:        143.244.144.70:443/TCP
C2 histórico de liso A:  178.128.60.155:443/TCP
Conexión de liso B:      185.173.39.100:443/TCP (sesión no descifrada)
Descarga incorporada:   170.205.28.207:80/TCP

Destinos sin clasificar: 87.58.159.3:443,43.157.198.137:443,159.223.66.236:443
  -> investigar; no promover automáticamente a C2
```

Agrupar por abonado/origen, destino, puerto, protocolo y tiempo. Registrar primera/última observación, paquetes, bytes y cobertura. Correlacionar varias señales: una coincidencia con un servicio público no basta.

## 3. DNS y captura pasiva

El respaldo de `.liso` consulta directamente `8.8.8.8:53/UDP`. Los logs del resolvedor del ISP no lo mostrarán salvo que la red le entregue esas consultas. En un sensor que vea el tránsito, buscar el nombre exacto `jxk.imarapi.st` y confirmar QTYPE TXT. Una futura detección debe distinguir el nombre de consulta del tipo de registro: no etiquetar toda coincidencia del dominio como TXT sin revisar QTYPE.

DoH oculta el nombre a un sensor sin descifrado de HTTPS. Una RPZ local no afecta DoH directo a otro proveedor. Capturar pasivamente solo tráfico autorizado, sin reabrir conexiones bloqueadas. Un PCAP puede aportar framing, tiempos y destinos; no promete descifrar órdenes sin el material criptográfico de esa sesión. La clave de configuración de `u` no sustituye la clave de sesión.

## 4. Exposición y tráfico de ataque

En el inventario autorizado revisar Telnet/23 **y mapeos como TCP/65000**, gestión web y escucha TCP/42363. No inferir qué servicio escucha solo por su número. Consultar primero ACL, NAT, inventario y logs; cualquier escaneo debe estar acotado al alcance aprobado.

Buscar ráfagas correlacionadas de SYN, UDP, GRE e ICMP. Entre las capacidades estáticas de `u` figuran UDP/1194 y UDP/51820, Minecraft **TCP/25565** y tráfico tipo FiveM sobre TCP/30120. Los puertos aislados son señales débiles, no firmas. Las tasas de flujos muestreados requieren corrección y contexto antes de afirmar impacto.

## 5. Contener fuera del equipo comprometido

Priorizar cuarentena o una política de salida de la gestión de la ONU que permita únicamente los destinos y servicios realmente necesarios. Aplicarla en un dispositivo de red confiable, con plan de retorno y acceso de gestión preservado. Cubrir IPv4/IPv6 y comprobar que también se filtra el tráfico reenviado donde corresponda.

Los destinos conocidos permiten bloqueos acotados temporales, usando el papel y la evidencia de `iocs/iocs.csv`. `.liso` puede obtener nuevas IP por TXT y `u` admite cambiar destino al iniciar; una lista estática no basta. No bloquear como C2 `159.223.66.236`, RPC/DoH públicos o destinos sin clasificar sin evidencia adicional. Se puede restringir el acceso a servicios legítimos **desde el origen comprometido** como parte de su cuarentena, sin atribuirlos al atacante.

Después de aplicar los filtros, terminar en el firewall las sesiones afectadas que ya estaban establecidas y verificar nuevos intentos denegados. Una regla colocada después de aceptar `established/related` puede no cortar el canal existente. El procedimiento concreto depende del fabricante de la red y no se incluye como una receta universal.

El DNS autorizado puede aplicar una política al dominio de descubrimiento, pero también debe controlar DNS externo y DoH desde el origen en cuarentena. No recomendar un bloqueo global de resolvedores para todos los abonados sin estudiar el servicio.

Contener el C2 **no elimina** persistencia ni garantiza que cese inmediatamente un trabajo ya iniciado. El aislamiento también debe impedir ataques salientes y movimiento lateral. Esta estrategia es coherente con el [playbook de respuesta de CISA](https://www.cisa.gov/sites/default/files/publications/Cybersecurity_Incident_Vulnerability_Response_Playbooks_508C.pdf).

## 6. Recolección antes de limpiar

Conservar evidencia de gestión autorizada: procesos y parentesco, enlaces `/proc/PID/exe`, sockets, script persistente, listado de archivos y hashes. Anotar tiempo UTC de la estación, reloj defectuoso del equipo, comandos usados y responsable. Leer binarios como datos; no ejecutarlos para identificarlos.

La extracción desde `/proc/PID/exe` puede recuperar un ejecutable eliminado que siga activo. Si se escribe una copia en la ONU se altera el estado: documentar ruta, tamaño y hash, y transferirla a almacenamiento controlado. No publicar transcripciones con credenciales, seriales, IP privadas o configuración de abonados. Mantener la captura cruda separada de los extractos públicos.

No se ofrece aquí un bloque de borrado masivo: los nombres y PID son históricos y los directorios pueden tener otros usos legítimos.

## 7. Recuperación

La opción preferida ante compromiso con privilegios amplios es recuperar desde firmware y configuración confiables del proveedor **corroborado** para ese modelo, hardware y aprovisionamiento. No se obtuvo un firmware oficial compatible; no se recomienda instalar una imagen por coincidencia parcial del nombre ZX.

La limpieza manual provisional requiere identificar los ejecutables activos y todas sus rutas de arranque, preservar copias y retirar únicamente las modificaciones confirmadas. Antes de sustituir `/var/config/run_customized_sdk.sh`, comparar el arranque y la copia legítima de **esa versión**. Que estuviera vacía en el caso A no convierte todo script no vacío en malicioso. Una limpieza parcial no certifica integridad.

Restringir gestión WAN/LAN a redes autorizadas; usar credenciales únicas, retirar accesos innecesarios y revisar las cuentas privilegiadas. No borrar `rzfiber` ni las líneas `admin` automáticamente sin entender la generación de cuentas y el acceso de recuperación. Revisar la LAN y aislar un dispositivo sospechoso cuando haya evidencia; el TV Box sigue siendo una hipótesis.

## 8. Criterios de verificación y cierre

- Arranque y configuración coinciden con una referencia confiable compatible.
- No reaparecen procesos, archivos ni lanzamientos maliciosos tras reinicio controlado.
- No hay reconexiones o tráfico de ataque desde el origen durante una ventana definida por el operador; registrar cobertura, no afirmar ausencia global.
- Se conservan accesos legítimos de gestión/servicio y se comprueban ACL IPv4/IPv6.
- La unidad vuelve a producción solo tras validar recuperación y exposición. Registrar posibles afectados adicionales con un criterio consistente.

Un reset de fábrica se prueba primero en una unidad de laboratorio compatible, con respaldo de aprovisionamiento y recuperación. Un archivo testigo puede mostrar persistencia de ese archivo, pero no demuestra por sí solo eliminación de todas las modificaciones ni mide todos los modos de reset.

## 9. Detección pendiente de implementación

Esta edición no incluye reglas YARA/Suricata ni scripts. Se documentan señales y límites para su desarrollo posterior; véase [validación](validacion.md). [NCSC](https://www.ncsc.gov.uk/guidance/protective-dns-for-private-sector) describe la necesidad de controlar resolvedores y DNS cifrado conforme a la política de la red.
