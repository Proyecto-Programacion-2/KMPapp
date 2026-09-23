# Arquitectura del sistema distribuido de turnos

Registro revisable de la arquitectura logica y de las decisiones adoptadas y pendientes. Este documento se actualiza cada vez que se cambie una decision arquitectural; no es un enunciado inamovible.

Estado actual: los tres repositorios aun no tienen codigo. La arquitectura se define a partir de `PROJECT_STATEMENT-v1.md` (enunciado) e `INTEGRATION_REFERENCE-v2.md` (contratos de catedra, fuente de verdad para los contratos externos).

## 1. Mapa logico general

```
                 ───── APP KMP (Android) ─────
   Registro/Login   Busqueda/Filtros   Disponibilidad   Reserva(telefono)   Mis reservas
        │                │                  │                 │                │
        └──────── APIs propias (DTO propios) [DEC] ───────────┘
                            ▼  JWT de usuario
┌─────────────────────────────────────────────────────────────────────────┐
│                         BACKEND PROYECTO (2 servicios)                  │
│                                                                          │
│  CATALOGANDSYNC         ── contrato interno ──▶   TURNOSYRESERVAS        │
│  . identidad: registro/ │ ◀── [DEC: JWT, DTO,      . disponibilidad        │
│    login + JWT usuario  │      endpoints]          . maquina de estados    │
│  . copia local catalogo                            . propiedad por usuario │
│    + version aplicada                              . cliente REST catedra  │
│  . sync completa/incremental                       . Kafka consumidor/prod │
│  . deteccion discontinuidad                        . [DEC] BD · maquina ·  │
│  . busquedas sobre datos locales                     dedup · reintentos ·  │
│  . [DEC] BD · entidades · estrategia                 mapeo externalPatientId│
│    transaccional sync · dedup · namespace Redis      · autenticacion interna│
└─────────┬──────────────────────────────────────────────┬────────────────┘
          ▼  REST snapshot + JWT tecnico                 ▼ REST/Kafka
┌─────────────────────────────────────────────────────────────────────────┐
│                 SERVICIO CENTRAL CATEDRA (EXTERNO, FIJO)                │
│   REST: snapshot · occupancies · appointment-holds · holds/{id}/confirm │
│         · appointments · appointments/{pid}/cancel                      │
│   Redis catedra:sync:* (solo lectura) — versiones, metadata, hashes     │
│   Kafka: catedra.catalog.{gid} ─▶  alumnos.turnos.acciones.{gid} ◀─    │
│         ─▶  catedra.turnos.telefono.{gid} (respuesta del alumno)        │
└─────────────────────────────────────────────────────────────────────────┘

Persistencia local: PostgreSQL, una instancia, esquemas separados por servicio
(`catalog` y `turnos`), usuarios de BD sin permisos cruzados, migraciones
independientes. Infraestructura de backends y dependencias: Docker Compose.
```

## 2. Flujo de punta a punta (usuario final → turno confirmado)

**Fase 0 — Provisionamiento (unica vez, manual)**
1. Registrar la cuenta tecnica con Postman → `POST /api/student/register`, obtener `id_token` (JWT 1 anio) + `integration` (hosts, topics, consumer group) [IR §5.1]. Fijo: guardarlo externalizado, jamas en el repo ni en KMP [IR §5.1, §9]. [DEC: como se guarda el secreto, rotacion via `POST /api/authenticate`].

**Fase 1 — Autenticacion del usuario final**
2. KMP → CatalogAndSync: registro/login; hashea la contrasena, crea usuario activo inmediato (modelo JHipster: `login`, `password`, `firstName`, `lastName`, `email`, `imageUrl`, `langKey`) y emite JWT de usuario [PS §3.2, §9]. TurnosYReservas valida ese JWT para identificar al usuario, pero no emite sesiones.
3. De ese usuario se deriva un identificador estable que viaja como `externalPatientId` en los contratos con catedra [IR §2.1]. [DEC: formato del identificador].

**Fase 2 — Catalogo local vigente (CatalogAndSync)**
4. Inicializacion: BD local vacia, discontinuidad o fuera de ventana → snapshot REST (guardar `snapshotVersion` solo tras aplicar las tres colecciones) [PS §6.1, IR §7, §14.2]. Fijo.
5. Incremental: Kafka `CatalogUpdated(newVersion, eventId)` → leer Redis `catedra:sync:changes:{v}` + hashes, aplicar en orden y avanzar version de forma atomica; deduplicar por `eventId` [PS §6.2, IR §14.5, §15.3, §16]. [DEC: estrategia transaccional de la sync, dedup, deteccion de perdida de notificaciones — PS §10].

**Fase 3 — Busqueda sobre datos locales**
6. KMP → CatalogAndSync: filtros por categoria, nombre, estado habilitado y disponibilidad, resueltos siempre sobre la copia local (prohibido consultar a catedra por busqueda) [PS §4.1, §6]. [DEC: endpoints/DTO de busqueda; el filtro por disponibilidad se resuelve consultando a Turnos].

**Fase 4 — Disponibilidad**
7. KMP → Turnos: profesional + fecha.
8. Turnos → CatalogAndSync: agenda semanal vigente del profesional (contrato interno) [PS §4.2]. Fijo: no usar descriptivos historicos de reservas como catalogo vigente; sin catalogo no se inician operaciones que exijan validar [PS §4.2, §8]. [DEC: contrato interno completo: endpoints, DTO, tiempo de validez de la respuesta].
9. Turnos → catedra: `GET /api/appointment-occupancies` (reservas confirmadas + holds; si aparece en ambos, prevalece `CONFIRMED`) [IR §8].
10. Turnos combina horarios semanales − ocupaciones y devuelve slots disponibles [PS §7]. [DEC: construccion/ordenamiento de la grilla; degradacion si catalogo caido ("seguir atendiendo las operaciones que dependan solo de datos propios") — PS §8].

**Fase 5 — Inicio de la reserva**
11. KMP → Turnos: iniciar reserva (slot + usuario).
12. Turnos valida el slot contra la agenda vigente y guarda proceso local con su estado e identidad de usuario [PS §4.2]. [DEC: maquina de estados — PS §10].
13. Turnos → catedra: `POST /api/appointment-holds` → `holdId`, `reservationProcessId`, `expiresAt`. Fijo: guardar `expiresAt`; el TTL real lo manda catedra, una copia local no lo extiende [IR §9, PS §7].
14. Turnos → catedra: `POST /api/appointment-holds/{holdId}/confirm` con `externalPatientId`, nombre y apellido → `202 WAITING_FOR_PHONE` [IR §10].

**Fase 6 — Intercambio asincrono del telefono (Kafka)**
15. Catedra publica `AdditionalInformationRequested` (topic acciones, key = `reservationProcessId`) [IR §15.4].
16. Turnos lo consume, deduplica por `eventId`, conserva `requestEventId` y pide el telefono en KMP [IR §15.4, §18.1]. [DEC: politica de dedup y de reintentos].
17. KMP → Turnos: telefono; Turnos publica `AdditionalInformationSubmitted` en el topic de telefono con los campos exactos (`eventType`, `schemaVersion`, `eventId`, key, `requestEventId`) [IR §15.5].
18. Catedra valida y llega un resultado final por Kafka: `AppointmentConfirmed` | `AdditionalInformationRejected` (reintentable hasta `expiresAt`, permite re-submit con `eventId` nuevo) | `AppointmentProcessExpired` | `AppointmentProcessInvalid` [IR §15.6-15.9].
19. Turnos consume, deduplica, hace la transicion final de la maquina de estados. Fijo: el estado final no retrocede; REST y Kafka pueden llegar desfasados pero deben converger sin duplicados [PS §7, IR §18.3].

**Fase 7 — Gestion del resultado**
20. KMP → Turnos: "mis reservas" — la API de catedra devuelve toda la cuenta tecnica; el backend filtra por el propietario local y nunca expone reservas ajenas [PS §4.2, IR §11]. [DEC: consulta/polling vs push del estado].
21. KMP → Turnos: cancelar reserva propia confirmada → `POST /api/appointments/{pid}/cancel` → `AppointmentCancelled` → estado local [IR §12, §15.10].

**Fase 8 — Robustez transversal [PS §8]**
22. Sobre todo el flujo: reinicios con procesos pendientes, duplicados de Kafka, timeouts con efecto incierto — antes de rehacer una operacion con efectos, reconciliar por los IDs persistidos; reintentos limitados y observables [IR §18.2-18.4]. [DEC: timeouts, backoff, limites, observabilidad — PS §10].

## 3. Decisiones ya adoptadas

| Decision | Valor | Ref. |
| --- | --- | --- |
| Registro/login de usuarios finales | Vive en CatalogAndSync (emite JWT de usuario); TurnosYReservas valida el JWT y deriva `externalPatientId` | PS §3.2, §9; IR §2.1 |
| Entorno de desarrollo/pruebas | Stub local de catedra en Docker Compose (gobernado por IR §6-15 y §17) + verificacion contra catedra real cuando lleguen las credenciales | IR §3; PS §3 |
| Motor de BD | PostgreSQL, instancia unica, esquemas separados (`catalog` / `turnos`), usuarios sin permisos cruzados, migraciones independientes | PS §3 |
| Hold no se cancela manualmente | No existe endpoint de cancelacion de hold (IR §6); el hold expira segun `expiresAt` de la catedra y una copia local no extiende el TTL. Si el usuario abre otro turno, el hold anterior queda sin confirmar y expira | IR §6, §9; PS §7 |

## 4. Decisiones pendientes (NO asumir; consultar al usuario)

- TurnosYReservas: maquina de estados local del proceso/reserva, motor de BD y entidades internas, idempotencia/deduplicacion y manejo de mensajes fuera de orden, timeouts/backoff/limites de reintentos, origen y formato de `externalPatientId`, autenticacion interna hacia CatalogAndSync (propagar JWT de usuario vs usar JWT tecnico — PS §9), observabilidad, alcance de pruebas.
- CatalogAndSync: motor de BD y entidades, estrategia transaccional de sincronizacion, dedup de notificaciones/cambios y deteccion de discontinuidad, uso del namespace privado de Redis `alumnos:{groupId}:*` (auxiliar, no fuente de verdad — IR §14.6), contrato interno hacia Turnos, endpoints/DTO de busqueda, alcance de pruebas.
- Globales: endpoints/DTO propios entre KMP y backends, politica de errores internos, documentacion y evidencias.

## 5. Lo fijo (no se decide)

- Contratos REST/Redis/Kafka con catedra (IR v2): mapa REST (IR §6), errores por `status` + `code` (IR §13), topics y eventos (IR §15), recuperacion (IR §18).
- Tecnologia: Java + Spring Boot para ambos backends, Kotlin Multiplatform para la app.
- Modelo de usuario compatible JHipster; usuario activo inmediato tras registro.
- Sincronizacion: snapshot cuando BD vacia/discontinuidad/sin ventana; incremental por Kafka→Redis en orden; la version solo avanza cuando la unidad de trabajo termino correctamente.
- Busquedas resueltas solo sobre datos locales.
- Antes de operar, informacion vigente de catalogo; sin catalogo no se inician operaciones que lo requieran; se degrada atendiendo operaciones propias.
- Ocupaciones: `CONFIRMED` prevalece sobre `HELD`; holds manipulados solo por REST.
- Kafka: entrega al menos una vez, `eventId` como clave de idempotencia, message key = `reservationProcessId`, campos exactos en `AdditionalInformationSubmitted`.
- El estado final del proceso no retrocede; los estados observados por REST y Kafka deben converger sin duplicados.
- JWT tecnico de catedra nunca en KMP; APIs de catedra operan sobre toda la cuenta tecnica y el backend aplica la propiedad por usuario.
- Persistencia principal en gestor de BD servidor (prohibido H2/SQLite/embebidas); infraestructura local con Docker Compose.
- Credenciales, tokens y secretos externalizados.

## Inventario de decisiones tuyas (lo que NO está fijado)

TurnosYReservas §10, IR §19
- Máquina de estados local del proceso/reserva y su persistencia.
- Motor de BD y entidades/tablas internas (debe ser BD servidor; prohibido H2/SQLite).
- Idempotencia/deduplicación, manejo de mensajes fuera de orden.
- Timeouts, backoff, límites de reintentos y tratamiento de indisponibilidad de catedra/catálogo.
- Origen y formato de externalPatientId.
- Endpoints/DTO propios hacia KMP y hacia CatalogAndSync, y autenticación interna (¿propagar JWT de usuario o usar JWT técnico? — §9).
- Obviamente también: cobertura de pruebas y cómo se documenta.

CatalogAndSync §10
- Motor de BD, entidades y copia local.
- Estrategia transaccional de la sincronización (aplicar versión atómica).
- Dedup de notificaciones/cambios, detección de discontinuidad y reconstrucción.
- Uso (o no) del namespace privado de Redis alumnos:{groupId}:* (auxiliar, no fuente de verdad) IR §14.6.
- Contrato interno hacia Turnos.

Globales
- ¿En cuál servicio viven las cuentas de usuarios finales y el "punto de entrada" del login?
- ¿Stub local de cátedra en Docker Compose para desarrollar/probar sin el entorno real, o trabajar contra cátedra real?
- Motor/esquema de BD compartido vs instancias separadas.

## 6. Referencias

- `PROJECT_STATEMENT-v1.md`: PS §3, §3.2, §4.1, §4.2, §5, §6, §7, §8, §9, §10.
- `INTEGRATION_REFERENCE-v2.md`: IR §2, §2.1, §3, §5.1, §6, §7, §8, §9, §10, §11, §12, §13, §14.2, §14.5, §14.6, §15, §15.3-15.10, §16, §17, §18, §19.

Las versiones canonicas de estos documentos viven en la raiz del proyecto. Los cambios a este archivo se propagan a los tres repositorios.
