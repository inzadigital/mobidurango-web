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
| Base de datos | Supabase (PostgreSQL gestionado) |
| Tipografías | Google Fonts (Archivo Black + Inter) |
| Hosting | Vercel, despliegue automático desde `main` |
| Dominio | mobidurango.kirolkudeaketa.com |

---

## Calendario (`egutegia.html`)

El calendario lee la tabla `eventos` de Supabase mediante la REST API pública, usando la clave `anon` (publishable). No tiene panel de administración web: los eventos se gestionan directamente desde el dashboard de Supabase.

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

## Despliegue

El repositorio está conectado a Vercel: cada push a la rama `main` lanza un deploy automático. No hay paso de build — son archivos estáticos.

---

## Mantenimiento

### Cambios de contenido en las páginas principales

Editar `index.html` (EU) y `mobidurango-es.html` (ES). Ambos archivos deben mantenerse sincronizados en estructura. Los IDs de sección son iguales en los dos idiomas para facilitar el mantenimiento paralelo.

### Cambios en imágenes

Sustituir los archivos en `img/` manteniendo el nombre. Los HTML referencian rutas relativas (`img/pello.jpg`, etc.).

### Cambios en el calendario

- **Añadir/editar eventos:** desde el dashboard de Supabase, tabla `eventos`.
- **Despublicar un evento:** cambiar `published` a `false` o `estado` a algo distinto de `'publicado'`.
- **Añadir una nueva población:** insertar el slug en el `POBLACIONES_MAP` de `egutegia.html` con su nombre formateado (tildes, eñes).

---

## Pendientes conocidos

- Panel de administración web para gestionar eventos sin entrar a Supabase.
- Integración con `durangokirolak.net` vía OMESA.
- Página institucional del proyecto (`mobidurango-proiektua.html`) — diseñada pero no integrada en este repositorio.

---

## Licencia y propiedad

Servicio del **Ayuntamiento de Durango** · Durangoko Udala
© 2026 Mobidurango
