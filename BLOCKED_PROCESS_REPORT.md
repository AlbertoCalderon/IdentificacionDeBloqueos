# Análisis de Blocked Process Report en SQL Server

Esta guía permite consultar y analizar eventos `blocked_process_report` registrados mediante **Extended Events** en SQL Server.

El objetivo es identificar bloqueos que permanecieron activos durante un periodo significativo y conocer:

- Cuándo ocurrió el bloqueo.
- Cuánto tiempo llevaba esperando el proceso.
- Qué sesión estaba bloqueada.
- Qué sesión estaba provocando el bloqueo.
- Qué usuario y equipo originaron las sesiones.
- Qué consultas estaban involucradas.
- Qué recurso estaba esperando el proceso.
- Qué tipo de espera se presentó.

---

## 📋 Contenido

1. [¿Qué es un Blocked Process Report?](#1-qué-es-un-blocked-process-report)
2. [Diferencia entre bloqueo, Blocked Process Report y Deadlock](#2-diferencia-entre-bloqueo-blocked-process-report-y-deadlock)
3. [Consultar Blocked Process Report](#3-consultar-blocked-process-report)
4. [Buscar bloqueos por horario](#4-buscar-bloqueos-por-horario)
5. [Interpretar el XML](#5-interpretar-el-xml)
6. [Identificar el proceso bloqueado](#6-identificar-el-proceso-bloqueado)
7. [Identificar el proceso bloqueador](#7-identificar-el-proceso-bloqueador)
8. [Interpretar el tipo de espera](#8-interpretar-el-tipo-de-espera)
9. [¿Qué hacer después de encontrar un bloqueo?](#9-qué-hacer-después-de-encontrar-un-bloqueo)

---

# 1. ¿Qué es un Blocked Process Report?

Un `blocked_process_report` es un evento generado por SQL Server cuando una sesión permanece bloqueada durante un tiempo mayor o igual al **Blocked Process Threshold** configurado en el servidor.

Ejemplo:

```text
SPID 177
   │
   │ Mantiene un recurso bloqueado
   ▼
┌───────────────────┐
│      RECURSO      │
└───────────────────┘
          ▲
          │ Esperando
          │
      SPID 184
```

En este ejemplo:

- `SPID 177` es el **bloqueador**.
- `SPID 184` es el **bloqueado**.

Si el SPID 184 permanece esperando durante el tiempo configurado como umbral, SQL Server puede generar un evento:

```text
blocked_process_report
```

> Un `blocked_process_report` es evidencia de que existió un bloqueo suficientemente prolongado para alcanzar el umbral configurado. No significa necesariamente que el bloqueo continúe activo al momento de consultar el historial.

---

# 2. Diferencia entre bloqueo, Blocked Process Report y Deadlock

Es importante no confundir estos conceptos.

| Situación | Bloqueo | Blocked Process Report | Deadlock |
|---|---|---|---|
| Una sesión espera a otra | Sí | Sí | Sí |
| Requiere dependencia circular | No | No | Sí |
| SQL Server selecciona una víctima | No | No | Sí |
| Se genera por superar un umbral | No necesariamente | Sí | No |
| SQL Server cancela automáticamente una transacción | No | No | Sí |
| Puede desaparecer sin intervención | Sí | Sí | SQL Server rompe el ciclo |
| Sirve para análisis histórico | No por sí solo | Sí | Sí |

### Bloqueo normal

```text
SPID 177
    │
    ▼
 Recurso
    ▲
    │
SPID 184 espera
```

El SPID 184 puede continuar cuando el SPID 177 libere el recurso.

### Blocked Process Report

```text
SPID 177
    │
    ▼
 Recurso
    ▲
    │
SPID 184 espera
    │
    │ Supera el umbral configurado
    ▼
blocked_process_report
```

SQL Server registra información del bloqueo para poder analizarlo posteriormente.

### Deadlock

```text
SPID 177 ───── espera ─────► SPID 184
    ▲                           │
    │                           │
    └──────── espera ───────────┘

              DEADLOCK
```

Aquí existe una dependencia circular y SQL Server debe seleccionar una víctima.

---

# 3. Consultar Blocked Process Report

La siguiente consulta permite recuperar los eventos `blocked_process_report` almacenados en los archivos `.xel` de Extended Events.

```sql
SELECT
    DATEADD(
        HOUR,
        DATEDIFF(HOUR, GETUTCDATE(), GETDATE()),
        x.event_data.value('(event/@timestamp)[1]', 'datetime2')
    ) AS FechaHora,

    x.event_data.query(
        '(event/data[@name="blocked_process"]/value/blocked-process-report)[1]'
    ) AS BlockedProcessXML

FROM
(
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file(
        'E:\BlockedProcess_XE_DEV\DEV_BlockMonitor*.xel',
        NULL,
        NULL,
        NULL
    )
) AS x

WHERE
    x.event_data.value(
        '(event/@name)[1]',
        'varchar(100)'
    ) = 'blocked_process_report'

ORDER BY FechaHora DESC;
```

## Resultado

La consulta devuelve:

| Campo | Descripción |
|---|---|
| `FechaHora` | Fecha y hora en que SQL Server registró el evento. |
| `BlockedProcessXML` | XML que contiene la información del proceso bloqueado y del bloqueador. |

> La ruta de los archivos `.xel` depende del servidor y de la configuración de la sesión de Extended Events.

---

# 4. Buscar bloqueos por horario

Cuando un usuario reporta lentitud o que un proceso dejó de responder, es recomendable consultar únicamente el horario en el que ocurrió el incidente.

Por ejemplo, para consultar eventos ocurridos el **19 de agosto de 2026 entre las 08:00 AM y las 09:00 AM**:

```sql
SELECT
    DATEADD(
        HOUR,
        DATEDIFF(HOUR, GETUTCDATE(), GETDATE()),
        x.event_data.value('(event/@timestamp)[1]', 'datetime2')
    ) AS FechaHora,

    x.event_data.query(
        '(event/data[@name="blocked_process"]/value/blocked-process-report)[1]'
    ) AS BlockedProcessXML

FROM
(
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file(
        'E:\BlockedProcess_XE_DEV\DEV_BlockMonitor*.xel',
        NULL,
        NULL,
        NULL
    )
) AS x

WHERE
    x.event_data.value(
        '(event/@name)[1]',
        'varchar(100)'
    ) = 'blocked_process_report'

    AND DATEADD(
        HOUR,
        DATEDIFF(HOUR, GETUTCDATE(), GETDATE()),
        x.event_data.value('(event/@timestamp)[1]', 'datetime2')
    ) >= '2026-08-19T08:00:00'

    AND DATEADD(
        HOUR,
        DATEDIFF(HOUR, GETUTCDATE(), GETDATE()),
        x.event_data.value('(event/@timestamp)[1]', 'datetime2')
    ) < '2026-08-19T09:00:00'

ORDER BY FechaHora DESC;
```

El rango utilizado es:

```text
08:00:00 <= FechaHora < 09:00:00
```

Esto permite concentrar el análisis únicamente en el periodo reportado.

---

# 5. Interpretar el XML

El campo:

```text
BlockedProcessXML
```

contiene información detallada sobre las sesiones involucradas.

La estructura principal puede visualizarse de forma simplificada como:

```text
blocked-process-report
│
├── blocked-process
│       └── process
│
└── blocking-process
        └── process
```

Las dos secciones más importantes son:

### `blocked-process`

Contiene la información de la sesión que **no pudo continuar porque estaba esperando un recurso**.

### `blocking-process`

Contiene la información de la sesión que **mantenía el recurso que impedía continuar al proceso bloqueado**.

Dentro de los nodos `process` pueden encontrarse datos como:

```text
spid
waittime
waitresource
lockMode
hostname
loginname
clientapp
currentdb
transactionname
inputbuf
```

---

# 6. Identificar el proceso bloqueado

Dentro del XML se debe localizar:

```xml
<blocked-process>
```

Dentro de esta sección se encuentra el proceso bloqueado.

Ejemplo simplificado:

```xml
<blocked-process>
    <process
        spid="184"
        waittime="15000"
        waitresource="KEY: ..."
        lockMode="X"
        hostname="SRVCORP-CTX01"
        loginname="usuario"
        clientapp="Aplicacion">
    </process>
</blocked-process>
```

En este ejemplo:

```text
SPID bloqueado : 184
Tiempo espera  : 15000 ms
Lock solicitado: X
```

El proceso está esperando poder continuar con su operación.

---

# 7. Identificar el proceso bloqueador

Posteriormente se debe localizar:

```xml
<blocking-process>
```

Ejemplo:

```xml
<blocking-process>
    <process
        spid="177"
        hostname="SRVCORP-CTX02"
        loginname="usuario"
        clientapp="Aplicacion">
    </process>
</blocking-process>
```

En este caso:

```text
SPID bloqueado  : 184
        │
        │ espera un recurso
        ▼
SPID bloqueador : 177
```

Se deben revisar principalmente los siguientes datos del bloqueador:

| Campo | Descripción |
|---|---|
| `spid` | Identificador de la sesión bloqueadora. |
| `hostname` | Equipo que originó la conexión. |
| `loginname` | Usuario conectado. |
| `clientapp` | Aplicación que abrió la conexión. |
| `currentdb` | Base de datos involucrada. |
| `transactionname` | Transacción relacionada con la sesión. |
| `inputbuf` | Última instrucción o procedimiento registrado. |

> El proceso bloqueador no necesariamente está ejecutando una consulta en el momento en que se genera el reporte. Puede existir una transacción abierta que continúa reteniendo Locks.

---

# 8. Interpretar el tipo de espera

El XML permite identificar el recurso y el tipo de Lock solicitado por la sesión bloqueada.

Algunos valores comunes son:

| Tipo | Significado |
|---|---|
| `S` | Shared - Lectura |
| `X` | Exclusive - Escritura |
| `U` | Update - Actualización |
| `IS` | Intent Shared |
| `IX` | Intent Exclusive |
| `SIX` | Shared + Intent Exclusive |

En las consultas de monitoreo también pueden aparecer esperas como:

| Tipo_Espera | Significado |
|---|---|
| `LCK_M_S` | Esperando un Lock compartido. |
| `LCK_M_X` | Esperando un Lock exclusivo. |
| `LCK_M_U` | Esperando un Lock de actualización. |
| `LCK_M_IS` | Esperando un Intent Shared. |
| `LCK_M_IX` | Esperando un Intent Exclusive. |
| `LCK_M_SIX` | Esperando Shared + Intent Exclusive. |
| `LCK_M_SCH_S` | Esperando estabilidad del esquema. |
| `LCK_M_SCH_M` | Esperando modificar el esquema. |

Por ejemplo:

```text
waitresource = KEY: ...
lockMode     = X
```

indica que el proceso necesita obtener un **Lock exclusivo** sobre un recurso de tipo `KEY`, pero otro proceso está impidiendo que lo obtenga.

---

# 9. ¿Qué hacer después de encontrar un bloqueo?

Una vez identificado un `blocked_process_report`, se recomienda recopilar:

1. Fecha y hora del evento.
2. SPID bloqueado.
3. SPID bloqueador.
4. Tiempo de espera.
5. Usuario bloqueado.
6. Usuario bloqueador.
7. Host bloqueado.
8. Host bloqueador.
9. Aplicación de ambas sesiones.
10. Base de datos involucrada.
11. Recurso esperado.
12. Tipo de Lock.
13. Query del proceso bloqueado.
14. Query o procedimiento asociado al bloqueador.

Posteriormente se debe determinar si se trató de:

- Un bloqueo temporal esperado.
- Una transacción abierta durante demasiado tiempo.
- Una consulta de larga duración.
- Procesos concurrentes modificando la misma información.
- Consultas accediendo a los mismos recursos en diferente orden.
- Falta o uso inadecuado de índices.
- Una operación masiva.
- Un proceso que quedó abierto sin finalizar su transacción.

---

## Un evento no significa necesariamente un problema diferente

Si una misma sesión permanece bloqueada durante un periodo prolongado, pueden registrarse **varios `blocked_process_report` relacionados con el mismo bloqueo**.

Por ejemplo:

```text
08:15:10  SPID 184 bloqueado por SPID 177
08:15:20  SPID 184 bloqueado por SPID 177
08:15:30  SPID 184 bloqueado por SPID 177
08:15:40  SPID 184 bloqueado por SPID 177
```

Esto no necesariamente representa cuatro problemas independientes.

Puede representar **el mismo bloqueo persistiendo durante aproximadamente 40 segundos**.

Por esta razón, al analizar el historial se deben revisar conjuntamente:

```text
FechaHora
SPID bloqueado
SPID bloqueador
Recurso
Consulta
```

para determinar si los eventos pertenecen al mismo incidente.

---

# IMPORTANTE

Antes de ejecutar cualquier consulta o realizar acciones sobre el servidor, se debe verificar que se está trabajando en el **servidor y ambiente correctos**.

Confirma siempre:

- Servidor.
- Ambiente (Pruebas o Producción).
- Base de datos.
- Fecha y horario del incidente.
- SPID bloqueado.
- SPID bloqueador.
- Usuario y equipo involucrados.
- Consulta o proceso relacionado.

> La existencia de un `blocked_process_report` **no significa automáticamente que se deba ejecutar `KILL` sobre el proceso bloqueador**.

El evento debe utilizarse primero para determinar la causa del bloqueo y evaluar su impacto.

Un `KILL` sobre una sesión con una transacción abierta puede provocar un `ROLLBACK`, afectar procesos relacionados y generar impacto en otros usuarios o aplicaciones.

**Antes de realizar cualquier acción que pueda afectar la operación del servidor, el diagnóstico y la acción propuesta deberán validarse con el responsable o superior correspondiente.**

> **Primero diagnosticar, después validar y finalmente actuar.**
