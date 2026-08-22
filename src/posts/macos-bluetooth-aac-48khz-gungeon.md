---
title: El misterio de los Sony que sonaban lentos al abrir Enter the Gungeon
date: 2026-08-22
description: Cómo investigamos un fallo de audio Bluetooth entre Wwise, CoreAudio y una allowlist oculta de AAC a 48 kHz en macOS.
permalink: /posts/macos-bluetooth-aac-48khz-gungeon/
---

Todo empezó con un síntoma difícil de describir y todavía más difícil de buscar en Internet.

Unos Sony WH-CH720N funcionaban perfectamente en un MacBook Air M4. Música, vídeos y sonido del sistema se escuchaban con normalidad. Pero bastaba abrir *Enter the Gungeon* para que todo el audio empezara a sonar lento, distorsionado y con un retardo considerable.

No era únicamente el juego. Si Brave estaba reproduciendo audio, también quedaba afectado. Al cerrar *Gungeon*, la normalidad regresaba.

Parecía el tipo de fallo que admite demasiadas explicaciones: un cambio de perfil Bluetooth, Game Mode, una mala negociación del codec, un bug del juego, una incompatibilidad de Unity o alguna peculiaridad de los auriculares. Durante la investigación probamos casi todas.

La solución final fue una sola preferencia de macOS. Llegar hasta ella exigió seguir la ruta completa del audio, descartar varias hipótesis razonables y terminar mirando dentro de `bluetoothd` y de una base SQLite que Apple instala con el sistema.

Este es el recorrido.

> **Aviso:** la solución utiliza una preferencia privada y no documentada de macOS. Es reversible, pero afecta globalmente al audio Bluetooth y puede cambiar en futuras versiones del sistema.

## Primera parada: ¿había activado el micrófono?

Cuando unos auriculares Bluetooth pierden calidad de forma brusca, el sospechoso habitual es el perfil manos libres.

Para música, macOS utiliza normalmente A2DP: audio estéreo con codecs como AAC o SBC. Si una aplicación necesita el micrófono de los propios auriculares, el sistema puede cambiar a HFP. Ese perfil reserva ancho de banda para enviar y recibir voz, a costa de reducir mucho la calidad de reproducción.

Era una explicación atractiva. Un juego puede inicializar dispositivos de entrada aunque el jugador no esté usando chat de voz.

Pero no era lo que sucedía:

- La salida seguía utilizando A2DP.
- El codec continuaba siendo AAC.
- El micrófono seleccionado era un dispositivo USB independiente.
- Los Sony no estaban activos como entrada.

El audio sonaba mal, pero no por el conocido «modo llamada» de Bluetooth.

Habíamos descartado la respuesta fácil.

## Segunda parada: Game Mode

macOS puede activar Game Mode cuando un juego se ejecuta a pantalla completa. Este modo modifica prioridades del sistema y promete reducir la latencia de accesorios inalámbricos. Si el fallo aparecía exactamente al abrir un juego, tenía sentido comprobarlo.

Desactivamos Game Mode para *Enter the Gungeon* y repetimos la prueba.

El problema continuó sin ningún cambio apreciable.

Esto no demostraba todavía dónde estaba el fallo, pero sí eliminaba otra capa del sistema. No era una optimización general aplicada por macOS al detectar un juego.

## Escuchar el sistema, no solo los auriculares

La interfaz gráfica indicaba que los Sony estaban conectados. No explicaba cómo se había configurado realmente el stream de audio.

El siguiente paso fue observar los logs de `bluetoothd` y `coreaudiod` mientras conectábamos los auriculares y abríamos el juego. Puede hacerse desde Console.app o con un filtro como este:

```bash
/usr/bin/log stream --style compact --level debug \
  --predicate '(process == "bluetoothd" OR process == "coreaudiod") AND (eventMessage CONTAINS[c] "codec" OR eventMessage CONTAINS[c] "KHz" OR eventMessage CONTAINS[c] "srIn")'
```

Antes de iniciar *Gungeon*, el enlace mostraba:

```text
A2DP configured at 44.1 KHz. Codec: AAC-LC
```

AAC seguía activo, así que la primera hipótesis quedaba todavía más lejos. Sin embargo, al abrir el juego sucedía algo interesante: la salida lógica de CoreAudio pasaba a 48 kHz y el stream se reiniciaba.

También aparecía este error:

```text
Device is creating a control with invalid class ID 0,
base class ID 1818588780
```

Por primera vez teníamos una diferencia objetiva entre el estado bueno y el malo:

```text
Antes del juego: 44.100 Hz
Con el juego:     48.000 Hz en la salida lógica
```

La pregunta cambió. Ya no era «¿por qué Bluetooth suena mal?», sino «¿quién está pidiendo 48 kHz y qué ocurre cuando lo hace?».

## Siguiendo los 48 kHz hasta Wwise

*Enter the Gungeon* es un juego construido con Unity 2017.4.27f1. Sin embargo, su audio no depende únicamente de la configuración habitual de Unity: utiliza Wwise como motor de sonido.

Al revisar su configuración encontramos que el audio normal de Unity estaba desactivado y que Wwise se inicializaba directamente a 48 kHz.

Esto no era, por sí solo, un error. Los 48 kHz son una frecuencia muy habitual en videojuegos y vídeo. Wwise estaba trabajando en el formato para el que el juego había sido diseñado.

La ruta empezaba a dibujarse así:

```text
Enter the Gungeon
       ↓
Wwise a 48 kHz
       ↓
CoreAudio cambia la salida a 48 kHz
       ↓
Algo falla antes de llegar a los Sony
```

Todavía no sabíamos si el problema estaba en el juego, en CoreAudio o en la negociación Bluetooth. Así que intentamos reducir toda la cadena a 44,1 kHz.

## El primer desvío: un dispositivo agregado a 44,1 kHz

Creamos un dispositivo de audio agregado llamado:

```text
WH-CH720N 44.1 (Gungeon)
```

La idea era sencilla: presentar al juego una salida fijada a 44,1 kHz y evitar que modificara el dispositivo Bluetooth real.

No funcionó. *Gungeon* también llevó el dispositivo lógico a 48 kHz.

El agregado no imponía una frontera capaz de detener la decisión de Wwise y CoreAudio. Lo eliminamos después de la prueba.

## El segundo desvío: pedir estabilidad a los Sony

La aplicación Sony | Sound Connect ofrece una opción para priorizar una conexión estable. Quizá los auriculares estaban negociando un modo demasiado exigente o reaccionaban mal a un cambio de parámetros.

Activamos `Priority on stable connection`, reconectamos y repetimos la captura.

macOS siguió configurando AAC a 44,1 kHz. La opción de Sony podía influir en el comportamiento de radio o en otras preferencias del auricular, pero no modificaba la frecuencia elegida por el Mac.

## El experimento que nos obligó a retroceder

Si Wwise insistía en 48 kHz, ¿podíamos hacer que arrancase a 44,1 kHz?

Antes de tocar el juego comprobamos que *Enter the Gungeon* no aparecía protegido mediante VAC ni otro sistema anti-cheat equivalente. Después preparamos un cambio mínimo y reversible: un parche de 13 bytes en `Assembly-CSharp.dll` para inicializar Wwise a 44,1 kHz.

Al principio pareció funcionar. Después entramos en una partida y disparamos.

Bajo carga, el motor empezó a producir muestras `NaN` y picos de amplitud extremos. No era una degradación sutil: existía riesgo de generar golpes desagradables o potencialmente peligrosos para los oídos y los auriculares.

Cerramos el juego y restauramos inmediatamente la DLL original. Su SHA-256 era:

```text
88981a5b96e52675a7420f61e010990fc70c5d75887caec86f73215b74bd110c
```

Este fracaso fue útil. Cambiar un valor de inicialización no significaba que todo el contenido de audio, los bancos de Wwise y los cálculos temporales del juego fueran compatibles con otra frecuencia.

El juego esperaba 48 kHz. Forzarlo a abandonar esa frecuencia era intervenir en el extremo equivocado de la cadena.

## El tercer desvío: mantener 48 kHz mediante una salida múltiple

Probamos entonces la estrategia opuesta. Creamos un dispositivo multi-salida denominado:

```text
Gungeon 48 -> Sony
```

La intención era ofrecer 48 kHz al juego y dejar que macOS adaptara el audio al dispositivo físico.

Tampoco funcionó. El dispositivo lógico adoptaba 48 kHz, pero el enlace Bluetooth subyacente no cambiaba de la forma esperada. Habíamos movido la frecuencia dentro de CoreAudio sin modificar la decisión tomada por `bluetoothd`.

Eliminamos también ese dispositivo.

A estas alturas habíamos aprendido algo importante: ni Audio MIDI Setup ni los dispositivos virtuales controlaban toda la ruta. Había una política más abajo.

## La línea que cambió la investigación

Al revisar los logs de conexión apareció un mensaje que hasta entonces no habíamos interpretado en toda su extensión:

```text
Bad48KHzCodecs: Disabling 48 KHz - Device is NOT in 48 KHz AAC allowlist
```

El texto era explícito. Los Sony anunciaban AAC, pero macOS estaba eliminando la posibilidad de usarlo a 48 kHz porque el dispositivo no pertenecía a una lista interna.

Después de esa decisión, el sistema terminaba configurando:

```text
A2DP configured at 44.1 KHz. Codec: AAC-LC
```

Esto aclara una confusión frecuente en foros: quedar fuera de esa allowlist no desactiva necesariamente AAC. Desactiva **AAC a 48 kHz**. El enlace puede continuar usando AAC a 44,1 kHz.

La ruta completa del fallo era ahora visible:

```text
Wwise:        48.000 Hz
CoreAudio:    48.000 Hz
Bluetooth:    AAC limitado a 44.100 Hz
```

Los reinicios del stream y la adaptación defectuosa entre ambos lados coincidían con el comienzo de la lentitud, la distorsión y el retardo. Si conseguíamos que el enlace AAC aceptara 48 kHz, toda la cadena podría quedar alineada sin modificar el juego.

Pero antes necesitábamos saber qué era realmente aquella allowlist.

## Abriendo `bluetoothd`

Apple no ofrece una interfaz para consultar esa lista. Tampoco encontramos documentación oficial sobre ella. Pasamos entonces al análisis estático de `/usr/sbin/bluetoothd`, siempre en modo de solo lectura.

Dentro del binario aparecieron varias pistas relacionadas:

- `Bad48KHzCodecs`
- `Default48KHz`
- una sección de preferencias llamada `A2DP`
- referencias a una allowlist y una denylist
- código que consultaba información persistente sobre dispositivos

La pista siguiente nos llevó a:

```text
/Library/Application Support/BTServer/pincode_defaults.db
```

Aunque el nombre sugiere una colección de PIN predeterminados para dispositivos antiguos, el archivo es una base SQLite con bastante más información. Contiene tablas como:

- `devices`
- `matching_rules_did`
- `matching_rules_oui`
- `manufacturers`
- `makes`
- `makeGroups`

Dentro de `devices` encontramos dos registros difíciles de interpretar de otra manera:

| `booleanProperties` | Descripción |
|---:|---|
| `0x00800000` | `48 KHz AAC Blacklist` |
| `0x01000000` | `48 KHz AAC Whitelist` |

Ya no estábamos deduciendo la existencia de las listas únicamente a partir de un mensaje de log. El propio sistema incluía registros con esos nombres.

## Quién estaba dentro

Esta consulta muestra las reglas asociadas en la versión de macOS analizada:

```bash
sqlite3 -header -column \
  '/Library/Application Support/BTServer/pincode_defaults.db' '
SELECT
  r.id,
  r.vid_src,
  printf("0x%04X", r.vid) AS vid,
  CASE WHEN r.pid IS NULL THEN "*"
       ELSE printf("0x%04X", r.pid) END AS pid,
  d.description
FROM matching_rules_did AS r
JOIN devices AS d ON d.id = r.device_id
WHERE (d.booleanProperties & (8388608 | 16777216)) != 0
ORDER BY r.id;
'
```

El resultado era sorprendentemente pequeño:

| Política | VID | PID |
|---|---:|---:|
| Blacklist | `0x0876` | cualquiera |
| Blacklist | `0x008A` | cualquiera |
| Blacklist | `0x000A` | `0xFFFF` |
| Whitelist | `0x004C` | cualquiera |

El binario incluía además comprobaciones especiales para identificadores de Apple: Bluetooth `0x004C` y USB `0x05AC`, cada uno con su tipo de origen correspondiente.

No parecía existir una larga lista de modelos aprobados. En este build, la política predeterminada era esencialmente confiar en dispositivos identificados como Apple y excluir del modo de 48 kHz a los demás, salvo otra lógica interna que no hubiéramos observado.

Los Sony aparecían con VID `0x054C`. Es fácil leerlo deprisa y confundirlo con `0x004C`, pero son valores distintos. Los WH-CH720N no estaban en la allowlist ni en la denylist.

## El interruptor escondido

La referencia `Default48KHz` resultó ser la pieza que faltaba.

La lógica observada podía resumirse así.

Con la política predeterminada:

```text
¿Está el dispositivo en la allowlist?
├─ Sí → se permite AAC a 48 kHz
└─ No → se elimina la opción de 48 kHz
```

Con `Default48KHz` activado:

```text
¿Está el dispositivo en la denylist?
├─ Sí → se elimina la opción de 48 kHz
└─ No → se permite 48 kHz si el dispositivo lo anuncia
```

La preferencia no enseña 48 kHz a unos auriculares incapaces de reproducirlo. Cambia el criterio de confianza: de «denegado salvo aprobación» a «permitido salvo incompatibilidad conocida».

Ese era exactamente nuestro caso. Los Sony ofrecían 48 kHz y no estaban en la denylist.

## Por qué podría estar construido así

Apple no lo documenta, por lo que cualquier explicación sobre sus motivos debe presentarse como especulación. Aun así, la estructura encontrada permite formular varias hipótesis razonables.

### Compatibilidad conservadora

`Bad48KHzCodecs` es un nombre bastante revelador. En Bluetooth, que un dispositivo anuncie una capacidad no garantiza que todas sus combinaciones de firmware, codec y transporte funcionen bien. Una implementación puede aceptar AAC a 48 kHz y después sufrir cortes, errores de reloj, paquetes inválidos o consumos excesivos.

Una allowlist permite a Apple habilitar esa modalidad únicamente en hardware que ha probado. Desde la perspectiva de soporte, una frecuencia algo menor suele ser preferible a una conexión inestable.

### Una herencia de dispositivos antiguos

La base `pincode_defaults.db` contiene conocimiento acumulado sobre muchos dispositivos y reglas de compatibilidad. Es posible que la política naciera cuando el soporte de AAC a 48 kHz era menos uniforme.

Conservar una allowlist evita regresiones en hardware antiguo, pero también hace que modelos modernos capaces queden limitados si Apple no actualiza sus reglas.

### Apple conoce mejor su propio hardware

Apple controla el firmware, los identificadores y el comportamiento de AirPods y Beats con chips propios. Puede validar la cadena completa y reconocer esos dispositivos de forma fiable.

Eso explica técnicamente que los identificadores Apple reciban confianza predeterminada. También produce una ventaja práctica para el ecosistema de la compañía: dispositivos de terceros que soportan 48 kHz pueden quedarse en 44,1 kHz.

No tenemos pruebas de que el objetivo sea crear deliberadamente esa ventaja comercial. Es una consecuencia compatible tanto con una estrategia conservadora de calidad como con una preferencia de ecosistema, y ambas pueden coexistir.

### Coste de probar un mercado enorme

Apple vende un número limitado de familias de auriculares, mientras que el mercado A2DP incluye miles de modelos, revisiones silenciosas de hardware y versiones de firmware. Mantener una allowlist completa de terceros requiere pruebas y actualizaciones continuas.

La solución operativamente más barata es aprobar lo propio y utilizar un valor seguro para el resto.

### Una migración que todavía no es pública

La existencia conjunta de `Default48KHz`, una allowlist y una denylist sugiere otra posibilidad: Apple puede disponer de dos políticas para desplegar gradualmente un comportamiento más permisivo.

Primero se usa una allowlist. Cuando la compatibilidad general mejora, el sistema puede pasar a permitir 48 kHz por defecto y conservar únicamente excepciones conocidas en la denylist.

No sabemos si esa migración está planificada, si la opción se usa solo durante pruebas internas o si se activa de manera diferente en otros productos. El nombre del ajuste y la estructura del código hacen plausible la hipótesis, pero no la demuestran.

## Aplicar la solución

Antes de modificar una preferencia privada guardamos el estado anterior:

```bash
sudo defaults export com.apple.MobileBluetooth.debug \
  "$HOME/Desktop/MobileBluetooth-debug-before-Gungeon.plist"
```

Después activamos los 48 kHz como comportamiento predeterminado para A2DP:

```bash
sudo defaults write com.apple.MobileBluetooth.debug A2DP \
  -dict-add Default48KHz -bool true
notifyutil -p com.apple.bluetooth.prefsChanged
```

Desconectamos y volvimos a conectar los Sony.

Esta vez el log decía:

```text
A2DP configured at 48.0 KHz. Codec: AAC-LC
```

CoreAudio confirmó además que no estaba convirtiendo entre ambas frecuencias:

```text
srIn = 48000, srOut = 48000
```

Después de muchas pruebas, la ruta completa por fin coincidía:

```text
Wwise 48 kHz
      ↓
CoreAudio 48 kHz
      ↓
AAC 48 kHz
      ↓
Sony WH-CH720N
```

Abrimos Brave y *Enter the Gungeon* al mismo tiempo. Iniciamos una partida. Disparamos, provocamos varios sonidos simultáneos y continuamos jugando.

El audio permaneció estable. Ya no sonaba lento, no se distorsionaba y el retardo anómalo había desaparecido.

Reiniciamos el Mac y repetimos la conexión. El ajuste persistía.

## Volver atrás

La preferencia afecta globalmente al audio Bluetooth, no solo a los Sony ni a *Gungeon*. Un dispositivo incluido por una buena razón en la denylist debería seguir protegido, pero siempre existe la posibilidad de encontrar otro auricular que anuncie 48 kHz y lo implemente mal.

Para restaurar la configuración guardada:

```bash
sudo defaults import com.apple.MobileBluetooth.debug \
  "$HOME/Desktop/MobileBluetooth-debug-before-Gungeon.plist"
notifyutil -p com.apple.bluetooth.prefsChanged
```

Después hay que desconectar y reconectar el dispositivo Bluetooth.

Al terminar nuestra investigación no quedaban modificaciones en el juego, dispositivos agregados ni salidas múltiples. La DLL original había sido restaurada y el único cambio permanente era la preferencia reversible de Bluetooth.

## Lo que encontramos en Internet

Apple no publica la lista ni explica `Default48KHz`. Sin embargo, el comportamiento es observable públicamente desde hace años.

En 2022, un usuario de Monterey y Ventura publicó exactamente:

```text
Bad48KHzCodecs: Disabling 48 KHz - Device is NOT in 48 KHz AAC allowlist
```

Otro análisis posterior capturó la negociación completa de unos Bowers & Wilkins Px7 S2e. macOS rechazaba 48 kHz por no estar en la allowlist y terminaba usando AAC-LC a 44,1 kHz. El autor mencionaba además unos Sennheiser HD 450BT con el mismo comportamiento.

Lo que no encontramos publicado fue:

- la estructura actual de estas reglas;
- los bits de `booleanProperties`;
- la lista exacta de VID/PID de este build;
- el funcionamiento de `Default48KHz` como cambio entre allowlist y denylist;
- una forma oficial y soportada de modificar la política.

Los foros habían visto la sombra del mecanismo en los logs. El análisis local permitió seguirla hasta el código y la base de datos.

## Lo que aprendimos durante el viaje

El primer aprendizaje fue que «usa AAC» no describe por completo una conexión Bluetooth. La frecuencia de muestreo y las restricciones aplicadas durante la negociación pueden ser igual de importantes.

El segundo fue que Audio MIDI Setup solo muestra y controla una parte de la ruta. Un dispositivo lógico a 48 kHz no implica que el transporte A2DP esté funcionando a esa frecuencia.

El tercero fue metodológico: el parche más cercano al síntoma no siempre está cerca de la causa. Modificar Wwise parecía directo, pero introducía riesgos dentro de un juego diseñado para 48 kHz. Corregir la política en el punto donde convergían Wwise, CoreAudio y Bluetooth resultó más pequeño y más seguro.

Y el último fue que los logs del sistema pueden convertir una sensación subjetiva —«el audio suena raro»— en una secuencia comprobable:

```text
El juego pide 48 kHz
→ CoreAudio cambia de frecuencia
→ bluetoothd rechaza AAC a 48 kHz
→ el stream se reinicia y falla
→ habilitamos 48 kHz
→ toda la ruta queda alineada
→ el problema desaparece
```

La solución ocupaba una línea. Entender por qué esa línea era la correcta fue casi todo el trabajo.

## Referencias

- [Bluetooth SIG — Advanced Audio Distribution Profile](https://www.bluetooth.com/specifications/specs/advanced-audio-distribution-profile/)
- [Bluetooth SIG — A2DP 1.4](https://www.bluetooth.com/specifications/specs/advanced-audio-distribution-profile-1-4/)
- [Dev1Galaxy — macOS Bluetooth Audio: annotated console logs](https://dev1galaxy.org/viewtopic.php?id=7730)
- [MyApple.pl — codecs Bluetooth en macOS 12 y 13](https://myapple.pl/forums/topic/353207-kodeki-bt-aptx-i-aac-w-mac-os-12-i-13-nie-dzia%C5%82aj%C4%85-lub-dzia%C5%82aj%C4%85-wybi%C3%B3rczo/)
- [Reddit — macOS Sonoma reducing Bluetooth audio quality](https://www.reddit.com/r/MacOS/comments/18kj10t)
- [Apple StackExchange — comprobar el codec Bluetooth activo](https://apple.stackexchange.com/questions/407691/how-do-i-check-the-active-audio-codec-for-a-bluetooth-connection-in-big-sur)
- [GitHub Gist — Enable High Quality mode on macOS](https://gist.github.com/dvf/3771e58085568559c429d05ccc339219)
- [Apple Support — About lossless audio in Apple Music](https://support.apple.com/118295)
