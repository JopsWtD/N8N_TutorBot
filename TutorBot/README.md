# TutorBot — Sistema de Asesorías Académicas

Proyecto Final 2778 · Automatización de tutorías con Telegram + n8n + Google Sheets + IA.

## 1. El problema

**Formulación:** Actualmente el estudiante necesita encontrar un tutor disponible para su materia, pero enfrenta un proceso disperso entre correos, mensajes informales y falta de visibilidad sobre la disponibilidad real de los tutores. Esto provoca cruces de horario, solicitudes sin atender y ausencia de trazabilidad. TutorBot debería ayudar a automatizar la solicitud, asignación y seguimiento de una asesoría, sin depender de coordinación manual en cada paso, y sin permitir que dos personas reserven el mismo recurso al mismo tiempo.

## 2. Actores

| Actor | Necesidad | Cómo se resuelve en TutorBot |
|---|---|---|
| Estudiante | Solicitar, consultar y cancelar una tutoría | Bot de Telegram + wizard de registro + formularios web para solicitar/cancelar |
| Tutor | Recibir solicitudes y gestionar su agenda | Bot de Telegram: botones Aceptar/Rechazar, "agenda" |
| Coordinación académica | Visibilidad y trazabilidad de la actividad | Reporte semanal por correo (HTML) + alertas de tutores con muchos timeouts |
| Sistema TutorBot | Mantener reglas y datos consistentes | 18 workflows en n8n, cada uno con una responsabilidad única |

## 3. Arquitectura general — 18 workflows

**Principio seguido:** un workflow = una responsabilidad de negocio (evitando el "workflow gigante" que la guía advierte explícitamente que hay que evitar). La comunicación entre workflows usa **Execute Sub-workflow** en todos los casos con un llamador fijo conocido de antemano (no cuenta como ejecución extra); se mantiene **Webhook real** solo donde el destino se decide en tiempo de ejecución (ver sección 8).

| # | Workflow | Responsabilidad |
|---|---|---|
| **Núcleo** | | |
| 01 | Núcleo - Administrador de Sesiones | Leer/guardar/limpiar el estado del wizard de cada usuario |
| 02 | Núcleo - Definición de Pasos | Leer la hoja STEPS y agruparla |
| 03 | Núcleo - Calcular Disponibilidad | Motor de asignación: materia + tutor activo + disponibilidad + sin cruce de horario |
| 04 | Núcleo - Notificaciones | Único punto de envío por Telegram (estudiante/tutor) y Gmail (HTML) |
| 05 | Núcleo - Procesador de Pasos | Motor del wizard: valida cada respuesta según `type`, sin un IF por pregunta |
| **Estudiante** | | |
| 06 | Estudiante - Router (entrada) | Entrada real de Telegram; detecta si hay flujo activo o estudiante registrado |
| 07 | Estudiante - Clasificador de Intención (IA) | **Agente de IA** (Gemini 3.1 Flash Lite) clasifica la intención del mensaje libre y dirige |
| 08 | Estudiante - Registro (wizard chat) | Primeros pasos del registro: ¿eres estudiante? + documento |
| 09 | Estudiante - Registro (formulario datos) | Nombre y confirmación final, por formulario web |
| 10 | Estudiante - Validar Documento | Valida que el documento tenga 6-15 dígitos |
| 11 | Estudiante - Solicitar Tutoría (formulario) | Formulario de 3 páginas en cascada: Materia+Fecha → Tutor → Hora |
| 12 | Estudiante - Cancelar Tutoría (formulario) | Formulario con dropdown de tutorías activas del estudiante |
| 13 | Estudiante - Consultar Tutorías | "Mis tutorías" por chat |
| **Tutor** | | |
| 14 | Tutor - Router (entrada) | Entrada real de Telegram; distingue mensaje de texto vs. botón |
| 15 | Tutor - Aceptar o Rechazar | Procesa el botón de una solicitud |
| 16 | Tutor - Agenda | "Agenda" por chat |
| **Automatización programada** | | |
| 17 | Cron - Timeouts y Monitoreo | Cierra solicitudes vencidas (>2h sin respuesta) + alerta si un tutor acumula 3+ timeouts |
| 18 | Cron - Reporte Semanal | Correo HTML a coordinación cada lunes con materia y tutor top |

```
        ESTUDIANTE (Telegram)                        TUTOR (Telegram)
               │                                            │
    06 Router de Estudiante                        14 Router del Tutor
     (flujo activo? / registrado?)               (mensaje vs. boton)
               │                                    │            │
      ┌────────┼─────────┐                          ▼            ▼
      ▼        ▼          ▼                    15 Aceptar/   16 Agenda
  08 Registro  07 Clasificador (IA)              Rechazar
  (wizard)         │
      │      ┌──────┼───────┬────────────┐
      ▼      ▼      ▼       ▼            ▼
  09 Form  11 Sol.  13     12 Cancelar  (info/ayuda)
  Datos    Tutoria  Consultar Tutoria
      │      │
      ▼      ▼
  10 Validar 03 Calcular Disponibilidad
  Documento

  Todos pasan por: 01 Sesion, 05 Procesador de Pasos (solo wizard chat),
  02 Definicion de Pasos (solo wizard chat), 04 Notificaciones (todos)
  17 y 18 corren solos, programados.
```

## 4. Modelo de datos — punto de partida vs. lo implementado

*(sin cambios respecto a la versión anterior de este documento — ver `docs/analisis.md` para la justificación completa de cada ajuste frente al modelo de 4 hojas del enunciado)*

Las 7 hojas finales de `TutorBot_DB`: ESTUDIANTES, TUTORES, TUTOR_MATERIA, DISPONIBILIDAD, TUTORIAS, SESSION, STEPS.

## 5. Flujo conversacional como máquina de estados (FSM)

**Solo el registro usa el motor de wizard genérico (Sesión + STEPS + Procesador de Pasos).** Solicitar y Cancelar Tutoría migraron a formularios web de varias páginas porque su naturaleza es distinta — "elegir de una lista que depende de la anterior", no "responder una pregunta a la vez". Esto es una decisión consciente: se usó la herramienta correcta para cada problema, no la misma para todo.

**Persistencia:** el estado del wizard vive en la hoja SESSION (`flujo_actual`, `pasos_restantes`, `datos_parciales`, `direccion_retorno`), leído/escrito en cada turno porque cada mensaje de Telegram es una ejecución de n8n independiente — no hay memoria de proceso entre una y otra.

## 6. Ciclo de vida de la tutoría

```
Solicitada → Asignada → Confirmada → Finalizada
                 │
                 ├── timeout (17) ──┐
                 └── rechazo (15) ──┴──► Cancelada (con motivo_cierre)
```

`motivo_cierre`: `rechazo_tutor | timeout_tutor | cancelacion_estudiante`.

## 7. Motor de asignación (reglas de negocio)

Dado `materia` + `fecha`, un horario es válido si: (1) existe tutor para esa materia en TUTOR_MATERIA, (2) ese tutor está `Activo`, (3) tiene disponibilidad habitual ese día de la semana, (4) el bloque no coincide con una tutoría ya existente en estado activo para ese tutor.

## 8. Decisiones de arquitectura

- **Webhook real solo donde el destino se decide en tiempo de ejecución:** el paso "finalizar" del registro, y el handler `Validar Documento` (step tipo `logic`). Todo lo demás usa Execute Sub-workflow.
- **IA solo para clasificar intención** (workflow 07, Gemini 3.1 Flash Lite + salida estructurada), nunca para decidir asignación, horario o estado — la asignación, validación de fecha/documento y reglas de negocio son 100% determinísticas (Code nodes y condiciones), justo la separación que pide la guía en su sección 12.

## 9. Cómo importar y probar

1. Crear el Google Sheet `TutorBot_DB` con las 7 hojas de la sección 4.
2. Importar los `.json` de `workflows/` a una instancia de n8n.
3. Configurar credenciales: Google Sheets, Telegram (2 bots), Gmail, y un modelo de IA (Google Gemini).
4. Activar todos los workflows salvo `17 - Cron - Timeouts y Monitoreo` (dejar inactivo hasta la demo final).
5. Completar la hoja STEPS con el flujo `registro` (ver `docs/analisis.md`).

## 10. Limitaciones conocidas y qué cambiaría en producción

- **Concurrencia / race conditions:** no resuelto, solo documentado — ver `docs/analisis.md`.
- **Menú de materias estático:** el formulario de Solicitar Tutoría (11) tiene el dropdown de materias fijo en el diseño del nodo, no leído dinámicamente de TUTOR_MATERIA (la primera página de un formulario no puede depender de datos previos en la misma ejecución). Agregar una materia nueva requiere editar ese nodo a mano.
- **Timezone:** fechas/horas como texto simple, sin zona horaria explícita.
- **Credenciales de Telegram:** viven en la configuración de los nodos HTTP dentro de n8n (no en el repositorio ni en las hojas), pero en texto plano dentro de la URL del nodo — en producción se recomendaría moverlas a una credencial de n8n dedicada en vez de la URL.

## 11. Uso de IA

| Decisión | ¿Quién decide? |
|---|---|
| Clasificar la intención de un mensaje libre del estudiante | IA (Gemini 3.1 Flash Lite + salida estructurada) |
| Validar fecha, documento, disponibilidad, asignar tutor/horario | Reglas determinísticas |
| Redacción de mensajes al usuario | Texto fijo, no generado por IA |

## 12. Capturas

### 12.1 Workflows

#### 01 · Administrador de Sesiones
![Administrador de Sesiones](capturas/workflows/01-administrador-sesiones.png)

#### 02 · Definición de Pasos
![Definición de Pasos](capturas/workflows/02-definicion-pasos.png)

#### 03 · Calcular Disponibilidad
![Calcular Disponibilidad](capturas/workflows/03-calcular-disponibilidad.png)

#### 04 · Notificaciones
![Notificaciones](capturas/workflows/04-notificaciones.png)

#### 05 · Procesador de Pasos
![Procesador de Pasos](capturas/workflows/05-procesador-pasos.png)

#### 06 · Router de Estudiante
![Router de Estudiante](capturas/workflows/06-router-estudiante.png)

#### 07 · Clasificador de Intención (IA)
![Clasificador de Intención](capturas/workflows/07-clasificador-intencion.png)

#### 08 · Registro (wizard chat)
![Registro wizard](capturas/workflows/08-registro-wizard.png)

#### 09 · Registro (formulario datos)
![Registro formulario](capturas/workflows/09-registro-formulario.png)

#### 10 · Validar Documento
![Validar Documento](capturas/workflows/10-validar-documento.png)

#### 11 · Solicitar Tutoría (formulario)
![Solicitar Tutoría](capturas/workflows/11-solicitar-tutoria.png)

#### 12 · Cancelar Tutoría (formulario)
![Cancelar Tutoría](capturas/workflows/12-cancelar-tutoria.png)

#### 13 · Consultar Tutorías
![Consultar Tutorías](capturas/workflows/13-consultar-tutorias.png)

#### 14 · Router del Tutor
![Router del Tutor](capturas/workflows/14-router-tutor.png)

#### 15 · Aceptar o Rechazar
![Aceptar o Rechazar](capturas/workflows/15-aceptar-rechazar.png)

#### 16 · Agenda del Tutor
![Agenda del Tutor](capturas/workflows/16-agenda-tutor.png)

#### 17 · Timeouts y Monitoreo
![Timeouts y Monitoreo](capturas/workflows/17-timeouts-monitoreo.png)

#### 18 · Reporte Semanal
![Reporte Semanal](capturas/workflows/18-reporte-semanal.png)

### 12.2 Conversaciones

#### Registro
![Registro](capturas/chat/chat-registro.png)
![Registro - formulario](capturas/chat/chat-registro-formulario.png)

#### Solicitar tutoría
![Solicitar - clasificación IA](capturas/chat/chat-solicitar-clasificacion.png)
![Solicitar - formulario](capturas/chat/chat-solicitar-formulario.png)
![Solicitar - confirmación](capturas/chat/chat-solicitar-confirmacion.png)

#### Tutor
![Tutor - solicitud con botones](capturas/chat/chat-tutor-solicitud.png)
![Tutor - agenda](capturas/chat/chat-tutor-agenda.png)

#### Consultar y cancelar
![Consultar tutorías](capturas/chat/chat-consultar.png)
![Cancelar - formulario](capturas/chat/chat-cancelar-formulario.png)

#### Casos de error y reportes
![Caso de error manejado](capturas/chat/chat-caso-error.png)
![Correo del reporte semanal](capturas/chat/chat-correo-reporte.png)

## 13. Estructura del repositorio

```
TutorBot/
├── README.md
├── workflows/          (los 18 .json de n8n, listos para importar)
├── docs/
│   ├── analisis.md
│   └── pruebas.md
└── capturas/
    ├── workflows/       (18 capturas de canvas)
    └── chat/            (capturas de conversaciones)
```
