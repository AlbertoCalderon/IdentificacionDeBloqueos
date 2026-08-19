# Identificación y Análisis de Deadlocks en SQL Server

Esta guía permite identificar y analizar **Deadlocks** registrados en SQL Server mediante Extended Events.

El objetivo es determinar:

- Cuándo ocurrió el Deadlock.
- Qué procesos participaron.
- Qué proceso fue seleccionado como víctima.
- Qué consultas estaban involucradas.
- Qué recursos provocaron el conflicto.

---

## Contenido

1. [¿Qué es un Deadlock?](#1-qué-es-un-deadlock)
2. [Diferencia entre Bloqueo y Deadlock](#2-diferencia-entre-bloqueo-y-deadlock)
3. [Consultar Deadlocks registrados](#3-consultar-deadlocks-registrados)
4. [Buscar Deadlocks por horario](#4-buscar-deadlocks-por-horario)
5. [Interpretar el Deadlock XML](#5-interpretar-el-deadlock-xml)
6. [Identificar la víctima](#6-identificar-la-víctima)
7. [Identificar los procesos involucrados](#7-identificar-los-procesos-involucrados)
8. [¿Qué hacer después de encontrar un Deadlock?](#8-qué-hacer-después-de-encontrar-un-deadlock)

---

# 1. ¿Qué es un Deadlock?

Un **Deadlock** ocurre cuando dos o más procesos mantienen recursos bloqueados y, al mismo tiempo, esperan recursos que se encuentran bloqueados por los otros procesos.

Ejemplo simplificado:

```text
PROCESO A
   │
   ├── Tiene bloqueado → Recurso 1
   │
   └── Espera          → Recurso 2
                           ▲
                           │
PROCESO B                  │
   │                       │
   ├── Tiene bloqueado → Recurso 2
   │
   └── Espera          → Recurso 1
```

Ninguno de los procesos puede continuar.

SQL Server detecta automáticamente esta situación y selecciona uno de los procesos como **víctima del Deadlock**.

La transacción de la víctima es cancelada y revertida mediante un `ROLLBACK`, permitiendo que el otro proceso continúe.

La aplicación correspondiente a la víctima normalmente recibe el error:

```text
Transaction (Process ID XX) was deadlocked on lock resources
with another process and has been chosen as the deadlock victim.
Rerun the transaction.
```

---

# 2. Diferencia entre Bloqueo y Deadlock

Un **bloqueo normal** y un **Deadlock** no son lo mismo.

| Situación | Bloqueo | Deadlock |
|---|---|---|
| Un proceso espera a otro | Sí | Sí |
| Puede resolverse cuando termina el bloqueador | Sí | No por sí solo |
| Existe dependencia circular | No necesariamente | Sí |
| SQL Server selecciona una víctima | No | Sí |
| Se cancela automáticamente una transacción | No | Sí |
| Puede consultarse mientras está ocurriendo | Sí | Generalmente se analiza después del evento |

### Bloqueo normal

```text
Proceso A
   │
   └── Bloquea recurso
          │
          ▼
      Proceso B espera
```

Cuando el proceso A termina y libera el recurso, el proceso B puede continuar.

### Deadlock

```text
Proceso A ─── espera ───► Proceso B
    ▲                       │
    │                       │
    └────── espera ─────────┘
```

Existe una dependencia circular.

SQL Server debe romper el ciclo seleccionando una víctima.

---

# 3. Consultar Deadlocks registrados

Los Deadlocks pueden consultarse desde los archivos generados por la sesión de **Extended Events**.

```sql
SELECT
    DATEADD(
        HOUR,
        DATEDIFF(HOUR, GETUTCDATE(), GETDATE()),
        x.event_data.value('(event/@timestamp)[1]', 'datetime2')
    ) AS FechaHora,

    x.event_data.query(
        '(event/data[@name="xml_report"]/value/deadlock)[1]'
    ) AS DeadlockXML

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
    ) = 'xml_deadlock_report'

ORDER BY FechaHora DESC;
```

## Resultado

La consulta devuelve principalmente:

| Campo | Descripción |
|---|---|
| `FechaHora` | Fecha y hora en la que ocurrió el Deadlock. |
| `DeadlockXML` | Información completa del Deadlock registrada por SQL Server. |

---

# 4. Buscar Deadlocks por horario

Cuando un usuario reporta un problema en un horario específico, es recomendable limitar la búsqueda al periodo donde ocurrió el incidente.

Por ejemplo, para consultar los Deadlocks ocurridos el **19 de agosto de 2026 entre las 08:00 AM y las 09:00 AM**:

```sql
SELECT
    DATEADD(
        HOUR,
        DATEDIFF(HOUR, GETUTCDATE(), GETDATE()),
        x.event_data.value('(event/@timestamp)[1]', 'datetime2')
    ) AS FechaHora,

    x.event_data.query(
        '(event/data[@name="xml_report"]/value/deadlock)[1]'
    ) AS DeadlockXML

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
    ) = 'xml_deadlock_report'

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

De esta manera se analizan únicamente los eventos ocurridos durante ese periodo.

---

# 5. Interpretar el Deadlock XML

La columna:

```text
DeadlockXML
```

contiene la información detallada del Deadlock.

Al abrir el XML se pueden encontrar tres partes importantes:

```text
deadlock
│
├── victim-list
│
├── process-list
│
└── resource-list
```

## `victim-list`

Indica qué proceso fue seleccionado por SQL Server como víctima.

```xml
<victim-list>
    <victimProcess id="process123" />
</victim-list>
```

El identificador:

```text
process123
```

debe buscarse posteriormente dentro de `process-list`.

---

## `process-list`

Contiene información sobre todos los procesos involucrados en el Deadlock.

Puede incluir información como:

```text
SPID
Usuario
Host
Base de datos
Aplicación
Estado de la transacción
Consulta ejecutada
Recurso solicitado
```

Ejemplo simplificado:

```xml
<process
    id="process123"
    spid="177"
    hostname="SRVCORP-CTX01"
    loginname="usuario"
    clientapp="Aplicacion">
```

---

## `resource-list`

Muestra los recursos involucrados en el conflicto.

Aquí se puede observar:

- Qué recurso estaba bloqueado.
- Qué proceso era propietario del bloqueo.
- Qué proceso estaba esperando.
- Qué tipo de Lock tenía cada proceso.
- Qué tipo de Lock estaba solicitando.

Esta sección es fundamental para determinar **por qué los procesos terminaron bloqueándose entre sí**.

---

# 6. Identificar la víctima

Dentro del XML se debe localizar:

```xml
<victim-list>
```

Por ejemplo:

```xml
<victim-list>
    <victimProcess id="process123" />
</victim-list>
```

Después se busca:

```text
process123
```

dentro de:

```xml
<process-list>
```

Ahí se podrá identificar el SPID correspondiente.

Ejemplo:

```xml
<process
    id="process123"
    spid="177"
    hostname="SRVCORP-CTX01"
    loginname="usuario">
```

En este caso:

```text
Víctima del Deadlock
        │
        ▼
 process123
        │
        ▼
    SPID 177
```

> Ser seleccionado como víctima **no significa necesariamente que ese proceso haya provocado el Deadlock**.

SQL Server selecciona una transacción para romper el ciclo y permitir que las demás puedan continuar.

---

# 7. Identificar los procesos involucrados

Una vez localizada la víctima, se deben revisar **todos los procesos participantes**, no solamente el proceso cancelado.

Para cada proceso es recomendable identificar:

| Información | ¿Qué permite conocer? |
|---|---|
| `spid` | Sesión involucrada. |
| `hostname` | Equipo donde se originó la conexión. |
| `loginname` | Usuario utilizado para conectarse. |
| `clientapp` | Aplicación que originó la operación. |
| `currentdb` | Base de datos involucrada. |
| `inputbuf` | Consulta o procedimiento ejecutado. |
| Lock solicitado | Recurso que necesitaba el proceso. |
| Lock retenido | Recurso que estaba bloqueando. |

El objetivo es reconstruir algo similar a:

```text
SPID 177
UPDATE TablaA
      │
      │ Tiene bloqueado TablaA
      │ Espera TablaB
      ▼
   SPID 184
UPDATE TablaB
      │
      │ Tiene bloqueado TablaB
      │ Espera TablaA
      └──────────────► DEADLOCK
```

Esto permite determinar qué operaciones entraron en conflicto.

---

# 8. ¿Qué hacer después de encontrar un Deadlock?

Encontrar el Deadlock es solamente el primer paso.

Se recomienda recopilar la siguiente información:

1. Fecha y hora exacta del evento.
2. SPID seleccionado como víctima.
3. SPID de los demás procesos involucrados.
4. Usuario de cada sesión.
5. Host de cada sesión.
6. Aplicación que originó cada conexión.
7. Base de datos involucrada.
8. Stored Procedures o consultas ejecutadas.
9. Tablas o recursos involucrados.
10. Tipo de Locks solicitados y retenidos.

Esta información permitirá determinar si el problema está relacionado con:

- Transacciones demasiado largas.
- Acceso a tablas en diferente orden.
- Consultas o procedimientos que modifican los mismos registros.
- Índices insuficientes o poco eficientes.
- Procesamiento concurrente de la misma información.
- Nivel de aislamiento utilizado.
- Diseño de la transacción.

> **Importante:** Un Deadlock no se soluciona ejecutando `KILL`. Cuando se consulta el evento, SQL Server ya detectó el Deadlock y seleccionó automáticamente una víctima para romper el ciclo.

---

# IMPORTANTE

Antes de ejecutar cualquier consulta o realizar acciones sobre el servidor, se debe verificar que se está trabajando en el **servidor y ambiente correctos**.

Confirma siempre:

- Servidor.
- Ambiente (Desarrollo, Pruebas o Producción).
- Base de datos.
- Fecha y horario del incidente.
- Usuarios y procesos involucrados.

La información del Deadlock debe utilizarse principalmente para **diagnosticar la causa del problema**.

No se recomienda realizar modificaciones en procedimientos, índices, transacciones o configuraciones del servidor únicamente con base en un evento aislado sin realizar previamente el análisis correspondiente.

**Antes de realizar cualquier acción que pueda afectar la operación del servidor, el diagnóstico y la acción propuesta deberán validarse con el responsable o superior correspondiente.**

> **Primero diagnosticar, después validar y finalmente actuar.**
