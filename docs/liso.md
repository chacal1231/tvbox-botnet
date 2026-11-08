# `.liso` — bot DDoS con descubrimiento de C2 por TXT

[← README](../README.md)

| Caso | SHA-256 | Bytes |
| --- | --- | --- |
| A | `4063cdb6e7b6427a82ccf160fc3a13e645e3bc87428eb094aa12b6b32e8306e7` | 631912 |
| B | `0c0342c4f3d1a976e3344d2f1508ea298fc8e7c65a513d080b17eca76f3315ef` | 631912 |

Las dos muestras comparten tamaño y dominio descifrado, pero sus hashes son distintos, así que no puede afirmarse que sean el mismo ejecutable. **Todas las direcciones de código de este documento corresponden a la muestra B.**

## Caso A

> 🟦 Observado / 🟩 Estático · ⬜ informe original

- El proceso hijo (PID 25301) abre conexiones masivas a `104.233.223[.]169:80`.
- Conexión de control a `178.128.60[.]155:443`. Es una IP histórica; no se verificó si sigue vigente.
- Negocia TLS con la suite `TLS13-AES256-GCM-SHA384`, envía un registro inicial y procesa órdenes binarias.
- Referencia al socket UNIX abstracto `@fdrunkkkk@@@@5`.

## Caso B — configuración cifrada

> 🟩 Estático · caso B

La tabla de configuración tiene diez entradas:

| Dirección | Función |
| --- | --- |
| `0x40a2a4` | Inicializa la tabla |
| `0x40a12c` | Desbloquea una entrada |
| `0x40a1bc` | Lee una entrada |
| `0x40a174` | Vuelve a bloquear una entrada |
| `0x409cec` | Genera el flujo XOR |

El flujo XOR de `0x409cec` usa operaciones de 32 bits, sumas, rotaciones y realimentación. **No es ChaCha20 estándar.**

La entrada 1 está en VA `0x489340` (offset de archivo `0x89340`) y ocupa 14 bytes:

```text
07 84 b3 93 1b 42 73 73 36 e5 76 e0 36 31  →  jxk.imarapi.st
```

Otras cadenas recuperadas de la tabla:

- `father?`
- `libssl.so`
- `/proc/self/oom_score_adj`, `/proc/self/oom_adj`
- `-17`, `-1000`
- `PPid:`
- Dos listas de rutas/utilidades, cuya función no se asigna sin seguir sus llamadores.

Una reconstrucción matemática independiente descifró las diez entradas y volvió a cifrarlas para verificar que coinciden. Esa comprobación no ejecutó ni emuló instrucciones de la muestra.

## Flujo TXT → candidatos → IPv4 → conexión

> 🟩 Estático · caso B

1. El llamador obtiene la entrada 1 y llama a `0x408210` con el dominio.
2. Consulta registros **TXT** mediante DoH.
3. Recorre hasta 64 respuestas y valida tipo TXT, clase IN y límites.
4. Procesa cada cadena de longitud prefijada por separado; acepta entre 1 y 254 bytes y **no concatena** fragmentos.
5. Divide cada cadena por `|` y pasa los tokens al decodificador `0x409690`.
6. Elige un candidato mediante **`R mod N`**, donde `R` lo produce el generador `0x4072b0` y `N` es el número de candidatos.
7. Devuelve una IPv4 y conecta por **TCP/443**.

### Decodificador `0x409690`

- Recorre el token **hacia atrás** buscando un sufijo de **8 dígitos hexadecimales**. Admite `:` como separador y toma el primer `]` como final si existe.
- Convierte esos 8 dígitos en 4 bytes y aplica XOR por octeto con **`6b 9e 3f 2a`**.
- El resultado es una IPv4, aunque la presentación del token pueda parecer una IPv6.
- El recorrido no procesa el primer carácter del token. Por eso un token desnudo de exactamente 8 dígitos **no** equivale al mismo sufijo con prefijo o separador.

Ejemplo con datos sintéticos (red de documentación `192.0.2.0/24`):

```text
token   "[2001:db8::ab9e:3d20]"
sufijo  ab 9e 3d 20
XOR     6b 9e 3f 2a
IPv4    c0 00 02 0a  →  192.0.2.10
```

La reconstrucción textual del parser se describe arriba. El ejemplo sintético produce `192.0.2.10`, una dirección de documentación que no es un IOC observado. Esta edición no incluye una implementación ejecutable del decodificador. La equivalencia completa del modelo con todos los casos posibles sigue siendo un límite del análisis.

### Selección

- No se observó ordenación ni deduplicación: una dirección repetida puede ocupar más posiciones seleccionables. No se midió la distribución del generador.
- Capacidad de candidatos: 16 × el número de respuestas, con un máximo de 1024.
- No se afirma que la selección sea uniforme en sentido criptográfico.

## Resolución principal y respaldo

> 🟩 Estático · caso B

El ayudante `0x4078cc` envía `POST /dns-query` con `Content-Type: application/dns-message` sobre TLS/TCP 443. Prueba en orden ocho pares host/IP y avanza si uno falla:

| Proveedor | IP |
| --- | --- |
| `dns.google` | 8.8.8.8 · 8.8.4.4 |
| `cloudflare-dns.com` | 1.1.1.1 · 1.0.0.1 |
| `dns.quad9.net` | 9.9.9.9 · 149.112.112.112 |
| `dns.adguard-dns.com` | 94.140.14.14 · 94.140.15.15 |

Si DoH falla o no devuelve candidatos válidos, recurre a **DNS UDP a `8.8.8.8:53`** con la misma consulta TXT, el mismo decodificador y la misma selección. Hace como máximo 15 intentos: el contador empieza en 1 y el bucle termina al llegar a 16.

Consecuencias para la detección:

- **DoH** oculta el nombre consultado a los registros del DNS local.
- **El respaldo UDP** expone el nombre a un sensor en tránsito hacia 8.8.8.8. Solo aparecerá en los logs del DNS del ISP si ese resolvedor recibe la consulta, por ejemplo mediante redirección explícita.

No se demuestra que el respaldo se usara durante las capturas.

## Rotación del C2

> 🟩 Estático · caso B

- Puede obtener otra IP **sin actualizar el binario**: basta con que cambie el TXT.
- En cada nueva conexión puede elegir otro candidato del mismo conjunto.
- Libera los candidatos no elegidos: no los prueba en orden si falla la conexión TCP.
- No se observó un temporizador de cambio de C2 durante una sesión activa en esta ruta.
- No se observó uso del TTL para programar cambios en esta ruta.
- No se identificó una IP C2 estática de respaldo en las rutas examinadas.
- Si no obtiene ningún candidato, devuelve fallo y el llamador entra en su lógica de reintento.

No se consultó el TXT real. **`178.128.60.155` es una observación histórica del caso A**, no un resultado de esta revisión.

## Mapa de direcciones (muestra B)

| Dirección | Función |
| --- | --- |
| `0x406794–0x40681c` | Obtención del dominio y llamada al procesador TXT |
| `0x406854–0x4068c0` | Conversión a IPv4, puerto 443 y conexión |
| `0x4072b0` | Generador para elegir el candidato |
| `0x4078cc` | Consulta DoH |
| `0x408210` | Procesamiento TXT y ruta de respaldo |
| `0x409690` | Decodificación del candidato |
| `0x409cec` | Transformación de la tabla cifrada |
| `0x40a12c` / `0x40a1bc` / `0x40a174` | Desbloquear / leer / bloquear entrada |
| `0x40a2a4` | Inicialización de la tabla |

## Pendiente

- Catálogo de órdenes de `.liso`. **No** debe trasladarse aquí el despachador de `u`.
- Contenido actual e histórico del TXT.

## Conexión observada en el caso B

> 🟦 Observado en consola: conexión establecida a `185.173.39[.]100:443`, atribuida a procesos de `.liso`; las capturas incluyen procesos descendientes y posible herencia del socket. Es una observación histórica, distinta de `178.128.60[.]155` del caso A. No se descifró esa sesión ni se obtuvo una respuesta TXT real que la vincule al mecanismo reconstruido.

## Evidencia y reproducción local

- [Configuración descifrada](evidencia/configuracion-descifrada-liso.csv).
- [Resultados estructurados](evidencia/configuracion-y-txt-liso.json).
- [Informe de TXT y configuración](evidencia/informe-configuracion-txt-liso.txt).
- [Fragmentos de desensamblado](evidencia/evidencia-configuracion-txt-liso.txt).
