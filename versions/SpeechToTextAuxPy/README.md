# 🎤 SpeechToTextAuxPy

**Reconocimiento de voz continuo en español desde la entrada auxiliar (line-in) usando SAPI de Windows**

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)
[![Windows 10](https://img.shields.io/badge/platform-Windows%2010-0078D6.svg)](https://www.microsoft.com/windows)
[![.NET Framework](https://img.shields.io/badge/.NET-4.7.2+-512BD4.svg)](https://dotnet.microsoft.com/)

---

## 📋 Descripción

SpeechToTextAuxPy captura audio en tiempo real desde el **dispositivo de entrada predeterminado** de Windows (micrófono o entrada auxiliar line-in) y transcribe la voz a texto usando el motor **SAPI** (Speech Application Programming Interface) de Windows, accedido a través de la biblioteca **pythonnet** que conecta Python con **System.Speech** de .NET Framework.

El programa se ejecuta de forma continua, mostrando cada frase reconocida en la consola con un timestamp `[HH:MM:SS]` hasta que el usuario presiona `Ctrl+C`.

---

## ⚙️ Principio de funcionamiento

```
┌──────────────┐     ┌───────────────────┐     ┌──────────────────┐
│  Micrófono   │────▶│  SAPI (Windows)   │────▶│  System.Speech   │
│  / Line-in   │     │  Reconocimiento    │     │  .NET DLL        │
└──────────────┘     └───────────────────┘     └────────┬─────────┘
                                                         │
                                                         │ pythonnet (clr)
                                                         ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Consola Windows │◀────│  Console_utils   │◀────│  SapiEngine      │
│  [HH:MM:SS] txt  │     │  (colorama)      │     │  (Python Class)  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

1. **SAPI de Windows** captura el audio desde el dispositivo de entrada predeterminado.
2. **System.Speech.Recognition** (`.NET Framework 4.7.2+`) procesa el audio y realiza el reconocimiento de voz.
3. **pythonnet** (`clr`) actúa como puente entre Python y .NET, permitiendo invocar las API de `System.Speech.dll` directamente desde código Python.
4. **SapiEngine** (clase Python) envuelve la lógica de inicialización, inicio y parada del motor de reconocimiento.
5. **console_utils** imprime el texto reconocido en la consola con formato y colores (vía `colorama`).

---

## 📦 Requisitos del sistema

| Requisito | Detalle |
|---|---|
| **Sistema operativo** | Windows 10 (cualquier edición) |
| **Python** | 3.10 o superior |
| **.NET Framework** | 4.7.2 o superior (incluido en Windows 10) |
| **Paquete de voz español** | Paquete de reconocimiento de voz en español de Windows |
| **Dispositivo de audio** | Micrófono o entrada auxiliar (line-in) funcional |

---

## 🔧 Instalación

### 1. Instalar dependencias de Python

```batch
pip install -r requirements.txt
```

Esto instalará:
- [`pythonnet`](https://github.com/pythonnet/pythonnet) ≥ 3.0.0 — Puente Python/.NET
- [`colorama`](https://github.com/tartley/colorama) ≥ 0.4.6 — Colores en consola Windows

### 2. Instalar paquete de voz en español (Windows)

El paquete de reconocimiento de voz en español **no viene preinstalado** en Windows 10. Para instalarlo:

1. Abrir **Configuración** → **Hora e idioma** → **Voz**
2. En *Administrar voces*, hacer clic en **Agregar voces**
3. Seleccionar **Español (España)** o **Español (México)**
4. Esperar a que se descargue e instale
5. Alternativamente: **Configuración** → **Privacidad y seguridad** → **Voz** → activar *Reconocimiento de voz en línea*

> ⚠️ **Importante:** Sin este paquete, SAPI no podrá reconocer el idioma `es-ES` y el programa fallará con un mensaje de error.

### 3. Configurar la entrada auxiliar (line-in)

Si usas una entrada auxiliar (line-in) en lugar de micrófono:

1. Conecta el cable de audio a la toma **Line-In** (azul) de tu PC
2. Abre **Configuración** → **Sonido** → **Panel de control de sonido**
3. En la pestaña **Grabación**, localiza **Entrada auxiliar** o **Line-In**
4. Haz clic derecho → **Habilitar** (si está deshabilitado)
5. Selecciona **Establecer como dispositivo predeterminado**
6. Abre **Propiedades** → **Niveles** y ajusta el volumen según sea necesario

---

## 🚀 Uso

### Desde la terminal

```batch
python src/main.py
```

### Con doble clic

Ejecuta `run.bat` (instala dependencias automáticamente si faltan y lanza el programa).

---

## 💻 Ejemplo de salida en consola

```
[10:15:32] === SpeechToTextAuxPy ===
[10:15:32] Reconocimiento de voz continuo mediante SAPI (System.Speech)
[10:15:32] Motor SAPI inicializado correctamente (idioma: es-ES)
[10:15:32] Iniciando reconocimiento de voz en español...
[10:15:32] Habla al micrófono/entrada auxiliar. Ctrl+C para salir.
[10:15:35] bienvenidos al curso de inteligencia artificial
[10:15:40] hoy vamos a hablar sobre procesamiento del lenguaje natural
[10:15:48] el reconocimiento de voz es una tecnología fascinante
^C
[10:15:52] Ctrl+C recibido. Deteniendo...
[10:15:52] Reconocimiento de voz detenido.
[10:15:52] Programa terminado.
```

---

## 🧠 Funcionamiento interno

### Flujo de ejecución

1. **`main.py`** configura la consola en UTF-8 y registra manejadores de señal (`SIGINT`/`SIGTERM`)
2. Crea una instancia de **`SapiEngine`** con el idioma `es-ES`
3. El constructor de `SapiEngine`:
   - Importa `clr` (pythonnet)
   - Agrega referencia a `System.Speech.dll`
   - Crea un `SpeechRecognitionEngine` con `CultureInfo("es-ES")`
   - Configura la entrada al dispositivo de audio predeterminado
   - Carga una `DictationGrammar` para texto libre
   - Registra el evento `SpeechRecognized`
4. **`engine.start()`** llama a `RecognizeAsync(RecognizeMode.Multiple)`, iniciando la escucha continua
5. `main.py` entra en un bucle de espera (`signal.pause()` con fallback `time.sleep(0.5)`)
6. Cuando SAPI reconoce voz, dispara el evento `SpeechRecognized`
7. El callback `_on_speech_recognized` extrae el texto de `args.Result.Text` y lo imprime
8. Al presionar `Ctrl+C`, se invoca `engine.stop()` que llama a `RecognizeAsyncCancel()` y `Dispose()`

### Manejo de errores

| Escenario | Mensaje | Código salida |
|---|---|---|
| `pythonnet` no instalado | `pip install -r requirements.txt` | `1` |
| Paquete de voz español faltante | `Verifica que el paquete de voz español esté instalado` | `2` |
| Sin dispositivo de audio | `No se encontró dispositivo de entrada de audio` | `2` |
| Error inesperado | Mensaje genérico con descripción | `3` |

---

## ⚠️ Troubleshooting

### "No se encontró dispositivo de entrada de audio disponible"
- Conecta un micrófono o cable a la entrada auxiliar
- Verifica que el dispositivo esté habilitado en **Configuración → Sonido → Administrar dispositivos**
- Establece el dispositivo como predeterminado

### "El idioma 'es-ES' no está soportado"
- Instala el paquete de voz en español (ver sección *Instalación* arriba)
- Verifica en **Configuración → Hora e idioma → Voz → Administrar voces**

### "pythonnet no está instalado"
- Ejecuta `pip install -r requirements.txt`
- Verifica que uses Python 3.10 o superior

### No se reconoce nada
- Habla más cerca del micrófono
- Verifica que el dispositivo de entrada sea el correcto
- Comprueba los niveles de volumen en **Propiedades del dispositivo → Niveles**
- Asegúrate de que el paquete de voz español esté correctamente instalado

### Caracteres extraños en la consola
- El programa configura UTF-8 automáticamente
- Si usas `run.bat`, se ejecuta `chcp 65001` para cambiar a UTF-8
- Alternativamente, ejecuta `chcp 65001` manualmente antes de lanzar el programa

---

## 📁 Estructura del proyecto

```
SpeechToTextAuxPy/
├── README.md              ← Este archivo
├── requirements.txt       ← Dependencias Python
├── run.bat                ← Lanzador para Windows
├── PLAN.md                ← Plan de implementación
├── AUDIT_REPORT.md        ← Informe de auditoría
├── src/
│   ├── main.py            ← Punto de entrada
│   ├── sapi_engine.py     ← Motor SAPI vía pythonnet
│   └── console_utils.py   ← Utilidades de consola
└── docs/
    ├── datasheet.md       ← Ficha técnica para usuario final
    └── desarrollador.md   ← Documentación técnica para desarrolladores
```

---

## 🛠️ Tecnologías utilizadas

- **Python 3.10+** — Lenguaje de programación
- **[pythonnet](https://github.com/pythonnet/pythonnet)** — Puente entre Python y .NET
- **System.Speech (.NET Framework 4.7.2+)** — API de reconocimiento de voz de Windows
- **SAPI (Speech API)** — Motor de reconocimiento de voz de Windows
- **[colorama](https://github.com/tartley/colorama)** — Colores en consola Windows

---

## 📄 Licencia

Este proyecto es de uso libre y educativo.

---

## 👤 Autor

Proyecto desarrollado como parte del ecosistema **AG.CreadorContenido**.
