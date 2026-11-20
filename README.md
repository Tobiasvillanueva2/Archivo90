# dept.archivo — Tienda + Panel (Etapas 1 y 2)

Tienda single-page de **dept.archivo** (Salta, Est. 2026) construida sobre la arquitectura de T.E.S Joyas (V12):

```
TIENDA (index.php renderizado en servidor + assets/js/app.js)
        │  fetch JSON
        ▼
API REST PHP (/api)  ←  única puerta de entrada a los datos (igual que TES)
        ▲
PANEL /admin (admin/index.php + admin/assets/admin.js) ← usa la misma API
        │  PDO, conexión única
        ▼
MySQL (Hostinger) — base NUEVA, separada de TES
```

## Etapa 1 — Tienda

- **Tienda single-page** según el brief: header fijo con contador de bolsa, carrusel editorial, manifiesto, catálogo 3:4 con `DA-001`, precio en ARS y bio, y bolsa lateral.
- **Mejoras sobre el HTML de referencia**:
  - Ficha de producto con URL propia (`/p/slug`), tabla de medidas en cm por talle, compartir y consulta por WhatsApp.
  - Filtro por talle disponible y opción de ocultar las piezas archivadas.
  - Drops programados con cuenta regresiva (`productos.fecha_publicacion`).
  - Bolsa en 3 pasos: bolsa → datos → WhatsApp. **El pedido se registra en la base y reserva el stock** antes de abrir WhatsApp, así una pieza 1 de 1 nunca se vende dos veces (probado con 6 compras simultáneas: se concreta 1).
  - Talles agotados tachados, aviso de "pieza única" y avisos cuando algo se vende mientras el cliente mira.
  - Lista de aviso de drops (newsletter).
- **SEO**:
  - El HTML llega renderizado desde el servidor: Google ve los productos sin ejecutar JS.
  - Title, description, canonical y Open Graph por producto.
  - Datos estructurados schema.org: `ClothingStore`, `Product` con precio, disponibilidad, marca y condición (nuevo/usado).
  - `sitemap.xml` (con imágenes) y `robots.txt` dinámicos.
- **Analítica igual que TES**: GTM → GA4 + Clarity. Eventos GA4 de e-commerce completos: `view_item_list`, `view_item`, `add_to_cart`, `remove_from_cart`, `view_cart`, `begin_checkout`, `add_shipping_info`, `add_payment_info`, `purchase`. Además los propios `whatsapp_click`, `arrepentimiento_enviado`, `sign_up` y `share`.
- **Legal Argentina**:
  - **Botón de arrepentimiento** (Res. 424/2020 · Disp. 954/2025): visible en el footer y en `/#arrepentimiento`, sin registro, con código de seguimiento al instante.
  - **"Precio sin impuestos nacionales"** (Ley 27.743) en tarjetas y ficha.
- **Precios como TES**: se guardan en **USD** y se muestran en **ARS** con la cotización de DolarApi (cache 1 h, override manual y redondeo configurable). Si la API falla, sigue con el último valor.

## Seguridad

- PDO con prepared statements en el 100% de las consultas.
- CSRF en todo POST, honeypot anti-bots y límites anti-abuso: 5 pedidos por hora por IP y 2 pendientes por teléfono.
- La IP se guarda solo como hash SHA-256.
- Sin CORS abierto (TES tenía `*`).
- La API responde con `no-store`, así el stock y los precios nunca quedan cacheados.
- `.htaccess` bloquea `/config`, `/database`, `.sql`, `.md` y `.ini`.
- `/uploads` no ejecuta PHP.
- Cabeceras de seguridad configuradas.
- `.user.ini` con `session.upload_progress.enabled=0`: mitiga CVE-2026-115560, que todavía no tiene parche.
- **Usar PHP 8.4 actualizado** (≥ 8.4.27, parche de PHP2Shell: CVE-2026-109618/109619). PHP 8.2 deja de tener soporte en diciembre de 2026.

---

## Base de datos (`database/instalacion-completa.sql`)

Reglas: el precio solo existe en USD, el stock solo vive en las variantes y todo atributo tipo categoría tiene su propia tabla. Los totales del pedido no se guardan: los calcula la vista `v_pedidos_totales`.

**Excepción deliberada:** `pedido_items` guarda una "foto" del producto, el precio y la cotización del momento de la compra. Sin eso, el historial cambiaría al editar un precio o al moverse el dólar.

| Tabla | Columnas |
|---|---|
| `categorias` | id, padre_id, nombre, slug, descripcion, imagen, orden, estado |
| `colecciones` (cápsulas) | id, nombre, slug, descripcion, imagen, orden, estado |
| `marcas` | id, nombre, orden, estado |
| `condiciones` | id, nombre, descripcion, schema_org, orden, estado |
| `etiquetas` | id, nombre, orden, estado |
| `talles` | id, nombre, orden, estado |
| `medidas` | id, nombre (Pecho, Largo, Manga, Cintura, Tiro), orden, estado |
| `productos` | id, codigo, categoria_id, coleccion_id, marca_id, condicion_id, etiqueta_id, nombre, slug, temporada, descripcion_corta (bio), descripcion_larga, precio_usd, seo_titulo, seo_descripcion, destacado, estado, fecha_publicacion, orden, fecha_creacion, fecha_actualizacion |
| `producto_imagenes` | id, producto_id, imagen, alt, orden |
| `producto_variantes` | id, producto_id, talle_id, sku, stock (UNSIGNED: nunca negativo), precio_usd (override opcional), estado |
| `variante_medidas` | variante_id, medida_id, valor_cm |
| `slides` | id, titulo_1, titulo_2, texto, cta_texto, cta_destino, imagen, imagen_movil, alt, foco, orden, estado |
| `configuracion` | id, clave, valor, grupo |
| `clientes`, `clientes_direcciones`, `verificaciones`, `clientes_tokens`, `favoritos` | iguales a TES |
| `carrito` | id, cliente_id, fecha_creacion, fecha_actualizacion |
| `carrito_items` | id, carrito_id, variante_id, cantidad, fecha_agregado |
| `metodos_entrega` | id, codigo, nombre, descripcion, requiere_direccion, costo_ars (NULL = a coordinar), orden, estado |
| `medios_pago` | id, codigo, nombre, instrucciones, descuento_pct, orden, estado |
| `pedidos` | id, token, cliente_id (NULL = invitado), estado, canal, contacto_nombre, contacto_telefono, contacto_email, metodo_entrega_id, medio_pago_id, envio_calle, envio_numero, envio_piso_depto, envio_ciudad, envio_provincia, envio_codigo_postal, envio_referencias, envio_seguimiento, cotizacion_usd_ars, descuento_pct, envio_ars, notas_cliente, notas_internas, reserva_expira, ip_hash, fecha_creacion, fecha_actualizacion |
| `pedido_items` | id, pedido_id, variante_id, producto_codigo, producto_nombre, talle, precio_usd, precio_ars, cantidad |
| `pedido_estados` (historial) | id, pedido_id, estado, usuario_id, nota, fecha |
| `pagos` | id, pedido_id, medio_pago_id, monto_ars, estado, referencia_externa, comprobante, nota, fecha_creacion, fecha_actualizacion |
| `arrepentimientos` | id, codigo, pedido_id, numero_pedido, nombre, email, telefono, motivo, estado, ip_hash, fecha_creacion, fecha_actualizacion |
| `suscriptores` | id, email, origen, estado, ip_hash, fecha_creacion |
| `usuarios` (panel) | igual a TES |
| `login_intentos` (panel) | id, ip_hash, usuario, exito, fecha |

**Nunca se guardan datos de tarjeta:** `pagos.referencia_externa` guarda solo el ID que devuelve la pasarela.
 
---

## Instalación en Hostinger (paso a paso)

> dept.archivo va en su **propio sitio y su propia base**. No reutilizar la base de TES.

1. **PHP**: hPanel → Avanzado → Configuración PHP → versión **8.4**.
2. **Base de datos**: hPanel → Bases de datos → MySQL → crear base y usuario nuevos. Anotá nombre, usuario y contraseña.
3. **Importar**: phpMyAdmin → seleccionar esa base → pestaña *Importar* → `database/instalacion-completa.sql`.
   - Crea 30 tablas y 1 vista, con los 6 productos de la Cápsula 01.
   - ⚠ Si algún día la base ya tiene datos reales: **exportala antes**, porque este archivo hace DROP TABLE.
4. **Subir archivos**: Administrador de archivos → `public_html/`. Subí todo el contenido de la carpeta.
   - Si existe un `index.html` viejo, borralo o renombralo: `index.php` tiene que ser la portada.
   - Activá "Mostrar archivos ocultos" para verificar que subieron `.htaccess` y `.user.ini`.
5. **Configurar** `config/config.php` (desde el editor del Administrador de archivos):
   - `DB_NAME`, `DB_USER` y `DB_PASS` con los datos del paso 2.
   - `APP_SECRET`: cualquier texto largo al azar.
   - ⚠ Una vez configurado en producción, **no volver a sobrescribir este archivo** en futuras entregas.
6. **SSL**: hPanel → Seguridad → SSL → activar y tildar **Forzar HTTPS**.
7. **Datos reales** (phpMyAdmin → tabla `configuracion` → editar `valor`):
   - `whatsapp`: número con código de país, sin + ni espacios (`549387…`).
   - `instagram`.
   - `gtm_id`: `GTM-XXXXXXX`. Mientras esté vacío no se carga ningún script de seguimiento.
   - `tipo_cambio_casa`: blue, oficial, tarjeta, etc. Para fijar la cotización a mano: `tipo_cambio_modo` = `manual` y el valor en `tipo_cambio`.
   - Alias/CBU: en `medios_pago.instrucciones`.
8. **Verificar**:
   - `https://tudominio.com/api/productos` responde JSON.
   - Abrí la home, después `/p/corinthians-2025-2026-local` y `/sitemap.xml`.
   - `https://tudominio.com/config/config.php` tiene que dar **403**.
9. **Google Search Console**: agregar el dominio y enviar `https://tudominio.com/sitemap.xml`.

### GTM (para que no se repita lo de Clarity en TES)

1. Crear un **contenedor nuevo** para este dominio.
2. Etiqueta GA4 *Configuración de Google* con activador *All Pages*.
3. Etiqueta Microsoft Clarity desde la Galería de plantillas de la comunidad, con activador *All Pages*.
4. Para los eventos de e-commerce: etiqueta *Evento GA4* con `{{Event}}` y la opción "Enviar datos de comercio electrónico", con activador de *Evento personalizado* para el regex `view_item_list|view_item|add_to_cart|remove_from_cart|view_cart|begin_checkout|add_shipping_info|add_payment_info|purchase`.
5. **Publicar** el contenedor (botón Enviar → Publicar) y verificar con Tag Assistant.

---

## Etapa 2 — Panel (`/admin`)

Entrá a `https://tudominio.com/admin` con usuario **admin** y contraseña **DeptArchivo2026!**. Cambiala apenas entres: está en el menú → *Cambiar contraseña* (mínimo 10 caracteres).

En el celular: abrí `/admin` en Chrome o Safari → *Agregar a la pantalla de inicio*. Queda como una app con ícono propio.

**Qué hay en el panel**

- **Resumen**:
  - Ventas del mes y comparación con el mes anterior, gráfico de los últimos 30 días y pedidos en curso.
  - Avisos de lo urgente: reservas por vencer, pedidos para preparar, arrepentimientos sin responder y productos sin foto.
  - Cotización del día.
- **Pedidos**:
  - Filtros por estado y buscador por número, nombre, teléfono o email.
  - Ficha con acciones según el estado: confirmar, marcar pagado, enviado o entregado, cancelar, registrar devolución y reactivar un vencido.
  - **El stock se repone o se vuelve a reservar solo.**
  - Registro de pagos: cuando se cubre el total, el pedido pasa a *pagado* solo.
  - Costo de envío editable, código de seguimiento y notas internas que se guardan solas.
  - Historial con quién hizo cada cambio y etiqueta de envío para imprimir.
  - **Mensajes rápidos de WhatsApp** al cliente (confirmar, pago recibido, envío) con nombre, número de pedido y total ya completos.
  - Exportación a CSV para Excel.
  - Aviso automático de pedidos nuevos (el panel consulta cada minuto).
- **Productos**:
  - Listado con fotos, stock y unidades reservadas. Arrastrando las filas se cambia el orden de la tienda; lo nuevo aparece primero.
  - Ficha con código sugerido, URL automática, precio en USD con vista del precio en pesos de hoy.
  - Talles con stock, medidas en cm por talle y precio distinto por talle (opcional). Atajos: *Pieza única* y *Curva S·M·L·XL*.
  - Fotos: arrastrar para subir y ordenar. Se convierten solas a WebP con una versión de 600 px para celular, y se corrige la rotación de las fotos de celular.
  - Drop programado y vista previa de cómo se ve en Google.
  - Duplicar (queda como borrador, con stock en 0) y eliminar.
  - Si entra un pedido mientras editás el stock, el panel avisa en vez de pisarlo.
  - Atajo **Ctrl/⌘ + S** para guardar. Si intentás salir con cambios sin guardar, pregunta antes.
- **Catálogo**: categorías, cápsulas, marcas, talles, medidas, condiciones y etiquetas.
  - Se edita en la misma fila y se ordena arrastrando.
  - Lo que está en uso no se puede borrar: se desactiva.
- **Contenido**:
  - Carrusel con imagen para compu y celular, y **punto de foco** (tocás la foto y elegís qué parte nunca se corta). Muestra cómo queda en compu y en celular.
  - Todos los textos de la marca.
- **Ajustes**:
  - WhatsApp, Instagram y GTM.
  - Cotización (automática o manual, y qué dólar usar), redondeo, IVA y precio sin impuestos.
  - Horas de reserva y máximo de unidades por pedido.
  - Métodos de entrega y medios de pago (con descuento e instrucciones).
  - Plantillas de WhatsApp.
- **Arrepentimientos y suscriptores**: gestión de estados, exportación de suscriptores a CSV y copiar emails.

**Seguridad del panel**

- Contraseñas con bcrypt.
- **Bloqueo de 15 minutos tras 5 intentos fallidos.**
- La sesión vence tras 4 h sin uso.
- CSRF en toda acción que modifica datos.
- El panel no se indexa en Google.
- Toda foto se re-codifica: un archivo con código escondido no sobrevive.
- Los datos que escriben los clientes se muestran siempre como texto (probado con `<script>` en nombre y notas).
- El CSV neutraliza fórmulas de Excel.

**¿Te olvidaste la contraseña?** En phpMyAdmin → SQL, ejecutá:

```sql
UPDATE usuarios SET password_hash = '$2y$10$sJkV0uOoTpa40UM2skO/oeN33UiFJDX2/5egWJ22hEUMuxrJ3f1Xi' WHERE usuario = 'admin';
DELETE FROM login_intentos;
```

Vuelve a ser `DeptArchivo2026!`. Cambiala apenas entres.

### Actualizar a la Etapa 2 (si ya tenés la Etapa 1 instalada)

1. **Backup**: phpMyAdmin → base de dept.archivo → *Exportar* → *Continuar*. Guardá el `.sql`.
2. **Base**: pestaña *SQL* o *Importar* → `database/actualizacion-etapa2.sql`.
   - No borra nada y se puede correr más de una vez.
   - Agrega `login_intentos`, la columna `pedidos.envio_seguimiento` y las plantillas de WhatsApp.
   - ⚠ **No** uses `instalacion-completa.sql` sobre una base con datos reales.
3. **Archivos**: subí todo el contenido del zip a `public_html/` **menos `config/config.php`**, que ya tiene tus credenciales y no hay que pisarlo.
   - Nuevo: carpeta `admin/` completa, 4 controladores y 4 modelos en `api/`, `api/services/ImageUpload.php` y `database/actualizacion-etapa2.sql`.
   - Cambian: `.htaccess`, `index.php`, `router.php`, `api/index.php`, `api/routes/routes.php`, `api/services/Auth.php`, `api/services/CatalogoService.php` y `assets/js/app.js`.
4. **Fotos pesadas**: si al subir fotos del celular dice que superan el tamaño, en hPanel → Avanzado → Configuración PHP subí `upload_max_filesize` a 8M y `post_max_size` a 40M. El `.user.ini` ya lo pide, pero algunos planes lo ignoran.
5. Si no ves cambios: hPanel → *Borrar caché* (LiteSpeed). Los archivos del panel se versionan solos.

## Desarrollo local

```bash
php -S localhost:8000 router.php   # router.php emula el .htaccess (en Hostinger está bloqueado)
```
Credenciales locales en `config/config.local.php` (no se sube).

Pruebas (todas OK): tienda 56 de API + 39 de recorrido en Chrome; panel 84 de API + 48 de recorrido en Chrome (escritorio y celular).

## Próximas etapas

- **Etapa 3 — Cuentas de clientes** (base: `cuenta.html` de TES):
  - Registro con verificación por email, direcciones y favoritos.
  - Carrito sincronizado al iniciar sesión e historial de compras.
  - Los pedidos como invitado se vinculan por email.
- **Etapa 4**: Mercado Pago (tabla `pagos` ya preparada) y envíos.
