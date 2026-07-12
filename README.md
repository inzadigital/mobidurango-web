# mobidurango-web

Web pública y calendario de actividades del servicio **Mobidurango** (Durangoko Udala — Ayuntamiento de Durango), un servicio gratuito de orientación en actividad física para la población de Durango y Durangaldea.

**🌐 En producción:** [mobidurango.kirolkudeaketa.com](https://mobidurango.kirolkudeaketa.com)

---

## Estructura del repositorio

```
mobidurango-web/
├── index.html              · Página principal en euskera (idioma institucional por defecto)
├── mobidurango-es.html     · Página principal en castellano
├── egutegia.html           · Calendario dinámico bilingüe (lista + mes)
├── admin.html              · Panel de gestión de eventos (acceso restringido)
└── img/                    · Fotografías del equipo, actividades y logos institucionales
```

Las dos páginas principales (`index.html` y `mobidurango-es.html`) son estáticas y comparten estructura, CSS y clases — solo cambia el contenido textual. Editar una requiere replicar el cambio en la otra.

El calendario (`egutegia.html`) es una página única bilingüe: alterna idioma con JavaScript sin recargar.

---

## Stack técnico

| Componente | Tecnología |
|---|---|
| Páginas principales | HTML5 + CSS3 puro, sin frameworks |
| Calendario dinámico | HTML + JavaScript vanilla, `fetch` contra Supabase REST |
| Panel de gestión | HTML + JavaScript vanilla, `supabase-js` v2 |
| Base de datos | Supabase (PostgreSQL gestionado) |
| Autenticación | Supabase Auth — email + contraseña |
| Tipografías | Google Fonts (Archivo Black + Inter) |
| Hosting | Vercel, despliegue automático desde `main` |
| Dominio | mobidurango.kirolkudeaketa.com |

---

## Calendario (`egutegia.html`)

El calendario lee la tabla `eventos` de Supabase mediante la REST API pública, usando la clave `anon` (publishable), en modo solo lectura. Los eventos se gestionan desde `admin.html` (ver sección siguiente) o, si hace falta, directamente desde el dashboard de Supabase.

### Filtros de visibilidad

Para que un evento aparezca en la web debe cumplir las tres condiciones:

- `published = true`
- `estado = 'publicado'`
- `tipo = 'puntual'`

Cambiar uno solo de estos campos oculta el evento sin borrarlo del histórico.

### Campos relevantes de la tabla `eventos`

| Campo | Tipo | Uso |
|---|---|---|
| `titulo_eu`, `titulo_es` | text | Título bilingüe |
| `detalle_eu`, `detalle_es` | text | Descripción ampliada (opcional) |
| `fecha_inicio`, `fecha_fin` | timestamptz | Fecha y hora; `fecha_fin` opcional |
| `lugar` | text | Lugar concreto (texto libre) |
| `poblacion` | text | Slug de población: `durango`, `abadino`, `atxondo`, `berriz`, `elorrio`, `ermua`, `garai`, `iurreta`, `izurtza`, `mallabia`, `manaria`, `zaldibar`, `otros` |
| `organizador` | text | Texto libre — los chips se generan automáticamente |
| `es_demo` | boolean | Si hay algún evento con `true`, aparece el banner amarillo "PROBA · DEMO" |
| `published`, `estado`, `tipo` | — | Filtros de visibilidad (ver arriba) |

### Funcionalidad del calendario

- **Dos vistas:** "Zerrenda" (lista, solo eventos futuros) y "Egutegia" (mes, incluye pasados navegables con flechas).
- **Filtros tipo chip** de población y organizador, construidos dinámicamente con los valores que aparecen en los datos.
- **Toggle de idioma EU/ES** con persistencia en `localStorage` (clave `mobidu_idioma`).
- **Persistencia de vista** preferida (clave `mobidu_vista`).
- **Compartir** cada evento por WhatsApp o copiando al portapapeles, con texto formateado y enlace.
- **Modal de día** al pulsar una celda con eventos en la vista mensual.
- **Banner DEMO** automático si hay eventos con `es_demo = true`.

### URL del sitio en compartidos

La constante `SITIO_URL` al inicio del bloque `<script>` (línea ~920) es la única fuente de verdad del dominio público dentro de `egutegia.html`. Si se cambia el dominio, basta con actualizar esa línea.

---

## Panel de administración (`admin.html`)

Página de gestión de eventos, no indexable (`meta robots: noindex, nofollow`), pensada para un círculo cerrado de personas de confianza: Durango Kirolak (rol `admin`) y Athlon (rol `editor_mobidurango`). No hay flujo de aprobación: cualquiera de los dos roles puede crear, editar, publicar y cancelar eventos de forma autónoma, sin depender del otro.

### Autenticación

- Login con **email + contraseña** (Supabase Auth). No se usa magic link: el servicio de email gratuito de Supabase tiene un límite muy bajo de envíos por proyecto (no por persona), y con el uso normal del panel se agota enseguida.
- Un usuario solo entra al panel si, además de tener sesión válida en Supabase Auth, existe como fila **activa** en `public.admin_users`. Tener cuenta de Auth no es suficiente por sí solo.
- **Alta de una persona nueva:**
  1. Supabase → Authentication → Users → *Invite user* (esto crea la fila en `auth.users` y genera su `id`).
  2. Insertar ese mismo `id` en `public.admin_users` con el `rol` correspondiente (ver tabla abajo).
  3. Fijar la contraseña **por SQL**, para no depender del límite de envío de correos:
     ```sql
     update auth.users
     set encrypted_password = crypt('CONTRASEÑA_AQUI', gen_salt('bf')),
         updated_at = now()
     where email = 'email@dominio.eus';
     ```
     (requiere la extensión `pgcrypto`, ya activa en el proyecto). Esto usa el mismo cifrado bcrypt que usa Supabase Auth por debajo, así que el login funciona igual que si se hubiera puesto por el formulario de recuperación de contraseña.
  4. Compartir la contraseña por un canal que no sea el propio email en texto plano si es posible (llamada, WhatsApp, gestor de contraseñas).

### Roles (tabla `public.admin_users`)

| Columna | Uso |
|---|---|
| `id` | Mismo UUID que `auth.users.id` |
| `email`, `nombre` | Identificación de la persona/cuenta |
| `rol` | `admin` \| `editor_mobidurango` \| `editor_asociacion` (este último reservado, sin usar todavía) |
| `activo` | Desactivar acceso sin borrar el historial: poner a `false` |
| `asociacion_nombre` | Reservado para cuando se abra el panel a más asociaciones (rol `editor_asociacion`) |

Las funciones `is_admin()` e `is_editor()` (SQL, en el propio proyecto Supabase) consultan esta tabla y sostienen todas las políticas RLS de `eventos`.

### Permisos sobre `eventos` (RLS)

- **INSERT / UPDATE**: `admin` o `editor_mobidurango`, sin restricción de `created_by` — cualquiera de los dos roles puede tocar eventos creados por cualquier otra persona del mismo círculo.
- **DELETE** (borrado físico, irreversible): reservado solo a `admin`. Es la única acción que Athlon no tiene.
- **"Eliminar" desde el panel** no es un `DELETE`: es un `UPDATE` que pone `estado = 'cancelado'`. El evento desaparece de la web pública (no cumple `estado = 'publicado'`) pero sigue existiendo y es reversible desde el filtro "Bertan behera utzitakoak" / "Cancelados" del propio panel.

### Estados de `eventos.estado`

`borrador` · `pendiente_aprobacion` (heredado de un diseño anterior con aprobación, sin uso actual) · `publicado` · `rechazado` (heredado, sin uso actual) · `cancelado` (borrado suave, usado por el panel)

---

## Despliegue

El repositorio está conectado a Vercel: cada push a la rama `main` lanza un deploy automático. No hay paso de build — son archivos estáticos.

---

## Mantenimiento

### Cambios de contenido en las páginas principales

Editar `index.html` (EU) y `mobidurango-es.html` (ES). Ambos archivos deben mantenerse sincronizados en estructura. Los IDs de sección son iguales en los dos idiomas para facilitar el mantenimiento paralelo.

### Cambios en imágenes

Sustituir los archivos en `img/` manteniendo el nombre. Los HTML referencian rutas relativas (`img/pello.jpg`, etc.).

### Cambios en el calendario

- **Añadir/editar/cancelar eventos:** desde `admin.html`, con cuenta de rol `admin` o `editor_mobidurango`. También es posible seguir haciéndolo directamente desde el dashboard de Supabase si hace falta.
- **Despublicar un evento:** desde el panel, botón "Cancelar" (pone `estado = 'cancelado'`), o manualmente cambiando `published` a `false` / `estado` a algo distinto de `'publicado'`.
- **Añadir una nueva población:** insertar el slug en el `POBLACIONES_MAP` de `egutegia.html` (y en el `<select>` de `admin.html`) con su nombre formateado (tildes, eñes), y en el `check constraint` `eventos_poblacion_check` de la tabla.

---

## Pendientes conocidos

- Limpieza de los eventos de demostración (`es_demo = true`) cuando se disponga de datos reales suficientes.
- Integración con `durangokirolak.net` vía OMESA (auto-altura del iframe + detección de embebido).
- Página institucional del proyecto (`mobidurango-proiektua.html`) — diseñada pero no integrada en este repositorio.
- Rol `editor_asociacion` preparado en base de datos pero sin usar — pendiente de decisión sobre abrir el panel a más asociaciones.

---

## Licencia y propiedad

Servicio del **Ayuntamiento de Durango** · Durangoko Udala
© 2026 Mobidurango
