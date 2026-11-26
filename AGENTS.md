Proyecto ESTACITAS — SABANA CITAS

1. Contexto

Proyecto de desarrollo de una aplicación web para la E.S.E. Hospital Santa Mónica, orientada a consultar, filtrar, analizar y generar reportes estadísticos a partir de información histórica de citas médicas.

2. Infraestructura existente

Servidor SQL Server de producción: LQLR-BD.

Base de datos origen: PANACEA.

Servidor SQL Server estadístico: RPRTS.

Base de datos estadística: ESTADISTICO.

Tabla consolidada: dbo.SABANA_CITAS.

Tabla de auditoría: dbo.LOG_CARGA_SABANA_CITAS.

Procedimiento de carga: dbo.SP_CARGAR_SABANA_CITAS.

Procedimiento mensual: dbo.SP_CARGAR_SABANA_CITAS_MENSUAL.

Job de SQL Server Agent: JOB_CARGA_MENSUAL_SABANA_CITAS.

Linked Server existente: LQLR-BD.

El histórico consolidado comprende octubre de 2019 a octubre de 2026, con 2.411.554 registros y conciliación anual sin diferencias de cantidad.

La primera ejecución automática del Job está programada para el 1 de diciembre de 2026 a las 02:00 a. m. y está pendiente de validación operativa.

3. Objetivo de la aplicación

Desarrollar una plataforma web para consultar y analizar los datos de SABANA_CITAS, generar reportes y presentar indicadores.

Los requerimientos específicos, tecnologías, arquitectura y módulos deberán definirse antes de comenzar la implementación.

4. Reglas de seguridad

No modificar la base de datos PANACEA.

No ejecutar operaciones de escritura sobre producción.

No modificar tablas, procedimientos ni Jobs existentes sin autorización explícita.

Utilizar conexiones de solo lectura para consultar los datos estadísticos de la aplicación, salvo que se apruebe una necesidad específica de escritura.

No almacenar contraseñas, cadenas de conexión reales ni secretos en Git.

No incluir información personal o sensible real de pacientes en ningún archivo del repositorio, documentación interna o pública, registros, ejemplos ni pruebas.

Utilizar datos ficticios para pruebas y demostraciones.

Aplicar controles de acceso y privilegio mínimo.

5. Reglas de desarrollo

Trabajar por etapas.

Explicar brevemente el objetivo y alcance antes de realizar cambios importantes.

Evitar modificaciones fuera del alcance autorizado.

Mantener código organizado, legible y documentado.

Incorporar pruebas para las funcionalidades implementadas.

No seleccionar frameworks, librerías o herramientas principales sin discutir previamente la arquitectura.

No ejecutar despliegues ni modificaciones de infraestructura sin autorización.

Mantener compatibilidad con SQL Server 2017.

6. Reglas Git

La rama main debe tratarse como rama estable.

No realizar commits directamente sobre main.

Crear ramas de trabajo a partir de main, usando convenciones descriptivas como docs/, feature/, fix/, refactor/ o test/.

Utilizar Pull Requests para integrar modificaciones.

No ejecutar push, merge o eliminaciones de ramas sin autorización explícita.

Verificar las políticas reales del repositorio antes de asumir que existen protecciones de ramas.

7. Forma de colaboración

Responder en español.

Trabajar paso a paso.

Priorizar precisión técnica y seguridad de datos.

Cuando se solicite una verificación, realizarla sin modificar archivos.

Indicar qué archivos se crean o modifican.

No asumir que una funcionalidad está implementada sin verificar el código correspondiente.