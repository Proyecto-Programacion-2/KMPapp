# KMPapp — Aplicacion Android (Kotlin Multiplatform)

Interfaz de usuario desarrollada con Kotlin Multiplatform. Solo Android es obligatorio; una segunda plataforma es opcional y no reduce el alcance. Repositorio: `github.com/Proyecto-Programacion-2/KMPapp`.

## Rol
- Registro e inicio de sesion contra el backend propio. El backend emite el JWT que la app usa durante la sesion; este token nunca se persiste como credencial expuesta.
- Modelo de usuario compatible con JHipster: `login`, `password`, `firstName`, `lastName`, `email`, `imageUrl` (opcional) y `langKey`, respetando las validaciones de catedra. Los IDs internos, estados de activacion y autoridades los administra el backend.
- Busca profesionales/filtros contra CatalogAndSync (copia local) y completa el flujo de reserva con TurnosYReservas (disponibilidad, hold, telefono, resultados).
- Debe implementar el proceso funcional completo; no puede depender de operaciones manuales externas para completar los casos de uso obligatorios.

## Limites criticos
- La app usa exclusivamente el JWT del usuario final. El JWT y las credenciales de integracion tecnica con la catedra nunca residen en KMP.

## Documentacion fuente de verdad (copiadas en este repo)
- `PROJECT_STATEMENT-v1.md` — interfaz grafica y autenticacion (seccion 3.2).
- `INTEGRATION_REFERENCE-v2.md` — identidades y flujo de reserva visible para el usuario (secciones 2 y 15).

Versiones canonicas en la raiz: `/home/franco/Facultad/Programacion-2/`.

Cada repositorio replica tambien `ARQUITECTURA.md`. Al modificar cualquiera de estos documentos, propagar el cambio a los 3 repositorios y a la raiz.

El plan de desarrollo de LA APP es `MAP.md` (este repo), autocontenido. El mapa global del proyecto vive en `MAP.md` en la raiz del workspace y no se propaga.

## Pautas
- Agente cooperador, no generador de codigo: escribir codigo solo cuando se pida; consultar antes de modificar archivos.
- CRITICO: decisiones de organizacion de la UI y arquitectura se consultan al usuario. NUNCA asumir.
- La app Android se ejecuta fuera de los contenedores Docker.