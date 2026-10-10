# Sincronización entre móviles

La app guarda los datos en cada navegador. Con la sincronización activada, todos los móviles conectados
a la misma lista ven y cambian los mismos datos (listas, historial de tickets y menú). Sigue funcionando
sin cobertura: los cambios se guardan en el móvil y se envían al volver la conexión.

Mientras no se configure Firebase (paso 4), la app funciona exactamente como siempre y no hace ninguna
petición a internet.

## Puesta en marcha (una sola vez, unos 5 minutos)

Los nombres de los menús de Firebase pueden cambiar un poco con el tiempo; busca el equivalente.

1. Entra en <https://console.firebase.google.com> con tu cuenta de Google y pulsa **Crear un proyecto**.
   Ponle un nombre (por ejemplo `mi-compra`). Google Analytics no hace falta: puedes desactivarlo.
2. En el menú **Compilación** (Build) elige **Firestore Database** → **Crear base de datos**.
   Ubicación en Europa (por ejemplo `eur3` o `europe-west`) y modo **producción**.
3. Abre la pestaña **Reglas**, borra lo que haya, pega el contenido del archivo
   [`firestore.rules`](firestore.rules) y pulsa **Publicar**. Estas reglas impiden listar, borrar o
   escribir cosas raras: solo se puede leer o escribir un documento conociendo su código secreto.
4. Pulsa el engranaje ⚙ → **Configuración del proyecto** → pestaña **General** → **Tus apps** → icono web
   `</>`. Registra la app (no hace falta Hosting) y copia del bloque `firebaseConfig` estos dos valores:
   `apiKey` y `projectId`. No son secretos: todas las webs que usan Firebase los llevan a la vista.
5. Pega esos dos valores en `index.html`, en la línea `const FIREBASE_CONFIG = { apiKey: '', projectId: '' };`,
   y publica el cambio (o pásaselos a quien mantenga la app para que lo haga).

## Uso

1. En el primer móvil: **Gestión → Sincronización entre móviles → Activar y crear código**.
   Se crea un código secreto de 32 caracteres y se suben tus datos.
2. Pulsa **Copiar enlace para otro móvil** y ábrelo en el otro móvil (por WhatsApp, por ejemplo).
   Ese móvil se conecta solo. Si ya tenía datos propios, antes te pregunta si quieres reemplazarlos
   (y guarda una copia local por si acaso).
3. A partir de ahí los cambios se envían solos al cabo de un par de segundos y cada móvil consulta la nube
   cada ~15 segundos mientras la app está abierta. El icono ⟳ fuerza una sincronización.

## Cosas que conviene saber

- **El enlace es la llave.** Quien tenga el enlace puede ver y cambiar tus listas. No lo publiques.
  Si se te va de las manos, usa **Dejar de sincronizar este móvil** y activa una lista nueva.
- **Cambios a la vez.** Se fusionan producto a producto: si en un móvil marcas la leche y en otro añades
  pan, quedan las dos cosas. Si los dos cambian el mismo dato del mismo producto, gana el móvil que
  sincroniza en último lugar. Si uno borra un producto que el otro ha modificado, se conserva.
- **Importar una copia** (Gestión → Copia de seguridad) reemplaza todo y, con la sincronización activada,
  también en los demás móviles.
- **Tamaño.** Un documento de Firestore admite como mucho 1 MiB. Las reglas limitan el historial a unos
  450 000 caracteres (cientos de tickets). Si se supera, la app avisa; exporta una copia y borra tickets antiguos.
- **Coste.** El plan gratuito de Firebase (Spark) cubre de sobra una lista de la compra. Comprueba las
  cuotas actuales en <https://firebase.google.com/pricing>.
- **Copia local de seguridad.** Cuando un móvil con datos propios se conecta a una lista existente, lo que
  tenía se guarda en el almacenamiento del navegador bajo la clave `mi_sync_backup_v1`.

## Cómo funciona (por si hay que tocarlo)

- Cada lista compartida es un documento `compras/{código}` en Firestore con los campos `lists`, `history`
  y `menu` (texto JSON) y `v` (versión del formato). Se usa la API REST, sin librerías externas.
- Cada sincronización lee la nube, fusiona (tres vías: última versión sincronizada, este móvil y la nube)
  y guarda con la condición «solo si nadie lo ha cambiado desde que lo leí» (`currentDocument.updateTime`).
  Si otro móvil se adelantó, se vuelve a leer, a fusionar y a guardar.
- Estado local: `mi_sync_code_v1` (código) y `mi_sync_base_v1` (última versión sincronizada).
