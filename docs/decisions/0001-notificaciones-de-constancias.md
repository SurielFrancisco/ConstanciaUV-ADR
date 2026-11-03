---
date: 2026-11-03
status: proposed
---

# Decisión arquitectónica: Procesamiento de notificaciones de constancias

## Contexto y planteamiento del problema

ConstanciaUV debe permitir que estudiantes autorizados soliciten y
obtengan constancias digitales. Después de generar y almacenar una
constancia, el sistema debe notificar al estudiante mediante un servicio
externo de correo electrónico.

El problema arquitectónico consiste en decidir cómo organizar la
comunicación entre el proceso de generación de constancias y el servicio
externo de notificaciones, de manera que la indisponibilidad de dicho
proveedor no interrumpa la operación principal del sistema.

La decisión está condicionada por las siguientes restricciones:

-   **RT1.** El servicio de notificaciones por correo electrónico es
    externo a ConstanciaUV.
-   **RT2.** Si el proveedor de correo no está disponible, la generación
    y el almacenamiento de constancias deberán continuar.
-   **RT3.** Las notificaciones pendientes deberán conservarse para
    intentar su envío posteriormente.
-   **RT4.** El sistema deberá prevenir solicitudes de notificación
    duplicadas para una misma constancia. No se presupone que el
    proveedor externo garantice por sí mismo una entrega única.

## Drivers de la decisión

1.  **Continuidad de la generación de constancias (RT2):** la generación
    y el almacenamiento deben mantenerse independientes de la
    disponibilidad del proveedor externo de correo.
2.  **Persistencia y control de notificaciones pendientes (RT3 y RT4):**
    el sistema debe conservar las notificaciones que no pudieron
    enviarse y evitar que una misma constancia origine solicitudes
    duplicadas.

## Alternativas consideradas

-   **Alternativa A. Comunicación directa con recuperación de fallos:**
    después de almacenar una constancia, la aplicación intenta solicitar
    directamente el envío de la notificación. Si el proveedor no
    responde, registra el envío pendiente para reintentarlo
    posteriormente.
-   **Alternativa B. Procesamiento asíncrono mediante eventos
    persistidos:** después de almacenar una constancia, la aplicación
    registra una notificación pendiente. Un componente independiente
    procesa posteriormente esos registros y se comunica con el proveedor
    externo.

## Resultado de la decisión

Se propone la **Alternativa B: procesamiento asíncrono mediante eventos
persistidos**.

Esta alternativa separa la generación y el almacenamiento de constancias
del envío de notificaciones. Una vez almacenada la constancia, el
sistema registra de forma persistente la notificación pendiente y
permite que un componente independiente intente enviarla.

La propuesta responde a RT2 porque la generación de constancias no
depende de que el proveedor de correo esté disponible en ese momento.
También responde a RT3 al conservar los registros pendientes para su
procesamiento posterior. Para atender RT4, cada notificación deberá
asociarse a un identificador único de constancia y contar con una
restricción o mecanismo de idempotencia que impida registrar solicitudes
duplicadas. Los reintentos deberán reutilizar el mismo identificador
lógico de notificación.

## Consecuencias

-   **Beneficio:** la generación y el almacenamiento de constancias
    pueden continuar aunque el proveedor externo de correo esté
    indisponible.
-   **Beneficio:** las notificaciones pendientes se conservan y pueden
    procesarse posteriormente mediante un componente independiente.
-   **Beneficio:** la separación de responsabilidades facilita modificar
    o sustituir la integración con el proveedor de correo sin alterar
    directamente el proceso de generación de constancias.
-   **Costo o riesgo:** se introduce un componente adicional y se
    requiere administrar los estados de las notificaciones, los
    reintentos, los errores y la supervisión del procesamiento.
-   **Costo o riesgo:** puede existir un retraso entre la generación de
    la constancia y la recepción del correo.
-   **Trade-off:** se favorecen la continuidad operativa y el
    desacoplamiento entre procesos, a cambio de mayor complejidad y de
    una notificación que no necesariamente será inmediata.

## Confirmación

En una implementación futura, la decisión podría verificarse mediante
las siguientes pruebas y revisiones:

1.  **Prueba de indisponibilidad del proveedor:** simular que el
    servicio externo no responde y comprobar que la constancia se genera
    y almacena correctamente, mientras la notificación permanece
    registrada como pendiente.
2.  **Prueba de recuperación:** restablecer el proveedor y comprobar que
    el componente independiente recupera y procesa las notificaciones
    pendientes.
3.  **Prueba de duplicados:** intentar registrar más de una solicitud de
    notificación para la misma constancia y verificar que el mecanismo
    de unicidad o idempotencia impide crear solicitudes duplicadas.
4.  **Prueba de reintentos:** provocar un error de envío y comprobar que
    los reintentos conservan el identificador de la notificación y no
    generan nuevos registros lógicos.
5.  **Revisión de persistencia:** comprobar que los registros pendientes
    sobreviven a un reinicio del componente encargado de procesarlos.

Estas comprobaciones son propuestas para una implementación futura; no
se afirma que hayan sido ejecutadas.

## Ventajas y desventajas de las alternativas

### Alternativa A. Comunicación directa con recuperación de fallos

-   **Ventaja:** permite intentar el envío inmediatamente después de
    almacenar la constancia, por lo que la notificación podría
    realizarse con menor demora cuando el proveedor está disponible.
-   **Desventaja:** la aplicación que genera la constancia también debe
    coordinar la comunicación con el proveedor y gestionar los fallos,
    los pendientes y los reintentos. Esto mantiene una mayor relación
    entre el proceso principal y la integración externa.

### Alternativa B. Procesamiento asíncrono mediante eventos persistidos

-   **Ventaja:** separa la generación de constancias del envío de
    notificaciones. El componente independiente puede procesar los
    pendientes y reintentar los envíos sin detener la operación
    principal.
-   **Desventaja:** requiere implementar y mantener el almacenamiento de
    notificaciones pendientes, el componente procesador, el control de
    estados, los reintentos y los mecanismos contra duplicados. Además,
    la entrega puede ocurrir después de que la constancia haya sido
    generada.

## Más información

### Revisión 1. Cambio en el proveedor de notificaciones

**Cambio recibido:** El proveedor externo de notificaciones modificó su interfaz de comunicación y la forma en que reporta los errores de envío.

**Impacto sobre la alternativa seleccionada:** El cambio afecta al componente independiente que procesa las notificaciones pendientes, particularmente a la integración encargada de comunicarse con el proveedor e interpretar sus respuestas. La generación y el almacenamiento de constancias, así como el registro persistente de notificaciones pendientes, no necesitan modificarse por este cambio.

**Vigencia de la decisión:** Se mantiene la propuesta de procesamiento asíncrono mediante eventos persistidos. La separación entre la generación de constancias y el envío de notificaciones permite localizar la adaptación en el componente de integración, sin alterar el flujo principal. La decisión original sigue siendo justificable porque continúa respondiendo a RT2, RT3 y RT4.

**Consecuencia o riesgo que debe revisarse:** Debe revisarse el riesgo de interpretar incorrectamente los nuevos códigos o respuestas de error. Una clasificación incorrecta podría impedir reintentos necesarios o provocar reintentos innecesarios.

**Comprobación que se agrega:** Incorporar una prueba de integración que utilice las nuevas respuestas de error del proveedor y compruebe que el componente las interpreta correctamente, clasifica los fallos y determina si corresponde reintentar el envío o mantener la notificación pendiente. La prueba debe contemplar errores temporales y errores que no se resuelven mediante un reintento. También debe verificarse que la adaptación no modifica ni duplica los registros de notificación existentes.