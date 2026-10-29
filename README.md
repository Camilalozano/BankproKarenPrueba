# Buscador de Convocatorias — FundsForNGOs Premium + TripleFunds

App de Streamlit que reemplaza el notebook `scraper_final_v2.ipynb` con una
interfaz web sencilla: un selector de plataforma (FundsForNGOs o
TripleFunds), filtros en la barra lateral según la plataforma elegida,
extracción completa con barra de progreso, evaluación y priorización CTI,
y descarga directa en Excel. Un solo enlace, un solo despliegue: la app
entera vive en un único repositorio y un único proyecto de Streamlit
Community Cloud — el selector solo cambia qué cliente y qué filtros se
usan por dentro, ambos por dentro producen los mismos datos.

## Estructura

```
ffn_streamlit/
├── app.py                          # Interfaz (Streamlit) — selector de plataforma + filtros + CTI
├── ffn_client.py                   # FundsForNGOs: login, búsqueda, detalle y export a Excel
├── triple_funds_client.py          # TripleFunds: login, búsqueda (HTML) y detalle — ver su sección abajo
├── probar_triplefunds.py           # Script de diagnóstico para correr localmente contra TripleFunds real
├── cti_scoring.py                  # Algoritmo de evaluación y priorización CTI (v5.3) — compartido por ambas plataformas
├── requirements.txt
├── .gitignore
├── assets/
│   └── logo_atenea.png             # Logo de Agencia Atenea, usado en el encabezado y como favicon
└── .streamlit/
    ├── config.toml                 # Tema visual (colores de marca de Atenea)
    └── secrets.toml.example        # Plantilla — NO subir la versión real
```

## Desplegar en Streamlit Community Cloud

1. **Sube esta carpeta a un repositorio de GitHub** (puede ser privado).
   No incluyas nunca un archivo `secrets.toml` real en el repo — el
   `.gitignore` ya lo excluye.

2. Entra a [share.streamlit.io](https://share.streamlit.io), conecta tu
   cuenta de GitHub y selecciona el repositorio. Como archivo principal
   indica `app.py`.

3. Antes de desplegar (o después, en **App settings → Secrets**), pega lo
   siguiente reemplazando los valores reales:

   ```toml
   APP_PASSWORD = "contraseña-que-vas-a-compartir-con-el-equipo"
   FFN_EMAIL = "correo-de-la-cuenta-compartida-de-FundsForNGOs"
   FFN_PASSWORD = "contraseña-de-la-cuenta-de-FundsForNGOs"
   TF_COOKIES = "pega-aquí-el-encabezado-Cookie-completo (ver sección TripleFunds más abajo)"
   TF_USER_AGENT = "pega-aquí-el-User-Agent-de-ese-mismo-navegador"
   ```

   - `APP_PASSWORD` es la contraseña que pedirá la app antes de mostrar
     el selector de plataforma y los filtros (protege que gente ajena use
     las cuentas compartidas).
   - `FFN_EMAIL` / `FFN_PASSWORD` son las credenciales de la cuenta
     **compartida** de FundsForNGOs.
   - `TF_COOKIES` no son credenciales sino las cookies de una sesión de
     TripleFunds ya iniciada a mano (ver la sección "TripleFunds — login
     con cookies manuales" más abajo — es el único método que funciona,
     porque el login automático está bloqueado por Cloudflare). Si
     todavía no vas a usar TripleFunds, puedes omitir esta línea por
     ahora — la app solo la pide en el momento en que alguien elige esa
     plataforma en el selector inicial; si falta, muestra un error claro
     ahí en vez de romper el resto de la app.

4. Comparte la URL pública que te da Streamlit + la contraseña de acceso
   con el equipo (por un canal seguro, no por correo abierto).

## Probar en tu computador antes de desplegar

```bash
pip install -r requirements.txt
cp .streamlit/secrets.toml.example .streamlit/secrets.toml
# Edita .streamlit/secrets.toml con las credenciales reales
streamlit run app.py
```

## Catálogos de filtros

Todos los filtros (región, país, área temática, monto y tipo de
convocatoria) son listas desplegables con nombres reales, cargadas en
`ffn_client.py`:

- **Países**: 251, verificado contra
  `https://core.fundsforngospremium.com/api/Country/GetRegionCountries?id=20`
- **Áreas temáticas**: 58
- **Regiones**: 34
- **Montos de grant**: 6
- **Tipos de convocatoria**: 6

Si en el futuro FundsForNGOs agrega nuevos países, áreas o tipos, hay que
repetir el proceso: capturar el JSON desde las Herramientas de
Desarrollador del navegador (F12 → pestaña Network → abrir el
desplegable correspondiente en la web) y actualizar la lista
correspondiente en `ffn_client.py`.

## Países filtrados por región (cascada)

Cuando seleccionas una o más regiones en el sidebar, el dropdown de
Países ya no muestra el catálogo completo (251): consulta en vivo
`GetRegionCountries?id=<id_region>` (el mismo endpoint público de arriba,
pero pasándole el id de la región en vez de 20) y arma el dropdown solo
con esos países. Si eliges varias regiones a la vez, se unen los
resultados sin duplicar. Esto vive en dos funciones de `ffn_client.py`:

- `obtener_paises_por_region(region_id)`: hace la consulta HTTP y
  normaliza la respuesta.
- En `app.py`, `calcular_catalogo_paises(ids_regiones)` combina varias
  regiones y cae de vuelta al catálogo completo (con un aviso visible)
  si la consulta falla por cualquier motivo.

## Fecha de cierre (deadline) — filtra bien en el servidor

A diferencia de la fecha de publicación (ver abajo), el filtro de
**fecha de cierre** sí funciona correctamente en el servidor de
FundsForNGOs. Se activa enviando `filterBy: "DD"` junto con
`fromDate`/`toDate` (confirmado capturando el payload real del sitio).
En la app es un filtro opcional — checkbox "Filtrar por fecha de cierre"
en el sidebar — que, cuando está activo, manda el rango directo a la
API: no necesita el escaneo local de páginas que sí requiere la fecha
de publicación, y se puede combinar con ella sin problema (cada una usa
un mecanismo distinto: una en el servidor, la otra localmente).

## Filtro de fechas de publicación (workaround local)

**El filtro de "fecha de publicación" está roto en el propio sitio de
FundsForNGOs** — no solo en esta app. Se confirmó probando directamente
en `grants.fundsforngospremium.com`: al filtrar por un rango de fechas,
el sitio sigue devolviendo convocatorias con `Post Date` fuera de ese
rango. No hay forma de arreglar esto pidiéndole a la API que filtre
bien, porque el problema es del servidor — por eso usa un campo
distinto (`filterBy`) del que sí funciona para la fecha de cierre.

La solución implementada es descartar el filtrado del lado de FFN y
hacerlo en Python, después de traer los resultados:

1. `ffn_client.buscar_convocatorias()` le pide a la API página por
   página (10 resultados cada una) y, cuando hay un rango de fechas de
   publicación activo, revisa la fecha (`lastUpdated`) de cada
   convocatoria con `fecha_en_rango()` y descarta las que caen fuera —
   sin dejar de recorrer páginas solo porque ya "llenó" el cupo
   aparente, porque el total que reporta el sitio tampoco está
   filtrado por esta fecha.
2. Por seguridad hay un tope de `MAX_PAGINAS_ESCANEADAS = 300` páginas
   (3000 resultados). Si se llega a ese tope, la app deja de buscar sin
   avisar en pantalla — si genera resultados incompletos, hay que subir
   esa constante en `ffn_client.py`.
3. Si además el filtro de fecha de cierre está activo, el "total" que
   usa este recorrido ya viene filtrado por el servidor, así que en la
   práctica hay muchas menos páginas que escanear (la combinación de
   ambos filtros es más rápida que la fecha de publicación sola).

Dos detalles más del payload que se encontraron capturando el payload
real del sitio:

- Las fechas se envían **sin ceros a la izquierda** (`2026-9-1`, no
  `2026-09-01`) — así es como las manda el propio frontend de FFN.
  `_formato_fecha_api()` en `ffn_client.py` hace esta conversión.
- `regionID` se envía como número entero (el id de la primera región
  seleccionada, o `0` si no hay ninguna) en vez del valor fijo `"19"`
  que traía el notebook original por error.

## Evaluación y Priorización CTI (Agencia Atenea / Bankpro)

Implementa la Especificación Técnica v5.3 (algoritmo de evaluación de
convocatorias de Ciencia, Tecnología e Innovación) en `cti_scoring.py`.
Se ejecuta automáticamente, sobre los mismos datos que ya trajo el paso
de "Extracción completa" — no vuelve a consultar FundsForNGOs.

Qué calcula por cada convocatoria:

- **C1 (Alineación Temática, 25%)**: cruza "Areas tematicas" contra el
  catálogo `ISSUES_DISPONIBLES`; si no hay coincidencia, hace un respaldo
  por palabras clave en título/descripción.
- **C2 (Tiempo Disponible, 25%)**: `Plazo_Semanas = (fecha_cierre - HOY) / 7`,
  calculado dinámicamente contra la fecha del sistema en cada corrida.
- **C3 (Accesibilidad Geográfica, 15%)**: revisa si "Paises elegibles"
  incluye a Colombia (ID 48) o alguna región/cobertura amplia; si es una
  lista concreta de países sin Colombia, activa la bandera roja
  **"Geografía restrictiva para Colombia"**.
- **C4 (Viabilidad Financiera, 10%)**: mapea "Monto" contra
  `TAMANOS_GRANT_DISPONIBLES` (IDs 0-5).

`Puntuacion_Final = [(C1×0.25) + (C2×0.25) + (C3×0.15) + (C4×0.10)] / 0.75`,
categorizada ALTA (≥7.50) / MEDIA (6.50-7.49) / BAJA (<6.50).

**Regla de exclusión geográfica (obligatoria):** si se activa la bandera
"Geografía restrictiva para Colombia", la convocatoria **nunca** puede
quedar como ALTA, sin importar la puntuación — se fuerza a BAJA. Esto
está implementado como una verificación posterior al cálculo de la
puntuación, no como parte de la fórmula.

`cti_scoring.py` documenta en su propio encabezado las decisiones que
tomé para cerrar ambigüedades del documento (ej. valor exacto dentro de
cada banda de puntuación, cómo distinguir restricción "explícita" de
"incierta" con los datos disponibles). Vale la pena revisarlas si en
algún momento los resultados no coinciden con lo esperado.

En la app, esta sección aparece debajo de la extracción cruda e incluye:
tarjetas de conteo, la tabla completa de 23 columnas, una tabla filtrada
solo con las convocatorias ALTA ("Oportunidades de Máxima Prioridad"),
dos gráficos de barras (por Categoría y por Estado de Plazo) y una tabla
cruzada de los 5 donantes con más convocatorias por categoría. Las tres
tablas tienen su propio botón de descarga en Excel.

## Identidad visual (Agencia Atenea)

- **Logo**: `assets/logo_atenea.png` se muestra junto al título (pantalla de
  login y encabezado principal) y como ícono de la pestaña del navegador.
  Si el archivo llegara a faltar, la app no se rompe: cae de vuelta a un
  emoji como ícono y simplemente no muestra la imagen.
- **Colores**: `.streamlit/config.toml` define el tema con la paleta de
  Atenea. Solo se usaron 2 de los 6 colores de la tabla que compartieron
  (naranja `#E65A0F` como color primario — botones, checkboxes, elementos
  seleccionados — y azul oscuro `#020142` como color de texto), porque el
  tema clásico de Streamlit solo expone 5 colores base y el fondo
  secundario (sidebar, campos de filtro) necesita quedarse neutro para
  que los controles se sigan viendo legibles. El resto de la paleta
  (azul `#065094`, azul violeta `#5263AB`, naranja claro `#EF7A45`) se
  usó puntualmente en las barras de los gráficos, pero queda disponible
  para quien quiera extenderla con CSS personalizado más adelante.
- Nota: la tabla de colores que compartieron tiene "Azul" y "Azul claro"
  con el mismo valor (`#065094`) — asumo que fue un error de tipeo al
  armar la tabla, no lo interpreté como dos colores distintos.

## Organización de la interfaz

- **Filtros del sidebar**: Región, País y Fechas quedan siempre visibles
  (son los más usados). Áreas temáticas, Monto y Tipo de convocatoria
  quedan colapsados dentro de un expander "Filtros avanzados".
- **Resultados en pestañas**: "Extracción completa", "Evaluación y
  Priorización CTI" y "Oportunidades de Máxima Prioridad" son pestañas
  separadas (antes se apilaban todas en la misma página).
- **Tablas**: título fijo al hacer scroll horizontal (`pinned=True`,
  requiere Streamlit 1.41+, ya actualizado en requirements.txt), las
  columnas de URL son links clicleables ("🔗 Abrir"), y las columnas de
  texto largo (descripciones) tienen un ancho fijo — la grilla las trunca
  visualmente y se puede hacer clic en la celda para ver el contenido
  completo.
- **Estados de carga**: conectar, buscar y extraer usan `st.status()`, que
  muestra un log expandible con cada paso en vez de un spinner genérico.

## Selector de plataforma (FundsForNGOs / TripleFunds)

Al entrar, después de la contraseña de acceso, la app muestra dos botones
("🌐 FundsForNGOs" y "🌐 TripleFunds") y espera a que se elija uno antes de
mostrar cualquier filtro. La plataforma elegida queda guardada en la
sesión del navegador (`st.session_state`), así que se mantiene mientras
esa persona siga usando la app; un botón "🔁 Cambiar de plataforma" en la
parte de arriba del sidebar permite volver al selector en cualquier
momento (y limpia los resultados de la extracción anterior, para no
mezclar datos de una plataforma con la otra en las mismas tablas).

Todo lo que va después del selector — filtros, botones de buscar/extraer,
las tres pestañas de resultados, la evaluación CTI, los gráficos y las
descargas en Excel — es el mismo código para las dos plataformas: solo
cambian (a) qué cliente se usa por dentro (`ffn_client.FFNClient` o
`triple_funds_client.TripleFundsClient`), (b) qué filtros aparecen en el
sidebar (los de cada plataforma, ver sus secciones respectivas en este
README) y (c) qué se lee de los *Secrets* para conectarse (`FFN_EMAIL`/
`FFN_PASSWORD` para FundsForNGOs, o `TF_COOKIES` para TripleFunds — este
último no son credenciales sino cookies de una sesión ya iniciada a mano,
ver la sección de TripleFunds más abajo). Esto funciona porque ambos
clientes producen un DataFrame con exactamente las mismas 12 columnas, así
que el resto de la app no necesita saber de qué plataforma vinieron los
datos.

## TripleFunds — ya integrado a la app y validado

Segunda fuente de convocatorias,
[triple-funds.com](https://triple-funds.com/find-funding/search-all-site/),
ya validada con una corrida real (login con cookies manuales + búsqueda +
detalle + evaluación CTI) y ya conectada a `app.py` a través del
selector de plataforma descrito arriba.

### Cómo se construyó y qué tan confiable es cada parte

Igual que con FundsForNGOs, el login y la búsqueda de TripleFunds se
reconstruyeron capturando el tráfico real del sitio desde las
Herramientas de Desarrollador del navegador (pestaña Network), y luego se
validaron corriendo `probar_triplefunds.py` contra el sitio real.

**Confirmado con capturas reales y una corrida real del sitio:**
- El login usa el mecanismo de WordPress + plugin "Paid Membership
  Subscriptions": un formulario con un token de un solo uso
  (`pms_login_nonce`) que hay que leer de la página de login antes de
  enviar usuario/contraseña.
- La búsqueda es un POST a
  `https://triple-funds.com/find-funding/calls-for-proposals/` que
  devuelve HTML (no JSON, a diferencia de FundsForNGOs) con una tabla de
  resultados (`table#searchResults`).
- Los 9 filtros del formulario, con su nombre de campo real: ubicación
  (`tf-location[]`), enfoque geográfico (`tf-location-focus`, valor
  único), sector (`tf-sectors[]`), tipo de donante (`tf-donor-type[]`),
  base del donante (`tf-donor-base[]`), aplicantes elegibles
  (`tf-applicant[]`), tamaño de la oportunidad (`tf-opportunity-size[]`),
  tipo de financiación (`tf-funding-instrument[]`) y fecha de publicación
  relativa (`tf-published`, valor único: last-week/last-14/last-month/
  last-3-months/last-year). Más el texto libre (`tf-freetext`) y el
  checkbox "open for application all year" (`tf-call=na`).
- El **catálogo completo** de valores posibles de cada uno de esos
  filtros (no solo un ejemplo): se extrajo directamente de los `<select>`
  del HTML real de la página de búsqueda, que ya trae todas las opciones
  incluidas — a diferencia de FFN, aquí no hizo falta una llamada a la
  API por cada filtro. Viven en `triple_funds_client.py` como las
  constantes `TF_UBICACIONES_DISPONIBLES`, `TF_SECTORES_DISPONIBLES`,
  `TF_TIPOS_DONANTE_DISPONIBLES`, `TF_BASE_DONANTE_DISPONIBLE`,
  `TF_APLICANTES_ELEGIBLES_DISPONIBLES`, `TF_TAMANOS_OPORTUNIDAD_DISPONIBLES`,
  `TF_TIPOS_FINANCIACION_DISPONIBLES`, `TF_ENFOQUE_PAIS_DISPONIBLE` y
  `TF_PUBLICADO_DISPONIBLE`.
- La estructura de cada fila de resultados y de la página de detalle de
  una convocatoria (título, donante, tablas de datos generales/
  financieros, sectores, países elegibles agrupados por continente,
  descripción completa) — confirmada con el HTML real de una convocatoria
  de ejemplo (Sumitomo Foundation), no con una captura de pantalla como
  en el primer intento. Esto corrigió dos errores que solo se veían con
  datos reales: el tamaño del grant en la tabla de resultados vivía en un
  atributo `title` (no en el texto visible), y la URL de referencia del
  donante se estaba tomando por error de un link del menú de navegación
  en vez del link real de la convocatoria.
- La paginación usa enlaces `.../calls-for-proposals/page/N`.

**Pendiente:**
- Solo se validó la página 1 de resultados (20 filas) contra el sitio
  real — la lógica de avance a la página 2 en adelante replica la misma
  estrategia ya probada en `ffn_client.py`, pero no se ha confirmado con
  una búsqueda real de más de 20 resultados.
- El filtro "Status" (Open/Closed) existe en el HTML del formulario pero
  está oculto (`display:none`); no se expone como filtro utilizable
  porque no está claro si el sitio todavía lo usa.

### TripleFunds — login con cookies manuales (no usuario/contraseña)

`triple-funds.com` está protegido por una verificación de seguridad de
Cloudflare (la pantalla "Verificación de seguridad en curso" / el check
"Verifique que es un ser humano"). Se probaron **cuatro** formas de resolver
esa verificación de manera automática, en orden creciente de esfuerzo, y
ninguna funcionó:

1. Peticiones HTTP directas con encabezados completos de navegador → 403.
2. La librería `cloudscraper` (pensada justo para resolver retos simples
   de Cloudflare) → 403.
3. Un navegador real controlado por Playwright — primero el Chromium de
   pruebas, luego el Chrome real instalado en el computador — → Cloudflare
   detecta la conexión de automatización en sí misma (no el tipo de
   navegador ni si está oculto) y repite el reto interactivo una y otra
   vez, sin dejarlo pasar ni resolviéndolo a mano dentro de esa ventana.
4. `nodriver` — sucesor de undetected-chromedriver, hecho específicamente
   para evitar la señal de automatización que detectó a Playwright — → el
   formulario de login nunca llegó a aparecer, ni siquiera dentro de un
   tiempo de espera generoso (45s). Mismo resultado que con Playwright,
   con una herramienta diseñada justamente para evitar ese problema (ver
   `probar_nodriver_triplefunds.py`, incluido en este paquete como
   registro de esa prueba).

La solución que sí funciona, porque no usa ninguna automatización que
Cloudflare pueda detectar: **iniciar sesión a mano, una vez, en tu
navegador normal**, y copiar las cookies de esa sesión ya autenticada para
que la app las reutilice directamente en sus peticiones HTTP normales
(rápidas, sin abrir ningún navegador). Así se configura:

1. Abre `https://triple-funds.com/login/` en tu navegador de siempre e
   inicia sesión normalmente con la cuenta compartida de TripleFunds.
2. Con la sesión ya iniciada (en cualquier página de triple-funds.com, no
   hace falta quedarte en el login), abre las Herramientas de Desarrollador
   (F12 en Windows/Linux, Cmd+Opt+I en Mac) → pestaña **Network**.
3. En la casilla de filtro de esa pestaña escribe `triple-funds.com` (para
   no toparte con peticiones de otros dominios, como Google/reCAPTCHA) y
   recarga la página (F5).
4. Busca la fila cuyo **Status sea 200** y cuyo **Type sea "document"**
   (no una que haya fallado, ni un `.css`/`.js` suelto). Haz clic derecho
   sobre ella → **Copy** → **Copy as cURL (bash)**.
5. Pega ese comando completo en un editor de texto. Dentro vas a
   encontrar dos partes que necesitas copiar tal cual:
   - Todo el valor que sigue a `-b '...'` (o a `-H 'cookie: ...'`, según
     el navegador) → eso va en `TF_COOKIES`.
   - Todo el valor que sigue a `-H 'user-agent: ...'` → eso va en
     `TF_USER_AGENT`.
6. Pega ambos en `secrets.toml` (local) o en **App settings → Secrets**
   (Streamlit Community Cloud):
   ```toml
   TF_COOKIES = "el-valor-completo-de--b-o--H-cookie"
   TF_USER_AGENT = "el-valor-completo-de--H-user-agent"
   ```

`TF_USER_AGENT` no es opcional en la práctica: Cloudflare liga la cookie
`cf_clearance` al User-Agent exacto del navegador que la obtuvo, así que
si falta o no coincide, la verificación de seguridad se vuelve a activar
aunque las cookies sean válidas y recientes (esto fue justo lo que pasó
al probarlo la primera vez, sin `TF_USER_AGENT`).

Esa sesión eventualmente expira (no se sabe con certeza cuánto dura —
puede ser días o semanas): cuando deje de servir, la app lo va a avisar
con un mensaje de error claro ("las cookies ya no pasan la verificación" o
"ya no mantienen la sesión iniciada"), y hay que repetir estos 6 pasos
para refrescar `TF_COOKIES` y `TF_USER_AGENT` con una sesión nueva.

### Mantenimiento continuo: qué hacer cuando expiren las cookies

A diferencia de FundsForNGOs (que se reloguea solo en cada sesión con
`FFN_EMAIL`/`FFN_PASSWORD`), TripleFunds **no se puede reloguear solo**:
por eso hace falta que alguien del equipo (quien administre la cuenta y
tenga acceso a los *Secrets* de Streamlit Cloud) repita el proceso de
copiar cookies de vez en cuando. En la práctica:

- **Quién se entera primero de que expiraron**: la persona del equipo que
  esté usando la app en ese momento, al elegir TripleFunds y ver el error
  de conexión. Como la app no puede avisar por su cuenta (no hay forma de
  "probar" la sesión sin que alguien la use), esa persona tiene que
  avisarle a quien administra la cuenta.
- **Para evitar que el equipo se tope con el error primero**: quien
  administre la cuenta puede probar la conexión de TripleFunds de vez en
  cuando (por ejemplo, una vez por semana) para refrescar las cookies de
  forma preventiva, antes de que alguien más las necesite y se encuentre
  con el error.
- **Cómo se refrescan**: los 6 pasos de la sección de arriba, y luego
  pegar los valores nuevos en **App settings → Secrets** de Streamlit
  Community Cloud (o en `secrets.toml` si se está probando en local). Al
  guardar los *Secrets* en la nube, la app se reinicia sola con los
  valores nuevos — no hace falta volver a desplegar ni tocar el código.

`triple_funds_client.py` produce un DataFrame con las mismas 12 columnas
que `ffn_client.construir_dataframe()`, así que el algoritmo de
evaluación CTI (`cti_scoring.evaluar_cti()`) funciona sin ningún cambio
sobre datos de TripleFunds — ya se verificó de punta a punta con datos
reales.

### Cómo volver a correr el diagnóstico

```bash
pip install -r requirements.txt
python probar_triplefunds.py
```

El script pide que pegues el encabezado `Cookie` de una sesión ya iniciada
a mano (ver "TripleFunds — login con cookies manuales" arriba; no se
muestra en pantalla al pegarlo) y hace, en orden: login → una búsqueda de
prueba → el detalle de la primera convocatoria encontrada → una corrida
completa del algoritmo CTI sobre esa convocatoria. Sirve para volver a
confirmar que todo sigue funcionando si el sitio cambia algo en el futuro.

## Notas de seguridad

- La cuenta de FundsForNGOs es compartida por todos los usuarios de la
  app — no hay sesiones individuales por persona.
- `APP_PASSWORD`, `FFN_EMAIL` y `FFN_PASSWORD` viven únicamente en los
  *Secrets* de Streamlit Community Cloud, nunca en el código ni en el
  repositorio.
- Si sospechas que la contraseña de la app se filtró, cámbiala en
  **App settings → Secrets** y la app se reinicia con la nueva.
