# Identificación del equipo, firmware y cuentas

[← README](../README.md)

## Datos que no deben confundirse

| Dato del caso B | Procedencia | Límite |
| --- | --- | --- |
| ZX8102WT, HW 1.0.0.03.0, SW V12.2.44 | Datos de interfaz aportados por el operador | Identidad declarada; sin etiqueta/ficha independiente corroborada |
| Plataforma Formosa, `rtk,9607c`, MIPS interAptiv V2.0 | Consola recogida del equipo | Identifica plataforma/chipset, no fabricante comercial |
| `V4.0.0-240520`, compilación de mayo de 2024 | Versión observada en consola | No se ha reconciliado con el campo V12.2.44 de la interfaz |
| Vendor PON `OPTI`, referencias `RZ`, CWMP `Realtek` | Campos locales del equipo | Orientan una personalización; no autentican marca/modelo ni sustituyen firmware oficial |

Se retira la atribución comercial automática a ZTE. Tampoco se afirma que el equipo sea de otra marca solo por esas cadenas. Los números de serie y credenciales no se incluyen en este paquete público.

## Cuentas privilegiadas observadas

> 🟦 Consola del caso B; interpretación y límites del informe local de cuentas.

- `/etc/passwd` apunta a `/var/passwd`; la tabla efectiva es escribible en RAM. El enlace dentro de una raíz de solo lectura no vuelve confiables sus entradas.
- Se observaron `root`, dos registros `admin` idénticos y `rzfiber` con UID 0. También `nobody` con UID 0 y shell `/dev/null`, y `user` con UID 1.
- La configuración respalda el nombre `admin`, pero no explica por sí sola su duplicación. No son dos identidades con contraseñas distintas según la comparación local.
- No se recuperó una instrucción que demuestre que el atacante creó `rzfiber` o duplicó `admin`. La ausencia de cadenas tampoco demuestra que sean de fábrica.
- `rzfiber` tiene UID 0 y shell; no se probaron sus credenciales ni se demostró acceso mediante esa cuenta.

## Evidencia necesaria para cerrar

Obtener una imagen limpia compatible y verificable del proveedor real o una unidad de referencia confiable; analizar la generación de cuentas y comparar las salidas. Reconciliar campos de versión y confirmar aprovisionamiento antes de instalar firmware. Las cuentas del dispositivo ya comprometido y su copia local de respaldo no son referencias independientes.

No se publicó ningún hash de contraseña ni se modificaron cuentas para esta revisión. Origen en firmware, personalización del distribuidor y modificación posterior siguen siendo posibilidades no resueltas.
