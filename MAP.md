# MAP.md — KMPapp

Plan de desarrollo de ESTE repositorio, autocontenido. El mapa global del proyecto (las 8 fases y su orden) vive en `MAP.md` en la raiz del workspace y no se propaga. Fuentes de verdad: `PROJECT_STATEMENT-v1.md`, `INTEGRATION_REFERENCE-v2.md`, `ARQUITECTURA.md`, `CatalogAndSync/Docs/CU_1.md` (cuenta del usuario final).

## Dependencias de otros repos
- **Entradas necesarias para avanzar**: registro/login y busquedas contra CatalogAndSync (Fase 2/3 de ese repo); disponibilidad, inicio de reserva y resultados contra TurnosYReservas (Fases 4-6 de ese repo).
- **Salidas que los backends consumen de vos**: el ingreso del telefono llega por el flujo de reserva (solo cuando TurnosYReservas lo pide).
- La app corre FUERA de los contenedores.

## Decisiones abiertas (no asumir; consultar al usuario)
| Decision | Bloquea | Referencia |
| --- | --- | --- |
| Organizacion de la UI y navegacion | Toda la app | PS §10 |
| Modelo de estados/cache local de datos en la app | — | — |

## Límite absoluto
La app usa exclusivamente el JWT del usuario final. El JWT tecnico y las credenciales de integracion con la catedra NUNCA residen en este repositorio.

## Fase 7 — Aplicacion Android (KMP)

- 7.1 Scaffold Kotlin Multiplatform con Android ejecutable. Solo Android obligatorio; segunda plataforma opcional.
- 7.2 Registro: `login`, `password`, `firstName`, `lastName`, `email`, `imageUrl` (opcional), `langKey` (default `es`), compatible JHipster. El backend hashea y valida; la app no persiste el password ni el JWT como credencial expuesta.
- 7.3 Login contra CatalogAndSync; el JWT de usuario se usa como Authorization en las llamadas a los backends.
- 7.4 Busqueda y filtros sobre el catalogo local (categoria, nombre, estado habilitado y disponibilidad) — contra CatalogAndSync, nunca contra la catedra. "Mis turnos" ver disponibles esten o no actualizados al dia de la consulta (CU_1 §1.3).
- 7.5 Disponibilidad e inicio de reserva contra TurnosYReservas: el usuario elige un turno; si ya esta tomado se muestra la excepcion, se actualiza el listado y se vuelve a elegir (CU_1 §1.4).
- 7.6 Ingreso del telefono SOLO cuando TurnosYReservas lo solicita (tras el hold y su confirmacion REST, evento `AdditionalInformationRequested`); no antes.
- 7.7 Mis reservas y cancelacion de la reserva propia; notificacion si el turno se cancela de forma externa (CU_1 §1.5).
- Criterio: el proceso funcional completo se ejecuta desde la app, sin operaciones manuales externas (PS §3.2).

## Evidencia
Las pruebas de la UI son opcionales y complementarias (PS §10.1); lo obligatorio es que el proceso funcional completo se demuestre desde la app ejecutable.