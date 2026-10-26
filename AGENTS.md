> Este archivo existe duplicado como CLAUDE.md y AGENTS.md. Si actualizas uno, actualiza también el otro con el mismo contenido.

# Liria

## Qué es
**Liria** es una app web (PWA) en español que **lee documentos en voz alta**. Se abre un PDF, un Word (.docx), un .txt o se pega texto, y la app lo lee con las voces del propio dispositivo, resaltando la frase que suena. Pensada para usarse en móvil (instalable en la pantalla de inicio) y en escritorio.

- URL publicada: https://elministroog.github.io/liria/
- Repositorio (público): https://github.com/ElMinistroOG/liria
- Idioma de la interfaz y de las voces: español.

## Tecnologías y dependencias
- HTML + CSS + JavaScript puro, **sin framework, sin build, sin npm**. Todo el código de la app está en un único `index.html` (~540 líneas; la mayor parte del peso son fuente e imagen incrustadas en base64).
- **Web Speech API** (`speechSynthesis`) para la voz. No hay servidor ni claves de API.
- **Librerías cargadas bajo demanda desde cdnjs** (solo al abrir un archivo de ese tipo):
  - pdf.js 3.11.174 (`pdf.min.js` + `pdf.worker.min.js`) para PDF.
  - mammoth 1.6.0 para .docx.
- Google Fonts: Inter (pesos 200–500) por `<link>`. La fuente de display de 7 segmentos (DSEG7) está incrustada en base64 en el CSS.
- PWA: `manifest.webmanifest` + service worker `sw.js` (funciona sin conexión; ver abajo).
- Wake Lock API para que la pantalla no se apague mientras lee.

## Estructura de archivos
| Archivo | Para qué sirve |
|---|---|
| `index.html` | Toda la app: estilos, marcado y lógica. Dos pantallas: inicio (`#home`: elegir archivo / pegar texto) y lector (`#reader`). |
| `sw.js` | Service worker. Caché `liria-v1`. Páginas: red primero, caché si no hay conexión. Resto (iconos, pdf.js, mammoth): caché primero. |
| `manifest.webmanifest` | Datos de instalación PWA (nombre, colores `#E7E7E4`, iconos, `display: standalone`). |
| `icon-192.png`, `icon-512.png` | Iconos PWA (el 512 se usa también como maskable). |
| `apple-touch-icon.png` | Icono para "Añadir a pantalla de inicio" en iOS. |

## Cómo está organizado `index.html` (bloque `<script>`, dentro de una IIFE)
Secciones marcadas con comentarios `/* ... */`:
- **tema**: 3 temas (`light`, `mist`, `dark`) con `data-theme` en `<html>`; se guarda en `localStorage` (`liria-theme`) y por defecto sigue `prefers-color-scheme`.
- **Texto → frases**: `toParas`, `sentences`, `chunk`, `hardSplit` (trocea frases de más de 200 caracteres), `buildDoc` construye el documento: `S` = frases, `P` = índice de inicio de cada párrafo, `cum`/`total` = tiempo estimado con `CPS=14.5` caracteres por segundo.
- **Reproducción**: `speakAt(i)` lee una frase con `SpeechSynthesisUtterance` y al terminar pasa a la siguiente (control con `token` para ignorar eventos de lecturas canceladas). `play/pause/stop/jump/prevPara/nextPara`.
- **Velocidad**: 0.5–2.0 en pasos de 0.1 (`setRate`); al cambiar durante la lectura reinicia la frase con un retardo de 350 ms.
- **Voces**: `loadVoices` filtra voces `es*`, descarta voces "novedad" (Eloquence, Eddy, Flo, Grandma…), ordena por calidad (Premium > Mejorada/Natural > Google > resto) y las etiqueta con región.
- **Tamaño de letra**: 15–28 px (18 en móvil, 19 en escritorio).
- **Pantalla encendida**: `keepAwake` / `letSleep`.
- **Archivos**: `handleFile` (PDF vía `readPdf`, .docx vía mammoth, `.doc` antiguo rechazado con aviso, resto como texto). Admite selector, arrastrar y soltar, y pegar.
- **Teclado**: Espacio = play/pausa, ←/→ = frase anterior/siguiente, Mayús+←/→ = párrafo anterior/siguiente, Esc = cierra menú.
- Al final, un segundo `<script>` registra `sw.js`.

## Ejecutar en local
No hay build. El service worker exige `http://localhost` o https (no funciona abriendo con `file://`):
```bash
cd "<carpeta del proyecto>"
python3 -m http.server 8000
# abrir http://localhost:8000
```
Para probar la lectura de PDF/Word hace falta internet (las librerías vienen de cdnjs). Para probar el modo offline: cargar una vez, luego DevTools → Application → Service Workers → Offline.

## Publicación
- GitHub Pages desde la rama `main`, carpeta `/` (modo "legacy", sin Actions). Cada `git push` a `main` republica en 1–2 min.
- Cuenta de GitHub autenticada con `gh`: **ElMinistroOG** (el keyring la etiqueta como "aarhonale", pero el login real es ElMinistroOG). `origin` = `https://github.com/ElMinistroOG/liria.git`.
- Repo **público** a petición del usuario (GitHub Pages gratis lo requiere). Nota: el usuario prefiere por defecto repos privados; si algún día se migra a un hosting que lo permita, valorar volver a privado.
- Copias de trabajo: existe la carpeta original `~/Documents/Mis Apps/liria` y esta copia en Google Drive (`Mi unidad/PROYECTOS IA/liria`), ambas con el mismo repo git. Conviene trabajar en una sola y sincronizar con git para no divergir.

## Decisiones importantes
- **Un solo `index.html` con todo incrustado** (CSS, fuente DSEG7 e imagen en base64): cero dependencias de build, fácil de alojar y de cachear offline.
- **Voz del dispositivo (Web Speech API)** en vez de un servicio TTS en la nube: gratis, privado (el texto nunca sale del dispositivo), sin claves. Contrapartida: la calidad depende de las voces instaladas (en iOS/macOS conviene descargar voces "Mejorada/Premium").
- **pdf.js y mammoth se cargan solo cuando hacen falta**, y el service worker los guarda en caché después, para que la primera carga sea ligera.
- **Service worker con "red primero" para la página**: así las actualizaciones llegan sin tener que cambiar versiones de caché; si no hay conexión usa la copia guardada.
- **Lectura frase a frase** (no un único texto largo): permite resaltar, saltar, cambiar velocidad/voz en mitad de la lectura y evita los cortes de los motores de voz con textos largos.
- **Reconstrucción de párrafos en PDF** por posición vertical (`y`) de las líneas: salto de párrafo si la separación supera 1.45× la mediana, o si la línea anterior termina en puntuación final; une palabras partidas por guion de fin de línea; ignora líneas que son solo un número (nº de página).

## Problemas resueltos / no tocar
- `speakAt` espera 70 ms tras `synth.cancel()` y usa `token`: es para evitar que Chrome/Safari ignoren el `speak()` justo tras un `cancel()` y para descartar `onend` de lecturas antiguas. **No quitar el retardo ni el control de `token`.**
- En `onerror` se ignoran `interrupted`/`canceled` (los provocamos nosotros); `not-allowed` pausa (el navegador exige gesto del usuario).
- Al volver a la pestaña (`visibilitychange`) se vuelve a pedir el wake lock y, si el navegador dejó de hablar, se relanza la frase actual.
- Se hace `synth.cancel()` en `pagehide` para que no siga hablando al cerrar.
- Voces "novedad" excluidas a propósito (suenan mal); si no queda ninguna, se usan todas las `es*`.
- Si no hay voces en español se muestra el aviso `#vhint` en vez de fallar.
- Los PDF escaneados (sin texto seleccionable, <20 caracteres) dan un aviso claro; **no hay OCR**.
- Los `.doc` antiguos no se soportan (aviso: guardar como .docx o PDF).
- Si se cambia algo del `CORE` cacheado en `sw.js` (lista de archivos precacheados), subir el nombre de la caché (`liria-v2`…) para forzar actualización de los recursos que van "caché primero".
- Los iconos y el manifest se referencian con rutas relativas (`./`) porque la app vive en un subdirectorio de GitHub Pages (`/liria/`); **no usar rutas absolutas que empiecen por `/`**.

## Estado actual
Funciona (código final = commit `230bdda "Primer commit: Liria"`, idéntico en ambas carpetas):
- Abrir PDF, .docx, .txt; pegar texto; arrastrar y soltar.
- Lectura con resaltado de frase, barra de progreso con reloj de 7 segmentos y tiempo estimado, saltos por frase/párrafo, velocidad 0.5–2×, selector de voz, tamaño de letra, 3 temas.
- Instalable como PWA y utilizable sin conexión (salvo abrir PDF/Word la primera vez sin haber cargado las librerías antes).
- Publicada en GitHub Pages.

No hay pruebas automáticas; la verificación ha sido manual.

## Pendientes e ideas futuras
- Recordar la posición de lectura y los últimos documentos (p. ej. `localStorage`/IndexedDB).
- Guardar velocidad, voz y tamaño de letra entre sesiones (ahora solo se guarda el tema).
- Empaquetar pdf.js y mammoth en el repo (en lugar de cdnjs) para que el primer uso de PDF/Word también funcione offline.
- OCR para PDF escaneados; soporte de .epub / .rtf / .odt.
- Marcadores y búsqueda de texto dentro del documento.
- Temporizador de apagado (para escuchar antes de dormir).
- Compartir con la app desde el menú "Compartir" del móvil (Web Share Target en el manifest).
- Decidir si el repo vuelve a ser privado (ver sección Publicación).
