# Proyecto integrador 2026: sistema distribuido de turnos

## 1. Presentacion general

El proyecto consiste en diseñar e implementar individualmente una aplicacion distribuida para consultar profesionales y gestionar reservas de turnos. La solucion tendra una aplicacion Android desarrollada con Kotlin Multiplatform (KMP) y dos servicios backend independientes implementados con Java y Spring Boot.

El servicio central administrado por la catedra sera la fuente autoritativa del catalogo de categorias, profesionales y horarios semanales. La aplicacion del alumno debera mantener una copia local de esa informacion para realizar busquedas y construir la disponibilidad de turnos.

El catalogo podra cambiar mientras la aplicacion se encuentre ejecutandose o desconectada. Por ese motivo, la solucion debera implementar sincronizacion completa e incremental, conservar la version local aplicada y reconstruir sus datos cuando haya perdido cambios.

La integracion con la catedra combinara REST, Redis y Kafka. REST se utilizara para autenticacion, snapshots, consulta de ocupaciones y operaciones sobre reservas; Redis contendra el catalogo publicado y su historial incremental; Kafka notificara cambios y permitira completar el proceso asincronico de reserva.

Los usuarios finales se registraran desde la aplicacion KMP y utilizaran un JWT para acceder a las funcionalidades protegidas. Cada usuario podra buscar profesionales utilizando la copia local y solamente podra consultar o cancelar sus propias reservas.

Para presentar los turnos disponibles, la solucion combinara los horarios semanales del catalogo con las ocupaciones informadas por la catedra. Cuando el usuario seleccione un horario, se solicitara un bloqueo temporal para evitar reservas concurrentes sobre el mismo turno.

La confirmacion de una reserva combinara REST y Kafka. Luego de confirmar inicialmente el bloqueo, la catedra solicitara un telefono mediante un evento; la solucion enviara esa informacion y recibira como resultado una confirmacion, un rechazo, una expiracion o la indicacion de un proceso invalido.

La solucion debera soportar mensajes duplicados, desconexiones temporales, vencimientos, perdida de cambios y reinicios. Cada alumno debera definir y justificar como organiza sus servicios, persistencia, seguridad, idempotencia, reintentos y recuperacion.

El caso de estudio es exclusivamente academico. No representa un sistema medico real y utilizara solamente datos ficticios.

## 2. Objetivos de aprendizaje

Al finalizar el proyecto, se espera que cada alumno pueda:

- diseñar servicios backend con responsabilidades y datos claramente separados;
- definir y consumir contratos entre sistemas;
- integrar APIs REST autenticadas, Redis y Kafka;
- mantener una replica local mediante sincronizacion completa e incremental;
- procesar mensajes de manera idempotente;
- resolver concurrencia y vencimiento de recursos temporales;
- recuperarse de desconexiones, mensajes duplicados y perdida de cambios;
- proteger los limites externos e internos de una aplicacion distribuida;
- implementar pruebas unitarias, de integracion y de contratos;
- documentar y defender decisiones tecnicas.

## 3. Solucion requerida

La solucion debera incluir una interfaz grafica desarrollada con Kotlin Multiplatform (KMP) y exactamente dos servicios backend independientes:

1. un servicio de catalogo y sincronizacion;
2. un servicio de turnos y reservas.

Los dos servicios deberan comunicarse mediante contratos explicitos y protegidos con JWT. Cada servicio sera propietario exclusivo de sus datos y administrara sus propias migraciones.

No se permitira:

- acceder directamente a las tablas, repositorios o estructuras internas del otro servicio;
- utilizar una base de datos compartida como mecanismo de integracion;
- duplicar en el servicio de turnos una replica del catalogo para busquedas o nuevas reservas;
- reemplazar los contratos entre servicios por acoplamiento a codigo interno.

Podra utilizarse una misma instancia fisica de base de datos siempre que existan bases logicas o esquemas separados, usuarios sin permisos cruzados y migraciones independientes.

Cada alumno podra elegir el motor de base de datos de sus servicios. La persistencia principal debera utilizar un gestor de base de datos servidor. No se aceptaran H2, SQLite ni otras bases embebidas o en memoria como almacenamiento principal de la solucion entregada.

Ambos servicios podran utilizar el mismo producto de base de datos o productos diferentes, siempre que mantengan la propiedad exclusiva de sus datos y cumplan las reglas de separacion indicadas anteriormente.

La infraestructura local de los dos servicios backend, sus bases de datos y las dependencias necesarias debera poder levantarse mediante Docker Compose. La aplicacion Android se ejecutara fuera de los contenedores.

### 3.1. Tecnologias backend

Los dos servicios backend deberan implementarse con Java y Spring Boot.

El uso de JHipster sera opcional, aunque recomendado por la catedra. Utilizarlo o no utilizarlo no modifica los requisitos funcionales, de integracion, seguridad, separacion de datos, pruebas y documentacion establecidos en este enunciado.

### 3.2. Interfaz grafica

La interfaz grafica debera desarrollarse con Kotlin Multiplatform (KMP), tener una aplicacion Android ejecutable e implementar el proceso funcional completo definido en este enunciado. No podra limitarse a una demostracion parcial ni depender de operaciones manuales externas para completar los casos de uso obligatorios.

No sera obligatorio entregar una segunda plataforma KMP en esta edicion. El alumno podra agregarla como extension, sin reemplazar ni reducir el alcance requerido para Android.

Cada usuario final debera poder registrarse e iniciar sesion desde la interfaz KMP contra el backend desarrollado por el alumno. Como resultado de la autenticacion, el backend emitira un JWT que la interfaz utilizara durante la sesion para consumir las APIs protegidas.

El modelo y el contrato de usuarios deberan ser compatibles con el usuario generado por JHipster. Esto permitira que un alumno que elija JHipster utilice su gestion de usuarios y que quien no lo utilice implemente los mismos datos y comportamientos observables.

El registro desde KMP contemplara `login`, `password`, `firstName`, `lastName`, `email`, `imageUrl` y `langKey`, respetando las validaciones definidas por la catedra. `imageUrl` sera opcional. Los identificadores internos, el estado de activacion, las autoridades y los campos de auditoria seran administrados por el backend y no podran ser elegidos libremente por quien se registra.

El usuario quedara activo inmediatamente despues de un registro correcto y podra iniciar sesion sin completar una verificacion por correo electronico. La recuperacion de contraseña sera opcional.

El backend debera validar los datos recibidos y almacenar la contraseña mediante un mecanismo criptografico adecuado; nunca en texto plano.

La cuenta de un usuario final de la aplicacion y la cuenta tecnica de integracion del proyecto ante el servicio de catedra representan identidades diferentes. Las credenciales de integracion con la catedra no deberan exponerse a la interfaz grafica ni a sus usuarios.

## 4. Responsabilidades de los servicios

### 4.1. Servicio de catalogo y sincronizacion

Sera responsable de:

- almacenar la copia local de categorias, profesionales y horarios semanales;
- conservar la version de catalogo aplicada;
- ejecutar la sincronizacion completa e incremental;
- detectar discontinuidades y reconstruir la copia local;
- buscar y filtrar profesionales utilizando exclusivamente datos locales;
- informar el estado y los errores de sincronizacion.

Este servicio sera la unica fuente local vigente para consultar profesionales y sus reglas de agenda.

La busqueda debera ofrecer, como minimo, filtros por categoria, nombre, estado habilitado y disponibilidad. Estos filtros se resolveran utilizando la copia local del catalogo y deberan poder utilizarse desde la interfaz KMP. El alumno podra incorporar filtros adicionales.

### 4.2. Servicio de turnos y reservas

Sera responsable de:

- construir la disponibilidad a partir de horarios y ocupaciones;
- iniciar y conservar el estado local de procesos de reserva;
- crear y confirmar inicialmente bloqueos temporales;
- participar del intercambio asincronico de informacion adicional;
- procesar confirmaciones, rechazos, vencimientos y procesos invalidos;
- consultar las reservas realizadas mediante la cuenta tecnica del proyecto;
- cancelar reservas confirmadas;
- recuperar procesos pendientes o interrumpidos.

Antes de iniciar una nueva operacion, debera obtener del servicio de catalogo la informacion vigente que necesite. Podra conservar en una reserva datos descriptivos historicos del profesional, pero no utilizarlos como catalogo vigente.

Cada proceso de reserva y cada reserva almacenada localmente deberan quedar asociados al usuario final que los inicio. Aunque el servicio central identifique las operaciones mediante el `groupId` tecnico del proyecto, el backend desarrollado por el alumno debera conservar y aplicar la propiedad por usuario.

## 5. Alcance funcional obligatorio

La aplicacion debera permitir:

1. registrar y autenticar a cada usuario final en el backend del proyecto;
2. registrar una vez la cuenta tecnica de integracion del proyecto ante el servicio de catedra mediante Postman o una herramienta equivalente;
3. autenticar la cuenta de integracion y recuperar la configuracion asignada;
4. obtener y almacenar localmente un snapshot completo del catalogo;
5. mantener el catalogo actualizado mediante Kafka y Redis;
6. detectar que no puede continuar una sincronizacion incremental y reconstruir su estado local;
7. buscar y filtrar profesionales usando la copia local;
8. construir y presentar turnos disponibles a partir de horarios semanales y ocupaciones actuales;
9. solicitar un bloqueo temporal para un turno seleccionado;
10. iniciar la confirmacion de la reserva mediante REST;
11. recibir la solicitud de informacion adicional por Kafka y enviar la respuesta correspondiente;
12. reflejar localmente el resultado final del proceso;
13. permitir que cada usuario consulte sus propias reservas;
14. permitir que cada usuario cancele una reserva propia confirmada;
15. informar errores de integracion y recuperarse sin corromper el estado local.

## 6. Sincronizacion del catalogo

### 6.1. Sincronizacion completa

La aplicacion debera obtener un snapshot REST cuando inicialice una base local vacia, cuando necesite reconstruirla o cuando no pueda continuar de manera segura desde su version actual.

La aplicacion debera aplicar el snapshot de forma consistente y registrar como version local exactamente la version que corresponde a los datos almacenados.

### 6.2. Sincronizacion incremental

Kafka notificara la disponibilidad de una nueva version. A partir de esa notificacion, la aplicacion consultara en Redis la metadata y los cambios posteriores a su version local.

Los cambios deberan aplicarse en orden. La version local solo podra avanzar cuando la unidad de trabajo correspondiente haya terminado correctamente. El procesamiento repetido de una misma notificacion o cambio no debera alterar el resultado final.

Si se detecta una discontinuidad, una version fuera de la ventana disponible o un estado local que no puede verificarse, la aplicacion debera ejecutar una sincronizacion completa.

Las busquedas de usuarios se resolveran sobre la copia local: no se aceptara consultar al servicio central por cada busqueda.

## 7. Disponibilidad y reserva de turnos

Para mostrar disponibilidad, el servicio de turnos debera combinar:

- las reglas semanales vigentes obtenidas del servicio de catalogo;
- las ocupaciones informadas por el servicio central;
- la fecha y el profesional elegidos por el usuario.

El proceso de reserva tendra el siguiente comportamiento observable:

1. el usuario selecciona un profesional, fecha y horario valido;
2. el servicio de turnos consulta la informacion necesaria y solicita un hold temporal;
3. la aplicacion confirma inicialmente el hold mediante REST;
4. recibe `AdditionalInformationRequested` por Kafka;
5. envia `AdditionalInformationSubmitted` al topic asignado;
6. recibe y procesa un resultado final por Kafka;
7. actualiza su estado local y permite consultar el resultado.

La solucion debera contemplar los resultados de confirmacion, rechazo de la informacion adicional, expiracion, proceso invalido y cancelacion. Un hold tiene duracion limitada: conservarlo localmente no extiende su vigencia en el servicio central.

El estado observado por REST y por Kafka puede llegar en momentos diferentes. La solucion debera converger sin generar reservas duplicadas ni retroceder un proceso que ya alcanzo un estado final.

## 8. Robustez y recuperacion

La solucion debera demostrar un comportamiento correcto frente a:

- recepcion duplicada de mensajes Kafka;
- repeticion de una operacion por timeout o respuesta perdida;
- interrupcion temporal de la conexion con el servicio central;
- indisponibilidad temporal de uno de los dos servicios propios;
- perdida de notificaciones de catalogo;
- vencimiento de un hold o de un proceso pendiente;
- reinicio de los servicios con operaciones pendientes;
- datos de entrada invalidos o referencias inexistentes.

Cada alumno debera definir timeouts, reintentos, idempotencia, persistencia de progreso y traduccion de errores. Las decisiones deberan evitar ciclos de reintento ilimitados y estados locales imposibles de recuperar.

Si el servicio de catalogo no esta disponible, no podran iniciarse operaciones que requieran validar informacion vigente. El servicio de turnos debera seguir atendiendo las operaciones que dependan solamente de datos propios, cuando resulte funcionalmente posible.

## 9. Seguridad

La solucion completa debera impedir el acceso anonimo o no autorizado a las funcionalidades protegidas. Como minimo, debera:

- permitir el registro y el inicio de sesion de los usuarios desde la interfaz KMP;
- proteger las contraseñas almacenadas mediante hashing adecuado y evitar exponerlas en respuestas o logs;
- emitir un JWT al autenticar correctamente al usuario;
- exigir y validar el JWT en las APIs protegidas consumidas durante la sesion;
- utilizar y validar JWT en la comunicacion protegida entre los dos servicios backend;
- autorizar cada operacion segun la identidad autenticada;
- impedir que un usuario consulte, modifique o cancele procesos y reservas pertenecientes a otro usuario;
- proteger los endpoints que no sean expresamente publicos;
- evitar confiar en identificadores de usuario o de integracion tecnica enviados libremente por el cliente;
- proteger la comunicacion entre la interfaz y los servicios backend;
- autenticar o validar la comunicacion entre ambos servicios;
- externalizar credenciales, tokens y secretos;
- evitar exponer secretos en el codigo fuente, respuestas innecesarias o logs;
- configurar CORS de acuerdo con los clientes permitidos;
- validar los datos recibidos en cada limite de la aplicacion;
- documentar las decisiones y limitaciones de seguridad.

El alumno debera definir si la comunicacion entre servicios propaga el JWT del usuario o utiliza un JWT tecnico, y justificar como preserva la identidad, la autorizacion y la trazabilidad necesarias. En todos los casos debera validar firma, vigencia y permisos del token recibido.

De manera opcional, el alumno podra implementar un rol administrador capaz de consultar o gestionar reservas de otros usuarios. En ese caso debera documentar sus permisos y demostrar que las operaciones administrativas estan correctamente autorizadas. Este rol no reemplaza el aislamiento obligatorio entre usuarios comunes.

## 10. Decisiones de diseño a cargo del alumno

La catedra define los contratos externos y los comportamientos evaluables, pero no la arquitectura interna completa de la solucion. Cada alumno debera decidir y justificar:

- clases, entidades y tablas internas;
- motor o motores de base de datos servidor;
- contratos entre sus dos servicios;
- estrategia transaccional de sincronizacion;
- mecanismo de idempotencia;
- manejo de mensajes duplicados o fuera de orden;
- timeouts, reintentos y tratamiento de indisponibilidad;
- maquina de estados local de las reservas;
- mecanismo de autenticacion entre servicios;
- organizacion de la interfaz de usuario;
- estrategia y alcance de las pruebas.

El anexo tecnico de integracion proporcionado por la catedra sera la fuente de verdad para los contratos REST, Redis y Kafka. Los ejemplos conceptuales del enunciado no reemplazan esos contratos.

### 10.1. Pruebas automatizadas

Cada alumno debera implementar pruebas automatizadas de los servicios backend para los comportamientos fundamentales de la solucion. Como minimo, las pruebas deberan cubrir servicios propios vinculados con registro y autenticacion, sincronizacion, reservas, autorizacion e idempotencia.

No se exigira probar exhaustivamente todas las clases ni alcanzar un porcentaje determinado de cobertura. Las pruebas de la interfaz KMP y cualquier cobertura adicional seran opcionales y se consideraran trabajo complementario.

## 11. Evidencias minimas esperadas

La entrega debera permitir verificar, al menos:

- una inicializacion desde snapshot;
- una actualizacion incremental del catalogo;
- recuperacion mediante snapshot luego de una discontinuidad;
- busquedas resueltas sobre datos locales;
- funcionamiento de los filtros obligatorios por categoria, nombre, estado habilitado y disponibilidad;
- una reserva confirmada mediante el flujo REST y Kafka;
- una reserva que expire sin completar la informacion solicitada;
- consulta y cancelacion de una reserva propia, sin acceso cruzado entre usuarios comunes;
- procesamiento idempotente de un mensaje duplicado;
- separacion efectiva de datos entre los dos servicios;
- rechazo de accesos no autenticados o no autorizados;
- pruebas automatizadas de servicios backend para los comportamientos principales.

La modalidad exacta para presentar estas evidencias se indicara junto con las condiciones de entrega.

La catedra no proporcionara una suite adicional comun de pruebas. Si durante el seguimiento se solicita una verificacion adicional, se definira individualmente para el alumno correspondiente.

## 12. Documentacion obligatoria

Cada alumno debera entregar, como minimo:

- un `README` con requisitos, configuracion y pasos reproducibles para ejecutar la solucion;
- un diagrama de arquitectura que muestre la aplicacion Android, los dos servicios backend, sus bases de datos y las integraciones con el servicio de catedra;
- la documentacion de los contratos entre los dos servicios, incluyendo operaciones, DTO, autenticacion y errores relevantes;
- el modelo de datos de cada servicio y la delimitacion de su propiedad;
- una explicacion de la estrategia de sincronizacion completa e incremental;
- una explicacion del procesamiento idempotente y la recuperacion ante mensajes duplicados, perdidos o fuera de orden;
- una descripcion de las decisiones de seguridad, autenticacion y autorizacion;
- instrucciones para ejecutar las pruebas automatizadas obligatorias.

La documentacion debera coincidir con la solucion entregada. Se podran agregar ADR, diagramas de secuencia, colecciones de solicitudes u otras evidencias que ayuden a justificar las decisiones del alumno.

## 13. Seguimiento, regularizacion y presentacion final

### 13.1. Acompañamiento individual

Cada alumno sera asignado a uno de los profesores, quien lo guiara de forma individual durante el desarrollo. Se realizaran reuniones de seguimiento cada dos semanas para revisar el avance, la comprension de las decisiones y la participacion efectiva en el proyecto.

El codigo y la documentacion se organizaran en tres repositorios Git separados y compartidos con el profesor asignado:

1. aplicacion KMP para Android;
2. servicio de catalogo y sincronizacion;
3. servicio de turnos y reservas.

El historial debera conservar la evolucion del trabajo y permitir identificar las contribuciones de cada alumno. No se aceptara como unica evidencia una carga final sin historial de desarrollo.

La entrega final se realizara durante los ultimos dias de noviembre. La catedra comunicara el dia y horario exactos. Las entregas parciales y los objetivos de seguimiento se definiran individualmente para cada alumno durante las reuniones quincenales.

### 13.2. Regularizacion

Durante el proceso, el profesor asignado analizara si el alumno se encuentra en condiciones de regularizar. Esta decision considerara el cumplimiento de los requisitos, la evolucion observada, las contribuciones registradas y la capacidad del alumno para explicar el trabajo realizado.

La regularizacion es un requisito previo para acceder a la presentacion final ante ambos profesores.

### 13.3. Presentacion final

Una vez regularizado, se pactara con el alumno una reunion remota ante los dos profesores. En esa instancia presentara y defendera el proyecto con la camara encendida.

Aunque existan responsabilidades distribuidas, cada alumno debera comprender las decisiones principales y la integracion entre todos los componentes de la solucion.

## 14. Fuera de alcance

No se requiere implementar:

- historias clinicas;
- recetas o diagnosticos;
- obras sociales;
- facturacion o pagos;
- informacion medica real;
- videollamadas;
- gestion hospitalaria completa;
- integraciones con instituciones externas.

## 15. Informacion que completara la catedra antes de publicar

Esta seccion no forma parte de los requisitos funcionales y debera resolverse antes de entregar el enunciado a los alumnos:

- dia y horario exactos de la entrega final de noviembre;
