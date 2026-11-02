# Tutor Raspberry Pi con Ollama

Asistente educativo local para estudiantes que comienzan con Raspberry Pi. Responde sobre instalación de Raspberry Pi OS, acceso remoto mediante SSH y precauciones de GPIO, usando instrucciones y contenido de referencia incluidos en un `Modelfile`.

**Ollama se ejecuta en el PC y Raspberry Pi es la temática del asistente.** Esta guía describe la configuración en Windows con PowerShell. El repositorio contiene la interfaz y las configuraciones; los pesos del modelo se descargan mediante Ollama.

## Qué necesitas

- [Ollama](https://ollama.com/download/windows) instalado y abierto.
- [Git](https://git-scm.com/downloads/win) para clonar el repositorio, o descargarlo desde GitHub con **Code → Download ZIP**.
- [Python 3](https://www.python.org/downloads/windows/) para servir la página, o Visual Studio Code con Live Server.
- Un navegador y conexión a Internet para descargar las herramientas y el modelo inicial.
- Memoria y espacio disponibles para el modelo. El rendimiento depende del equipo; la primera consulta puede tardar más por la carga del modelo.

No necesitas una placa Raspberry Pi para probar el asistente ni una clave de API para utilizar el modelo local. Después de las descargas, las consultas se procesan en Ollama local.

## 1. Descargar el proyecto

Abre PowerShell y ejecuta:

```powershell
git clone https://github.com/BorBP/Proyecto_RaspberryPi.git
cd Proyecto_RaspberryPi
```

Si descargaste el ZIP, extráelo y abre una terminal dentro de la carpeta que contiene `index.html` y `Modelfile`. Ejecuta todos los comandos de creación desde esa carpeta.

## 2. Iniciar Ollama y descargar el modelo base

Abre Ollama desde el menú Inicio. En PowerShell comprueba la instalación y descarga el modelo:

```powershell
ollama --version
ollama pull qwen3.5:2b
ollama list
```

Comprueba que el servidor responde:

```powershell
Invoke-RestMethod -Uri 'http://localhost:11434/'
```

Si Ollama no está iniciado como aplicación, puedes ejecutar `ollama serve` en una terminal y dejarla abierta. Usa una sola instancia del servidor. Si el puerto ya está ocupado por Ollama, no necesitas iniciar otro servidor.

## 3. Crear el asistente personalizado

```powershell
ollama create asistente-raspberrypi -f .\Modelfile
ollama list
ollama show --parameters asistente-raspberrypi
ollama show --modelfile asistente-raspberrypi
```

Espera a que `create` finalice correctamente. El nombre `asistente-raspberrypi` debe aparecer en `ollama list` antes de seleccionarlo en la web.

El modelo base es `qwen3.5:2b`. `SYSTEM` define el rol del tutor, incorpora un resumen de las fuentes y establece el formato de las respuestas. La personalización usa instrucciones y parámetros de inferencia; no modifica los pesos mediante entrenamiento ni implementa recuperación automática de documentos.

## 4. Crear las cinco variantes de prueba

Para que funcionen todas las opciones del selector, ejecuta:

```powershell
ollama create raspberrypi-a -f .\Modelfile-A
ollama create raspberrypi-b -f .\Modelfile-B
ollama create raspberrypi-c -f .\Modelfile-C
ollama create raspberrypi-d -f .\Modelfile-D
ollama create raspberrypi-e -f .\Modelfile-E
ollama list
```

| Modelo | Archivo | temperature | num_ctx | num_predict | num_batch |
| --- | --- | ---: | ---: | ---: | ---: |
| `asistente-raspberrypi` | `Modelfile` | 0.2 | 4096 | 600 | 80 |
| `raspberrypi-a` | `Modelfile-A` | 0.2 | 4096 | 600 | 80 |
| `raspberrypi-b` | `Modelfile-B` | **0.8** | 4096 | 600 | 80 |
| `raspberrypi-c` | `Modelfile-C` | 0.2 | **2048** | 600 | 80 |
| `raspberrypi-d` | `Modelfile-D` | 0.2 | 4096 | **150** | 80 |
| `raspberrypi-e` | `Modelfile-E` | 0.2 | 4096 | 600 | **256** |

Cada variante cambia un parámetro respecto de A. El modelo final actualmente coincide con A.

- `temperature`: modifica la variación del muestreo de las respuestas.
- `num_ctx`: establece la ventana de contexto en tokens.
- `num_predict`: limita la cantidad de tokens generados.
- `num_batch`: configura el tamaño del lote de procesamiento de entrada; su efecto en el rendimiento depende del equipo.

Estos valores son configuraciones para experimentar, no resultados de rendimiento garantizados.

## 5. Probar desde la terminal

```powershell
ollama run asistente-raspberrypi
```

Escribe, por ejemplo:

```text
¿Puedo conectar directamente una señal de 5 V a un GPIO de Raspberry Pi 4?
```

Espera la respuesta. Para salir del chat escribe `/bye`. Puedes repetir la prueba con `ollama run raspberrypi-a` o `ollama run qwen3.5:2b`.

## 6. Abrir la interfaz web

Desde la carpeta del proyecto, en otra terminal, inicia un servidor local:

```powershell
py -m http.server 5500 --bind 127.0.0.1
```

Si `py` no está disponible pero Python está instalado, usa:

```powershell
python -m http.server 5500 --bind 127.0.0.1
```

Deja esa terminal abierta y visita **[http://127.0.0.1:5500/index.html](http://127.0.0.1:5500/index.html)**.

Como alternativa, abre esta carpeta en VS Code y usa **Open with Live Server** sobre `index.html`. Comprueba que la dirección y el puerto coincidan con los que utilizas.

`localhost:11434` es el servidor de Ollama; la interfaz del proyecto está en el puerto `5500`. La página se conecta a `http://localhost:11434/api/generate`, por lo que Ollama debe ejecutarse en el mismo equipo que el navegador con esta configuración.

### Usar el chat

1. Selecciona el tutor final, Qwen original o una variante creada previamente.
2. Escribe una consulta y pulsa **Enviar**.
3. Espera a que termine la generación y se muestre la respuesta.
4. Revisa el nombre del modelo y el tiempo que aparece en su mensaje.

La interfaz envía `stream: false` y `think: false`: espera la respuesta completa y solicita desactivar la salida de razonamiento. Cada consulta se envía sin el historial anterior. El chat visible sirve para revisar respuestas, pero no se conserva al recargar o limpiar.

El tiempo mostrado mide la espera total en el navegador, incluida la carga cuando corresponda. Si Ollama devuelve `done_reason: "length"`, se muestra una nota de respuesta posiblemente incompleta. Esa nota indica el límite de generación; no detecta todos los posibles errores u omisiones del contenido. La espera máxima configurada es de cinco minutos.

## 7. Si el navegador muestra un error CORS

Si la consola del navegador informa que el origen está bloqueado, configura los orígenes locales permitidos. En PowerShell ejecuta:

```powershell
[Environment]::SetEnvironmentVariable('OLLAMA_ORIGINS', 'http://127.0.0.1:5500,http://localhost:5500', 'User')
```

Cierra completamente Ollama desde su icono de la bandeja del sistema y vuelve a abrirlo desde Inicio para que reciba la variable. Si usas `ollama serve`, detén ese proceso, abre una nueva terminal y vuelve a iniciarlo. Recarga después la página.

Si utilizas otro puerto, sustituye o añade su origen completo en la variable. No es necesario permitir todos los sitios con `*`.

## 8. Comparar los modelos

Envía exactamente la misma pregunta al modelo base y al tutor personalizado:

1. ¿Cómo preparo Raspberry Pi OS con una microSD vacía y un PC Windows?
2. ¿Cómo me conecto sin monitor si todavía no conozco la IP ni he creado el usuario?
3. ¿Puedo conectar directamente una señal de 5 V a un GPIO de Raspberry Pi 4?

Compara exactitud, claridad, precauciones, uso de las referencias y completitud. Guarda capturas con el modelo y el tiempo visibles antes de limpiar el chat.

Para estudiar parámetros, compara A con B, C, D y E manteniendo la misma entrada y las mismas condiciones. Repite las pruebas y distingue la primera carga de las consultas posteriores. Usa entradas largas para estudiar contexto y procesamiento de entrada; una pregunta corta no demuestra la capacidad de contexto. Si comparas el original con el personalizado sin igualar sus parámetros, las diferencias pueden deberse tanto a las instrucciones como a la configuración.

## 9. Cambiar la configuración final

Edita las instrucciones o los parámetros en `Modelfile`, guarda el archivo y vuelve a crear el modelo:

```powershell
ollama create asistente-raspberrypi -f .\Modelfile
ollama show --parameters asistente-raspberrypi
```

Editar el archivo sin ejecutar `ollama create` no actualiza el modelo instalado. Si cambias el nombre del modelo, actualiza también el `value` correspondiente en el selector de `index.html`.

## Solución de problemas

| Problema | Qué comprobar |
| --- | --- |
| `ollama` no se reconoce | Instala Ollama y abre una terminal nueva. |
| No conecta con Ollama | Abre la aplicación y comprueba `http://localhost:11434/`. |
| Modelo no encontrado | Revisa `ollama list` y ejecuta su comando de creación. |
| `Modelfile` no encontrado | Abre la terminal en la carpeta del proyecto o usa la ruta real del archivo. |
| La página no aparece o devuelve 404 | Ejecuta el servidor desde la carpeta que contiene `index.html`. |
| Puerto 5500 ocupado | Usa el servidor existente si corresponde al proyecto, o cambia de puerto y ajusta el origen permitido si hace falta. |
| Error CORS | Sigue el paso 7 y reinicia Ollama completamente. |
| El navegador muestra texto sin la interfaz en el puerto 11434 | Abre el index en `127.0.0.1:5500/index.html`. |
| Respuesta lenta o espera agotada | Revisa recursos disponibles y prueba una consulta más corta; la carga inicial puede tardar. |
| Respuesta incompleta | Revisa `num_predict`, aumenta el límite si corresponde y vuelve a crear el modelo. |

## Detener y volver a iniciar

Para detener el servidor de la web, pulsa `Ctrl+C` en su terminal. Para descargar un modelo de la memoria puedes usar `ollama stop asistente-raspberrypi`; sus archivos permanecen instalados.

En usos posteriores solo necesitas abrir Ollama, iniciar de nuevo el servidor de la web y abrir el index. No hace falta descargar ni crear los modelos otra vez, salvo que los elimines o cambies su configuración.

## Archivos y referencias

- `index.html`: interfaz con estilos y JavaScript incluidos en el mismo archivo.
- `Modelfile`: instrucciones y parámetros del tutor final.
- `Modelfile-A` a `Modelfile-E`: configuraciones para comparar parámetros.

Contenido de referencia incorporado al mensaje de sistema:

- [D1 — Raspberry Pi Getting started](https://www.raspberrypi.com/documentation/computers/getting-started.html).
- [D2 — Raspberry Pi Remote access](https://www.raspberrypi.com/documentation/computers/remote-access.html).
- [D3 — Raspberry Pi Computer hardware y GPIO](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html).

Documentación de las herramientas:

- [Ollama Modelfile](https://docs.ollama.com/modelfile).
- [Ollama API Generate](https://docs.ollama.com/api/generate).
- [Ollama FAQ y variables de entorno](https://docs.ollama.com/faq).
