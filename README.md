# Identificación de Bloqueos en SQL Server

Guía de consultas SQL Server para monitorear usuarios conectados, identificar sesiones bloqueadas y bloqueadoras, e interpretar los tipos de espera.

---

## Contenido

1. [Usuarios conectados](#1-usuarios-conectados)
2. [Identificación de bloqueos](#2-identificación-de-bloqueos)
3. [Interpretación de resultados](#3-interpretación-de-resultados)
4. [Tipos de espera](#4-tipos-de-espera)
5. [Finalizar una sesión con KILL](#5-finalizar-una-sesión-con-kill)
6. [Recomendaciones](#6-recomendaciones)

---

# 1. Usuarios conectados

Esta consulta permite conocer la cantidad de usuarios conectados actualmente al sistema desde conexiones cuyo nombre contiene `CTX`.

```sql
/* USUARIOS CONECTADOS */

SELECT
    COUNT(DISTINCT s.login_name) AS USUARIOS_CONECTADOS
FROM sys.dm_exec_sessions s
WHERE s.host_name LIKE '%CTX%';
```

### Resultado esperado

La consulta devuelve un único valor:

| Campo | Descripción |
|---|---|
| `USUARIOS_CONECTADOS` | Número de usuarios distintos conectados actualmente desde equipos `CTX`. |

Ejemplo:

```text
USUARIOS_CONECTADOS
-------------------
47
```

Esto indica que existen **47 usuarios distintos conectados**.

> **Nota:** La consulta cuenta usuarios distintos mediante `login_name`. Un mismo usuario puede tener más de una sesión abierta.

---

# 2. Identificación de bloqueos

La siguiente consulta permite identificar si existen sesiones bloqueadas actualmente en SQL Server y obtener información tanto del proceso bloqueado como del proceso que está provocando el bloqueo.

```sql
SELECT
    'RUBA' AS SERVIDOR,
    DB_NAME(r.database_id) AS BaseDatos,
    DATEDIFF(MINUTE, r.start_time, GETDATE()) AS MINUTOS,

    r.session_id AS SPID_Bloqueado,
    s.login_name AS Login_Bloqueado,
    s.host_name AS Host_Bloqueado,
    s.program_name AS Programa_Bloqueado,

    r.blocking_session_id AS SPID_Bloqueador,
    sb.login_name AS Login_Bloqueador,
    sb.host_name AS Host_Bloqueador,
    sb.program_name AS Programa_Bloqueador,

    r.status AS Estado,
    r.wait_type AS Tipo_Espera,
    r.wait_time AS Tiempo_Espera_ms,
    r.wait_resource AS Recurso_Espera,

    r.start_time AS Inicio_Query,
    DATEDIFF(SECOND, r.start_time, GETDATE()) AS Duracion_Segundos,

    SUBSTRING(
        tb.text,
        (r.statement_start_offset / 2) + 1,
        (
            (
                CASE r.statement_end_offset
                    WHEN -1 THEN DATALENGTH(tb.text)
                    ELSE r.statement_end_offset
                END
                - r.statement_start_offset
            ) / 2
        ) + 1
    ) AS Query_Bloqueada,

    tq.text AS Query_Bloqueador

FROM sys.dm_exec_requests r

INNER JOIN sys.dm_exec_sessions s
    ON r.session_id = s.session_id

LEFT JOIN sys.dm_exec_sessions sb
    ON r.blocking_session_id = sb.session_id

LEFT JOIN sys.dm_exec_connections cb
    ON r.blocking_session_id = cb.session_id

CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) tb

OUTER APPLY sys.dm_exec_sql_text(cb.most_recent_sql_handle) tq

WHERE r.blocking_session_id <> 0

ORDER BY r.wait_time DESC;
```

### ¿Qué significa el resultado?

Si la consulta **no devuelve registros**, no existen solicitudes activas bloqueadas por otra sesión en ese momento.

Si devuelve registros, existe al menos una sesión esperando a que otra sesión libere un recurso.

---

# 3. Interpretación de resultados

| Campo | Descripción |
|---|---|
| `SERVIDOR` | Servidor donde se realizó la consulta. |
| `BaseDatos` | Base de datos donde se encuentra la solicitud bloqueada. |
| `MINUTOS` | Minutos transcurridos desde el inicio de la solicitud. |
| `SPID_Bloqueado` | Identificador de la sesión que está esperando. |
| `Login_Bloqueado` | Usuario de la sesión bloqueada. |
| `Host_Bloqueado` | Equipo desde donde se originó la sesión bloqueada. |
| `Programa_Bloqueado` | Aplicación utilizada por la sesión bloqueada. |
| `SPID_Bloqueador` | Identificador de la sesión que está provocando el bloqueo. |
| `Login_Bloqueador` | Usuario correspondiente a la sesión bloqueadora. |
| `Host_Bloqueador` | Equipo desde donde se originó el bloqueo. |
| `Programa_Bloqueador` | Aplicación utilizada por la sesión bloqueadora. |
| `Estado` | Estado actual de la solicitud. |
| `Tipo_Espera` | Tipo de espera que está experimentando la sesión. |
| `Tiempo_Espera_ms` | Tiempo de la espera actual en milisegundos. |
| `Recurso_Espera` | Recurso sobre el cual está esperando SQL Server. |
| `Inicio_Query` | Fecha y hora de inicio de la solicitud. |
| `Duracion_Segundos` | Tiempo transcurrido desde el inicio de la solicitud. |
| `Query_Bloqueada` | Instrucción SQL que se encuentra bloqueada. |
| `Query_Bloqueador` | Última instrucción conocida ejecutada por la sesión bloqueadora. |

### Ejemplo

```text
SPID_Bloqueado  : 184
Host_Bloqueado  : SRVCORP-CTX01

SPID_Bloqueador : 177
Host_Bloqueador : SRVCORP-CTX02

Tipo_Espera     : LCK_M_X
```

En este ejemplo:

**SPID 184** se encuentra bloqueado y está esperando al **SPID 177**.

---

# 4. Tipos de espera

El campo `Tipo_Espera` permite identificar qué tipo de recurso está esperando la sesión.

Cuando el valor comienza con:

```text
LCK_M_
```

la espera está relacionada con un **bloqueo (LOCK)**.

| Tipo de espera | Lock | Significado | Ejemplo típico |
|---|---|---|---|
| `LCK_M_S` | Shared | Lectura | Un `SELECT` quiere leer un recurso bloqueado por otro proceso. |
| `LCK_M_X` | Exclusive | Escritura exclusiva | Un `UPDATE`, `DELETE` o `INSERT` necesita modificar un recurso bloqueado. |
| `LCK_M_U` | Update | Actualización | SQL Server está esperando obtener un bloqueo de actualización. |
| `LCK_M_IS` | Intent Shared | Intención de lectura | Se requieren bloqueos compartidos en niveles inferiores. |
| `LCK_M_IX` | Intent Exclusive | Intención de escritura | Se requieren bloqueos exclusivos en filas o páginas inferiores. |
| `LCK_M_SIX` | Shared + Intent Exclusive | Lectura + modificación | Existe lectura compartida con intención de modificar registros. |
| `LCK_M_SCH_S` | Schema Stability | Estabilidad de esquema | Consultas, compilaciones o acceso a metadatos. |
| `LCK_M_SCH_M` | Schema Modification | Modificación de esquema | `ALTER TABLE`, `CREATE INDEX`, `DROP INDEX`, etc. |
| `LCK_M_BU` | Bulk Update | Carga masiva | Operaciones como `BULK INSERT`. |
| `LCK_M_RS_S` | RangeS-S | Rango compartido | Protección de rangos normalmente asociada a `SERIALIZABLE`. |
| `LCK_M_RS_U` | RangeS-U | Rango para actualización | Lectura de un rango que posteriormente puede actualizarse. |
| `LCK_M_RX_S` | RangeI-S | Inserción / rango | Validación de un rango antes de realizar una inserción. |
| `LCK_M_RX_U` | RangeI-U | Rango + actualización | Operaciones de actualización sobre rangos. |
| `LCK_M_RX_X` | RangeI-X | Rango exclusivo | Modificaciones exclusivas sobre un rango. |

---

# 5. Finalizar una sesión con KILL

Una vez identificado el `SPID_Bloqueador`, SQL Server permite finalizar esa sesión utilizando el comando `KILL`.

```sql
KILL <SPID>;
```

Por ejemplo, si la consulta de bloqueos muestra que el proceso bloqueador es el SPID `177`:

```sql
KILL 177;
```

## ¿Qué hace KILL?

`KILL` finaliza la sesión indicada.

Si la sesión tenía una transacción abierta, SQL Server debe realizar un **ROLLBACK** para deshacer los cambios que todavía no habían sido confirmados.

Por este motivo, ejecutar `KILL` **no significa necesariamente que el bloqueo desaparecerá inmediatamente**.

Dependiendo de la cantidad de información modificada por la transacción, el proceso de `ROLLBACK` puede tardar desde unos segundos hasta varios minutos.

---

##  Antes de ejecutar KILL

Antes de finalizar una sesión se debe revisar la información obtenida en la consulta de bloqueos, principalmente:

- `SPID_Bloqueador`
- `Login_Bloqueador`
- `Host_Bloqueador`
- `Programa_Bloqueador`
- `Query_Bloqueador`
- Tiempo que lleva activo el proceso
- Cantidad de sesiones que está bloqueando

No se recomienda ejecutar `KILL` únicamente porque una sesión aparece como bloqueadora.

Un bloqueo puede ser temporal y formar parte del funcionamiento normal de SQL Server.

---

## Ejecutar KILL

Una vez confirmado que la sesión debe ser finalizada:

```sql
KILL 177;
```

Donde `177` corresponde al `SPID_Bloqueador` identificado previamente.

>  **Importante:** Verificar cuidadosamente el SPID antes de ejecutar el comando.

---

## ¿Cuándo considerar un KILL?

Puede considerarse finalizar una sesión cuando:

- Mantiene un bloqueo durante un periodo anormalmente largo.
- Está bloqueando a múltiples usuarios.
- La aplicación que originó la sesión ya no responde.
- Se identificó una transacción abierta que no está avanzando.
- Se confirmó que cancelar la operación no afectará un proceso crítico.

>  **Advertencia:** `KILL` debe utilizarse como una acción correctiva y no como la solución habitual a los bloqueos. Si el mismo problema ocurre constantemente, se debe investigar la causa del bloqueo.

---

# 6. Recomendaciones

La existencia de un bloqueo **no significa automáticamente que exista un problema**.

Los bloqueos forman parte del funcionamiento normal de SQL Server y permiten mantener la consistencia de la información.

Se recomienda investigar cuando:

- El bloqueo permanece durante un periodo prolongado.
- Existen varios usuarios bloqueados por el mismo SPID.
- El tiempo de espera continúa aumentando.
- Los usuarios reportan lentitud.
- La aplicación deja de responder.
- Una transacción permanece abierta durante demasiado tiempo.

> **Importante:** No ejecutar `KILL` sobre una sesión únicamente porque aparece como bloqueadora. Primero se debe identificar la operación que está realizando y evaluar las consecuencias de cancelar la transacción.

---

# IMPORTANTE

Antes de ejecutar cualquiera de las consultas o acciones descritas en esta guía, es indispensable **verificar que se está trabajando en el servidor y ambiente correctos**.

Especialmente antes de realizar acciones como `KILL`, se debe confirmar cuidadosamente:

- El servidor al que se encuentra conectado.
- El ambiente en el que se está trabajando (Pruebas o Producción).
- El `SPID` que se pretende finalizar.
- El usuario, equipo y aplicación asociados a la sesión.
- La consulta o proceso que se encuentra ejecutando.
- El posible impacto de finalizar dicha sesión.

>  **ADVERTENCIA:** La ejecución de comandos administrativos como `KILL` puede interrumpir procesos en ejecución, provocar la reversión (`ROLLBACK`) de transacciones y afectar temporalmente la operación de otros usuarios o aplicaciones.

**Antes de realizar cualquier acción que pueda afectar la operación del servidor, se deberá validar el diagnóstico y confirmar el procedimiento con el responsable o superior correspondiente.**

La información obtenida mediante estas consultas debe utilizarse primero para **identificar y analizar el problema**. No se debe ejecutar una acción correctiva hasta tener certeza sobre el proceso afectado y las posibles consecuencias.

> **Primero diagnosticar, después validar y finalmente actuar.**
