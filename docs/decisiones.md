# Decisiones de diseño

Cada decisión tiene: contexto, decisión, alternativas y motivo.

> **Nota:** la decisión 1 (uso personal, clave secreta única) y la 2 (SQLite "para un
> usuario") quedaron **reemplazadas** el 17 de octubre de 2026: la app pasó a ser
> multiusuario con 5 usuarios. Se conservan aquí como registro de por qué se cambió.

## 1. ~~Proyecto de uso personal~~ (reemplazada)

- **Contexto:** en el diseño inicial la app era solo para mí, no para otros usuarios.
- **Decisión original:** sin registro de usuarios; el acceso se protegía con una clave secreta única (`ACCESS_TOKEN`).
- **Por qué se reemplazó:** la app pasó a tener 5 usuarios. Una clave global obligaba a que
  todos compartieran el mismo secreto y, sobre todo, a que **cualquiera con el token viera los
  datos de todos**: no había forma de separar los datos de cada persona. La decisión original
  era correcta solo bajo el supuesto de un usuario, que ya no se cumple.
- **Reemplazada por:** las decisiones 6, 7 y 9.

## 2. ~~SQLite como base de datos~~ (reemplazada)

- **Contexto:** necesito guardar hábitos y registros diarios.
- **Decisión original:** usar SQLite.
- **Por qué se reemplaza:** no la base de datos, sino el **motivo**. La razón escrita era
  "sobra para un usuario". Sigue siendo cierta con 5: SQLite soporta 5 usuarios sin problema
  y con menos piezas que un servidor de base de datos. Lo que cambia es que ahora las tablas
  llevan `usuario_id` y cada consulta va filtrada por el usuario autenticado.
- **Reemplazada por:** la decisión 9.

## 3. Fechas guardadas como texto local

- **Contexto:** un hábito marcado a las 11 p. m. no debe contarse como del día siguiente.
- **Decisión:** guardar la fecha como texto `YYYY-MM-DD` ya resuelta en la zona horaria del
  usuario que marca (cada usuario tiene la suya; ver decisión 8).
- **Alternativas:** guardar instantes en UTC.
- **Motivo:** evita errores de cambio de día al calcular el progreso diario.

## 4. Archivar hábitos en vez de borrarlos

- **Contexto:** la montaña depende del historial.
- **Decisión:** al "borrar" un hábito se marca como inactivo.
- **Alternativas:** eliminarlo de la base de datos.
- **Motivo:** conservar el historial de semanas anteriores.

## 5. Cliente sin framework al inicio

- **Contexto:** estoy afianzando los fundamentos de la web.
- **Decisión:** HTML, CSS y JavaScript puros en el cliente.
- **Alternativas:** Angular, React.
- **Motivo:** entender bien lo básico primero; el cliente podrá reescribirse más adelante usando la misma API.

## 6. Login con usuario y contraseña, en círculo cerrado

- **Contexto:** la app pasó de 1 a 5 usuarios. Hay que separar los datos de cada persona y
  protegerlos, sin abrir un servicio público en internet.
- **Decisión:** cada usuario tiene una cuenta con **nombre de usuario y contraseña** (contraseña
  con hash, nunca en texto plano) e inicia sesión. El alta de cuentas es **solo por invitación**:
  no hay registro público. Las cuentas se crean desde la CLI del servidor, que genera una
  invitación de un solo uso con fecha de expiración; la persona invitada la acepta, elige su
  contraseña y queda activa. Una invitación usada o vencida se invalida.
- **Alternativas:** (a) una clave secreta por usuario, sin login; (b) signup abierto.
- **Motivo:** (a) evita el login pero obliga a repartir tokens por canales inseguros y no
  permite recuperar el acceso ni rotar una credencial filtrada; (b) convierte un proyecto
  personal en un servicio público, que hay que robustecer mucho más. Con login + invitación se
  cumple lo importante: **cada quien ve solo lo suyo** y la lista de usuarios está cerrada y
  conocida.

## 7. Sin roles: todos los usuarios son iguales

- **Contexto:** con 5 usuarios podría haber un administrador y cuatro usuarios normales, o un
  modelo de permisos más elaborado.
- **Decisión:** no hay roles ni permisos. **Todos los usuarios son equivalentes**: cada uno
  administra únicamente sus propios hábitos, registros, recordatorios y suscripciones push, y
  solo puede exportar sus propios datos. Nadie ve ni modifica los datos de otro.
- **Alternativas:** roles `admin` / `usuario`, con un administrador capaz de listar usuarios y
  exportar todo el contenido de la base.
- **Motivo:** con 5 personas de confianza no hay un problema de permisos que resolver, y un
  administrador con acceso a los datos ajenos es justamente el punto débil que conviene evitar.
  Quien administra las cuentas lo hace desde la CLI, no desde la app: así el control de acceso
  queda separado de los datos de cada usuario.

## 8. Zona horaria por usuario

- **Contexto:** la app la usan 5 personas que pueden no estar en `America/Bogota`. Con una zona
  horaria global, el "día" de cada usuario se calcularía con la zona equivocada y un hábito
  marcado a las 11 p. m. caería en el día equivocado.
- **Decisión:** cada usuario tiene su zona horaria en su perfil (`zona_horaria`, tipo IANA, por
  ejemplo `America/Bogota`). El cliente envía la fecha ya resuelta en la zona del usuario; el
  servidor la usa para el progreso y para programar recordatorios.
- **Alternativas:** (a) una zona horaria global fija; (b) guardar instantes en UTC y convertir al
  presentar.
- **Motivo:** (a) es incorrecto para quien no comparta zona; (b) reintroduce el error de cambio
  de día que motivó la decisión 3. Guardar el texto local ya resuelto, junto a la zona del
  usuario, es simple y correcto.

## 9. Aislamiento por `usuario_id` en cada consulta

- **Contexto:** con una sola base de datos compartida por 5 usuarios, un `WHERE` olvidado es
  una fuga de datos entre personas.
- **Decisión:** `habitos` y `suscripciones_push` llevan `usuario_id`; `registros` lo heredan a
  través de `habito_id`, y toda consulta sobre ellos entra por un `JOIN` a `habitos`. El
  middleware de autenticación deja el usuario en `req.usuario` y **ningún handler recibe un
  `usuario_id` del cliente**: siempre sale de la sesión, nunca de un parámetro de la petición.
  Los recursos de otra persona se responden `404`, no `403`, para no confirmar que existen.
- **Alternativas:** una base de datos por usuario; `usuario_id` también en `registros`.
- **Motivo:** una base por usuario multiplica la operación por 5 sin ganancia a esta escala, y
  `usuario_id` en `registros` sería redundante con `habito_id` (un hábito no cambia de dueño).
  La garantía se apoya en tres capas: el modelo de datos, el middleware, y pruebas que
  intentan leer datos ajenos y deben fallar.