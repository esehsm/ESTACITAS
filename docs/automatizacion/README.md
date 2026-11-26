# Documento maestro — Automatización de SABANA_CITAS

**Proyecto:** ESTACITAS — E.S.E. Hospital Santa Mónica  
**Tipo:** Referencia técnica para el repositorio `esehsm/ESTACITAS`  
**Fecha de corte documental:** 26 de noviembre de 2026  
**Estado:** Carga histórica validada; primera ejecución automática mensual pendiente de evidencia  
**Ruta recomendada:** `docs/automatizacion/README.md`

> **Alcance y procedencia.** Documento elaborado con el historial de implementación y con los scripts SQL de trabajo disponibles. Describe el estado reportado y las verificaciones realizadas; no sustituye una inspección actual de SQL Server. No contiene contraseñas ni registros de pacientes. Los scripts originales deben contrastarse con las definiciones vigentes en el servidor antes de cualquier cambio.

## 1. Propósito y límites

La solución mantiene una **fotografía histórica de las citas** en una base estadística independiente de PANACEA. Permite consultas analíticas sin tener que recorrer directamente tablas transaccionales para cada reporte y conserva los valores capturados al momento de la extracción.

**Fuera del alcance de la automatización actual:** la construcción de la aplicación web, la autenticación de sus usuarios, dashboards, APIs y nuevas reglas de negocio. Esos temas se definirán en las sesiones de arquitectura y desarrollo.

## 2. Infraestructura y flujo de datos

```text
Servidor origen: LQLR-BD                 Servidor destino: RPRTS
Base PANACEA                              Base ESTADISTICO
Agenda.TM_CITAS y tablas relacionadas     dbo.SABANA_CITAS
          │                                      ▲
          └── Linked Server [LQLR-BD] ──────────┘
                       │
        dbo.SP_CARGAR_SABANA_CITAS
                       ▲
        dbo.SP_CARGAR_SABANA_CITAS_MENSUAL
                       ▲
        SQL Server Agent / Job mensual
                       │
             dbo.LOG_CARGA_SABANA_CITAS
```

- **Origen:** SQL Server de producción `LQLR-BD`, base `PANACEA`; consulta principal `Agenda.TM_CITAS`, enriquecida con tablas relacionadas de parametrización, agendas, usuarios, programas, etc.
- **Destino:** servidor `RPRTS`, SQL Server 2017, base `ESTADISTICO`, esquema `dbo`.
- **Transporte:** Linked Server `[LQLR-BD]` desde `RPRTS`, mediante acceso SQL de solo lectura configurado previamente. La consulta emplea nombres de cuatro partes (`[LQLR-BD].[PANACEA].[esquema].[tabla]`).
- **Datos personales:** la sábana contiene información potencialmente sensible (identificación, contacto y otra información relacionada con la atención). Su tratamiento requiere autorización, control de acceso, auditoría y minimización en la aplicación.

Se utilizó consulta distribuida con nombres de cuatro partes porque la consulta consolidada superaba el límite práctico de 8 KB de texto literal de `OPENQUERY`.

## 3. Objetos implementados

| Objeto | Ubicación | Función |
|---|---|---|
| `dbo.SABANA_CITAS` | `RPRTS.ESTADISTICO` | Almacena las fotografías históricas de citas |
| `UX_SABANA_CITAS_ID_CITA` | `RPRTS.ESTADISTICO` | Índice único de `ID_CITA`, evita duplicados |
| `dbo.LOG_CARGA_SABANA_CITAS` | `RPRTS.ESTADISTICO` | Guarda resultados, fechas y errores de cargas |
| `dbo.SP_CARGAR_SABANA_CITAS` | `RPRTS.ESTADISTICO` | Realiza la extracción y carga por rango de fechas |
| `dbo.SP_CARGAR_SABANA_CITAS_MENSUAL` | `RPRTS.ESTADISTICO` | Calcula automáticamente el mes calendario anterior |
| `JOB_CARGA_MENSUAL_SABANA_CITAS` | SQL Server Agent de `RPRTS` | Ejecuta el procedimiento mensual según calendario |
| `PROGRAMACION_MENSUAL_SABANA_CITAS` | SQL Server Agent de `RPRTS` | Dispara el Job el día 1 de cada mes a las 02:00 |

### 3.1. Tabla `dbo.SABANA_CITAS`

La tabla tiene **90 columnas**: 88 provenientes o calculadas desde la consulta consolidada y dos campos de trazabilidad:

- `FECHA_CARGA DATETIME NOT NULL` (valor de inserción `GETDATE()`).
- `TIPO_CARGA VARCHAR(20)` (etiqueta de tipo de extracción).

La clave de control es `ID_CITA BIGINT NOT NULL`, protegida por índice único. Los valores de columnas restantes fueron derivados con `SELECT TOP (0) ... INTO` a partir del `SELECT` original, por lo que sus tipos exactos deben obtenerse del catálogo SQL si se documenta un contrato de API.

**Inventario de columnas según el `INSERT` del procedimiento compartido:**

| # | Columna |
|---:|---|
| 1 | `ID_CITA` |
| 2 | `ID_MEDIO_SOLICITUD` |
| 3 | `MEDIO_SOLICITUD` |
| 4 | `TIPO_IDENTIFICACION` |
| 5 | `NUMERO_IDENTIFICACION` |
| 6 | `ID_PACIENTE` |
| 7 | `PRIMER_NOMBRE` |
| 8 | `SEGUNDO_NOMBRE` |
| 9 | `PRIMER_APELLIDO` |
| 10 | `SEGUNDO_APELLIDO` |
| 11 | `NOMBRE_COMPLETO_PACIENTE` |
| 12 | `EDAD` |
| 13 | `ID_GENERO` |
| 14 | `GENERO` |
| 15 | `ABR_GENERO` |
| 16 | `Telefono_Domicilio` |
| 17 | `DIRECCION` |
| 18 | `BARRIO` |
| 19 | `MUNICIPIO` |
| 20 | `DEPARTAMENTO` |
| 21 | `ZONA` |
| 22 | `PERTENENCIA_ETNICA` |
| 23 | `CORREO_ELECTRONICO` |
| 24 | `NIVEL_EDUCACION` |
| 25 | `ID_PROGRAMA` |
| 26 | `PROGRAMA` |
| 27 | `ID_SEDE` |
| 28 | `SEDE` |
| 29 | `ID_TIPO_PRESTACION` |
| 30 | `TIPO_PRESTACION` |
| 31 | `ID_ESPECIALIDAD` |
| 32 | `ESPECIALIDAD` |
| 33 | `ID_PROCEDIMIENTO` |
| 34 | `PROCEDIMIENTO` |
| 35 | `TIPO_CITA` |
| 36 | `CLASIFICACION_PROCEDIMIENTO` |
| 37 | `TIPO_PROCEDIMIENTO` |
| 38 | `FECHA_INICIAL_AGENDA` |
| 39 | `FECHA_FINAL_AGENDA` |
| 40 | `HORAS_PROGRAMACION` |
| 41 | `MINUTOS_PROGRAMACION` |
| 42 | `ID_ESTADO_CITA` |
| 43 | `ESTADO_CITA` |
| 44 | `CITA_EXTRA` |
| 45 | `CITA_GRUPAL` |
| 46 | `FECHA_SOLICITUD` |
| 47 | `FECHA_PROGRAMADA` |
| 48 | `DURACION` |
| 49 | `OPORTUNIDAD` |
| 50 | `OPORTUNIDAD_ESPERADA` |
| 51 | `FECHA_LLEGADA` |
| 52 | `LLEGA_TARDE` |
| 53 | `LLEGA_TEMPRANO` |
| 54 | `ID_PRESTADOR` |
| 55 | `PRESTADOR` |
| 56 | `AUTORIZACION` |
| 57 | `VIGENCIA_AUTORIZACION` |
| 58 | `ID_CONVENIO` |
| 59 | `CONVENIO` |
| 60 | `EPS` |
| 61 | `PERSONA_SOLICITA` |
| 62 | `IP_ORIGEN_SOLICITUD` |
| 63 | `USUARIO_SOLICITUD` |
| 64 | `NOMBRE_USUARIO_SOLICITUD` |
| 65 | `GESTANTE` |
| 66 | `ID_IPS` |
| 67 | `IPS` |
| 68 | `VALOR_PACIENTE` |
| 69 | `ID_ESTRUCTURA_FISICA` |
| 70 | `ESTRUCTURA_FISICA` |
| 71 | `ID_CITA_PADRE` |
| 72 | `ID_GRUPO_SERVICIO` |
| 73 | `GRUPO_SERVICIO` |
| 74 | `CODIGO_ALTERNO` |
| 75 | `ID_MODULO_FUNCIONAL` |
| 76 | `MODULO_FUNCIONAL` |
| 77 | `ID_TARIFARIO` |
| 78 | `TARIFARIO` |
| 79 | `OBSERVACIONES` |
| 80 | `USUARIO_LEGALIZA` |
| 81 | `NOMBRE_USUARIO_LEGALIZA` |
| 82 | `FECHA_SOLICITUD_PACIENTE` |
| 83 | `ID_MODALIDAD_ATENCION` |
| 84 | `MODALIDAD_ATENCION` |
| 85 | `FECHA_FINAL_CITA` |
| 86 | `MES` |
| 87 | `ANO` |
| 88 | `MES_Y_ANO` |
| 89 | `FECHA_CARGA` |
| 90 | `TIPO_CARGA` |

### 3.2. Tabla `dbo.LOG_CARGA_SABANA_CITAS`

Campos documentados: `ID_LOG` (identidad, PK), `FECHA_INICIO_EJECUCION`, `FECHA_FIN_EJECUCION`, `FECHA_INICIO_PERIODO`, `FECHA_FIN_PERIODO`, `TIPO_CARGA`, `REGISTROS_INSERTADOS`, `ESTADO`, `ERROR_NUMERO`, `ERROR_MENSAJE`.

Estados observados en la lógica del procedimiento: `EJECUTANDO`, `OK`, `ERROR`.

## 4. Reglas de extracción y comportamiento

1. **Rango de fecha semiabierto:** `C.FECHA_PROGRAMADA >= @FECHA_INICIO AND C.FECHA_PROGRAMADA < @FECHA_FIN`. El límite final es exclusivo; así se evitan solapamientos de períodos.
2. **Prevención de duplicados:** `NOT EXISTS` contra `dbo.SABANA_CITAS` por `ID_CITA`, con `UPDLOCK, HOLDLOCK`; el índice `UNIQUE` es segunda barrera.
3. **Sin actualización de históricos:** las citas que ya existen no se vuelven a enriquecer ni se actualizan; son una fotografía de su primera extracción.
4. **Concurrencia:** `sp_getapplock` (`Exclusive`, propietario de sesión, `LockTimeout=0`) impide dos cargas simultáneas del mismo procedimiento en RPRTS.
5. **Tipos válidos:** `PRUEBA`, `HISTORICO`, `MENSUAL`, `RECUPERACION`.
6. **Auditoría:** se crea registro `EJECUTANDO`, luego se actualiza a `OK` con `@@ROWCOUNT`; si hay error, a `ERROR` con número y mensaje cuando el registro está disponible.
7. **Propagación de errores:** el bloque `CATCH` finaliza con `THROW;` para que SQL Server Agent pueda marcar el paso como fallido.
8. **Transacciones distribuidas:** se mantuvo `SET XACT_ABORT OFF` por el error `Msg 3989` detectado con la consulta remota. **No cambiar esto sin pruebas controladas.**

**Precaución de consistencia:** el procedimiento documentado no envuelve toda la carga en una transacción explícita. Un fallo o desconexión puede requerir verificación/reintento; el estado de auditoría por sí solo no prueba una reversión atómica de la operación. La recuperación debe usar un período explícito y comparar los identificadores y los conteos antes de cerrar el incidente.

## 5. Procedimientos almacenados

### 5.1. `dbo.SP_CARGAR_SABANA_CITAS`

**Parámetros:**

| Parámetro | Tipo | Semántica |
|---|---|---|
| `@FECHA_INICIO` | `DATETIME` | Inicio inclusivo |
| `@FECHA_FIN` | `DATETIME` | Fin exclusivo |
| `@TIPO_CARGA` | `VARCHAR(20)` | Etiqueta validada de ejecución |

Ejemplo **documental** de ejecución para recuperación manual, **no ejecutar sin revisión y autorización**:

```sql
USE ESTADISTICO;
GO
EXEC dbo.SP_CARGAR_SABANA_CITAS
    @FECHA_INICIO = '20261101',
    @FECHA_FIN = '20261201',
    @TIPO_CARGA = 'RECUPERACION';
```

La ejecución anterior **inserta datos** y no debe utilizarse como consulta de verificación.

### 5.2. `dbo.SP_CARGAR_SABANA_CITAS_MENSUAL`

Se apoya en la fecha local del servidor `RPRTS` (`GETDATE()`) para calcular el intervalo del mes calendario anterior y llamar al procedimiento principal con `@TIPO_CARGA = 'MENSUAL'`.

```sql
DECLARE @FECHA_FIN DATETIME = DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()), 0);
DECLARE @FECHA_INICIO DATETIME = DATEADD(MONTH, -1, @FECHA_FIN);
-- Se procesaría el intervalo [@FECHA_INICIO, @FECHA_FIN)
```

Por ejemplo, si se ejecuta el **1 de diciembre de 2026**, el intervalo es `[2026-11-01 00:00:00, 2026-12-01 00:00:00)`.

## 6. Histórico consolidado y validación

| Año | Período cubierto | Registros conciliados |
|---|---|---:|
| 2019 | Octubre–diciembre | 580 |
| 2020 | Enero–diciembre | 205.788 |
| 2021 | Enero–diciembre | 327.582 |
| 2022 | Enero–diciembre | 313.083 |
| 2023 | Enero–diciembre | 372.950 |
| 2024 | Enero–diciembre | 393.639 |
| 2025 | Enero–diciembre | 432.061 |
| 2026 | Enero–octubre | 365.871 |
| **Total** | **Oct. 2019 – oct. 2026** | **2.411.554** |

La comparación de cantidad de registros entre PANACEA y SABANA_CITAS por año y mes terminó **sin diferencias de conteo** en el cierre histórico. La igualdad de conteos no sustituye una comprobación exhaustiva de IDs y contenido de cada fila; no se debe documentar esa última comparación como realizada sin evidencia.

Se comprobó la idempotencia con una prueba histórica de 15 inserciones y un segundo intento que insertó 0 registros.

## 7. Programación del Job y pruebas efectuadas

**Job:** `JOB_CARGA_MENSUAL_SABANA_CITAS`  
**Paso:** `01_EJECUTAR_CARGA_MENSUAL`  
**Comando:** `EXEC dbo.SP_CARGAR_SABANA_CITAS_MENSUAL;`  
**Base de datos del paso:** `ESTADISTICO`  
**Calendario:** `PROGRAMACION_MENSUAL_SABANA_CITAS`, día 1 de cada mes a las **02:00**, a partir del **01/12/2026**, sin fin definido.  
**Propietario configurado:** `sa`.

Verificaciones efectuadas durante la configuración:

- SQL Server Agent se encontraba en estado `Running`, inicio `Automatic`, en `RPRTS`.
- Usuario de configuración `sa`, miembro de `sysadmin`.
- El Job llegó a verificarse con estado deshabilitado y el calendario habilitado; posteriormente el responsable informó que lo habilitó.
- Un Job temporal `JOB_TEST_CONEXION_PANACEA` ejecutado desde Agent pudo consultar `TOP (1) ID` de `PANACEA.Agenda.TM_CITAS` a través del Linked Server y finalizó correctamente.
- Se revisó la definición del procedimiento principal y se constató el `THROW` final del bloque de excepción.

**Estado al 26/11/2026:** la programación está preparada y se informó su habilitación, **pero aún no hay evidencia de la primera carga mensual real por el Job**. No describir el resultado de diciembre como exitoso antes de comprobarlo.

## 8. Operación: verificar ejecución mensual

### 8.1. Revisión del registro de auditoría (solo lectura)

```sql
USE ESTADISTICO;
GO
SELECT TOP (20)
    ID_LOG, FECHA_INICIO_EJECUCION, FECHA_FIN_EJECUCION,
    FECHA_INICIO_PERIODO, FECHA_FIN_PERIODO,
    TIPO_CARGA, REGISTROS_INSERTADOS, ESTADO,
    ERROR_NUMERO, ERROR_MENSAJE
FROM dbo.LOG_CARGA_SABANA_CITAS
ORDER BY ID_LOG DESC;
```

Verificar un registro `MENSUAL`, estado `OK`, y límites del período anterior. Un resultado de **0 registros insertados** no demuestra por sí solo un problema: comparar el número esperado de citas y la existencia de registros en destino.

### 8.2. Comprobar cantidad en el destino (solo lectura)

```sql
USE ESTADISTICO;
GO
SELECT COUNT_BIG(*) AS TOTAL_NOVIEMBRE_2026
FROM dbo.SABANA_CITAS
WHERE FECHA_PROGRAMADA >= '20261101'
  AND FECHA_PROGRAMADA < '20261201';
```

### 8.3. Comprobar la programación vigente del Agent (solo lectura)

```sql
USE msdb;
GO
SELECT
    j.name AS JOB,
    j.enabled AS JOB_HABILITADO,
    s.name AS PROGRAMACION,
    s.enabled AS PROGRAMACION_HABILITADA,
    s.active_start_date AS FECHA_INICIO,
    s.active_start_time AS HORA_INICIO
FROM dbo.sysjobs AS j
LEFT JOIN dbo.sysjobschedules AS js ON js.job_id = j.job_id
LEFT JOIN dbo.sysschedules AS s ON s.schedule_id = js.schedule_id
WHERE j.name = N'JOB_CARGA_MENSUAL_SABANA_CITAS';
```

### 8.4. Historial de ejecuciones del Agent (solo lectura)

```sql
USE msdb;
GO
SELECT TOP (20)
    h.instance_id, j.name AS JOB,
    h.step_id, h.step_name, h.run_date, h.run_time,
    h.run_duration, h.run_status, h.message
FROM dbo.sysjobhistory AS h
INNER JOIN dbo.sysjobs AS j ON j.job_id = h.job_id
WHERE j.name = N'JOB_CARGA_MENSUAL_SABANA_CITAS'
ORDER BY h.instance_id DESC;
```

`run_status`: 0 = error; 1 = éxito; 2 = reintento; 3 = cancelado; 4 = en progreso. En esta tabla la fila `step_id=0` corresponde al resultado global del Job.

**Criterios mínimos de cierre de diciembre:** Job ejecutado a la hora prevista; historial con éxito; auditoría `MENSUAL/OK` para noviembre; recuento de noviembre conciliado con producción y revisión de incidencias. Registrar fecha, ID de auditoría y evidencia en la documentación operativa.

## 9. Incidentes conocidos y recuperación

| Situación | Evidencia / criterio | Acción prudente |
|---|---|---|
| Error de consulta distribuida `3989` | Descriptor de transacción inválido con XACT_ABORT ON | Mantener `XACT_ABORT OFF` hasta revisar el diseño transaccional de forma aislada |
| Error de acceso Linked Server desde Agent | Mensaje de permiso o conexión | Revisar historial, contexto del Job y mapeo de seguridad; no ampliar permisos sin necesidad |
| Registro `ERROR` en auditoría | `ERROR_NUMERO`, `ERROR_MENSAJE` | Comparar carga parcial y origen; autorizar repetición del período si procede |
| Job termina sin carga esperada | Sin registros nuevos o conteos diferentes | Validar rangos, fechas y disponibilidad de origen, luego investigar |
| Ejecución duplicada | Bloqueo aplicativo o `ID_CITA` único | No deshabilitar controles; verificar motivo y trazas |

**Operación de recuperación:** confirmar que no hay otra carga activa, delimitar el mes faltante, evaluar cantidad/IDs presentes, registrar el incidente, autorizar la ejecución con `TIPO_CARGA='RECUPERACION'` y conciliar de nuevo. Un `NOT EXISTS` evita repetir IDs ya cargados, pero no recalcula los valores congelados previamente.

## 10. Restricciones y consideraciones para la futura aplicación

- Consultar preferentemente `RPRTS.ESTADISTICO.dbo.SABANA_CITAS` y **no** el origen transaccional para generar reportes habituales.
- Exponer datos mediante servicios autenticados; evitar conexión directa del navegador a SQL Server.
- Definir acceso por roles, trazabilidad de consultas, uso de filtros y medidas de rendimiento sobre millones de filas.
- Realizar **mínima exposición de datos personales**, especialmente identificación, teléfono, correo y dirección.
- No interpretar `FECHA_CARGA` como fecha de cita, ni `TIPO_CARGA` como estado clínico.
- Los datos son fotografías históricas, no un reflejo continuamente actualizado; la carga mensual ocurre a mes vencido.
- Para escalamiento, revisar índices sobre `FECHA_PROGRAMADA` y otros filtros usados por la app con base en planes de ejecución; **no agregar índices sin una evaluación de impacto**.

## 11. Decisiones pendientes / backlog técnico

1. Registrar la evidencia de primera ejecución real el **01/12/2026**.
2. Documentar y validar los tipos reales y nulabilidad de las 90 columnas usando `sys.columns` y `sys.types`.
3. Verificar, si se requiere garantía exhaustiva, igualdad de conjuntos `ID_CITA` entre origen y destino en los períodos cargados, con bajo impacto a producción.
4. Revisar la consistencia frente a interrupciones y el manejo de fallos durante `INSERT` y actualización de auditoría.
5. Definir monitoreo y alertas ante errores del Job, con responsables y canal aprobado.
6. Definir retención de logs y revisión de respaldos de `ESTADISTICO`.
7. Confirmar la visibilidad del repositorio institucional y la política de publicación de nombres de infraestructura interna.

## 12. Referencias de trabajo

- Definición compartida del procedimiento principal `dbo.SP_CARGAR_SABANA_CITAS` (incluye los 90 nombres del `INSERT`, consulta distribuida, control de concurrencia y bloque de errores).
- Scripts históricos disponibles en el contexto del trabajo: `01_Crear_SABANA_CITAS_V3.sql`, `04_SP_CARGAR_SABANA_CITAS_CORREGIDO.sql`. Son **artefactos de trabajo**, no se presume que sus versiones sean idénticas a la definición vigente en producción.
- Historial de implementación, pruebas de SQL Server Agent y conciliación del proyecto de automatización.

---

**Convención de mantenimiento:** actualizar este documento con Pull Request, acompañando cualquier cambio operativo relevante con fecha, motivo, responsable técnico y evidencia; no introducir secretos ni registros reales de pacientes en Git.
