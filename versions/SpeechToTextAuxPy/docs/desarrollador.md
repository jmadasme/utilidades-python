# 🧑‍💻 Documentación técnica para desarrolladores — SpeechToTextAuxPy

---

## Índice

- [Arquitectura](#arquitectura)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Descripción detallada de módulos](#descripción-detallada-de-módulos)
  - [main.py](#mainpy)
  - [sapi_engine.py](#sapi_enginepy)
  - [console_utils.py](#console_utilspy)
- [APIs utilizadas](#apis-utilizadas)
- [Flujo de eventos](#flujo-de-eventos)
- [Manejo de errores](#manejo-de-errores)
- [Guía para extender/modificar](#guía-para-extendermodificar)

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────┐
│                      main.py                            │
│  Punto de entrada, señal SIGINT/SIGTERM, bucle ppal.    │
└────┬──────────────┬──────────────────────┬──────────────┘
     │              │                      │
     ▼              ▼                      ▼
┌──────────┐ ┌──────────────┐ ┌──────────────────────┐
│SapiEngine│ │console_utils│ │   signal / sys        │
│(clr/.NET)│ │ (colorama)   │ │  (biblioteca std)    │
└────┬─────┘ └──────────────┘ └──────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│         System.Speech (.NET)             │
│  SpeechRecognitionEngine + DictationGrammar│
└──────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│      SAPI (Windows Speech API)           │
│  Reconocimiento de voz offline           │
└──────────────────────────────────────────┘
```

### Principio de diseño

- **Separación de responsabilidades**: Cada módulo tiene una única responsabilidad bien definida.
- **Degradación elegante**: El programa funciona incluso sin `colorama` (sin colores).
- **Manejo defensivo**: Todas las operaciones propensas a fallo están envueltas en `try/except`.
- **Portabilidad limitada**: Conscientemente diseñado solo para Windows 10.

---

## Estructura del proyecto

```
SpeechToTextAuxPy/
├── README.md              ← Documentación principal
├── requirements.txt       ← pythonnet>=3.0.0, colorama>=0.4.6
├── run.bat                ← Lanzador (chcp 65001 + pip install + python src/main.py)
├── PLAN.md                ← Especificación original
├── AUDIT_REPORT.md        ← Informe de auditoría de código
├── src/
│   ├── main.py            ← Punto de entrada, manejo de señales, orquestación
│   ├── sapi_engine.py     ← SapiEngine: envoltura de System.Speech.Recognition
│   └── console_utils.py   ← Impresión con formato, timestamp y color
└── docs/
    ├── datasheet.md       ← Ficha técnica para usuario final
    └── desarrollador.md   ← Esta documentación
```

---

## Descripción detallada de módulos

### main.py

**Propósito**: Punto de entrada del programa. Orquesta la inicialización, ejecución y finalización limpia.

#### Variables globales

```python
_engine: SapiEngine = None
```
Referencia global al motor SAPI para que el manejador de señales pueda acceder a él.

#### Funciones

##### `signal_handler(signal_num: int, frame) -> None`
- **Propósito**: Manejador de `SIGINT` (Ctrl+C) y `SIGTERM`.
- **Flujo**:
  1. Imprime mensaje de notificación
  2. Llama a `_engine.stop()` si el motor existe
  3. Imprime mensaje de finalización
  4. Termina el proceso con `sys.exit(0)`

##### `main() -> None`
- **Propósito**: Función principal del programa.
- **Flujo**:
  1. Registra manejadores `SIGINT` y `SIGTERM` (este último con fallback `AttributeError`)
  2. Imprime cabecera del programa
  3. Crea `SapiEngine(language="es-ES")`
  4. Llama a `engine.start()` para iniciar reconocimiento asíncrono
  5. Entra en bucle con `signal.pause()` (fallback `time.sleep(0.5)` en Windows)
  6. Captura y maneja `ImportError`, `RuntimeError` y `Exception` genérica

#### Manejo de señales en Windows

`signal.pause()` no está disponible en Windows. El código usa:

```python
try:
    signal.pause()
except AttributeError:
    import time
    time.sleep(0.5)
```

Esto bloquea el hilo principal con esperas de 0.5s, permitiendo que el callback de SAPI se ejecute en el hilo de .NET. Es suficiente para mantener el programa vivo.

---

### sapi_engine.py

**Propósito**: Envoltura (wrapper) de la clase `System.Speech.Recognition.SpeechRecognitionEngine` de .NET, accesible mediante `pythonnet` (`clr`).

#### Clase `SapiEngine`

```python
SapiEngine(language: str = "es-ES")
```

##### `__init__(self, language: str = "es-ES")`

**Parámetros**:
- `language`: Código ISO del idioma (ej: `"es-ES"`, `"en-US"`, `"es-MX"`). Por defecto `"es-ES"`.

**Atributos de instancia**:
- `self._language: str` — Código de idioma
- `self._engine: Optional[Any]` — Instancia de `SpeechRecognitionEngine` (.NET)
- `self._grammar: Optional[Any]` — Instancia de `DictationGrammar` (.NET)
- `self._RecognizeMode: type` — Referencia al enum `RecognizeMode` (.NET) para evitar reimportaciones

**Flujo de inicialización**:
1. Importa `clr` de pythonnet (lanza `ImportError` si no existe)
2. `clr.AddReference("System.Speech")` — Carga el assembly .NET
3. Importa desde `System.Speech.Recognition`:
   - `DictationGrammar`
   - `RecognizeMode`
   - `SpeechRecognitionEngine`
4. Importa `System.Globalization.CultureInfo`
5. Guarda `RecognizeMode` como `self._RecognizeMode`
6. Crea `CultureInfo(language)` para el idioma solicitado
7. Crea `SpeechRecognitionEngine(cultura)`
8. `SetInputToDefaultAudioDevice()` — Configura el dispositivo predeterminado
9. Crea `DictationGrammar()` y la carga con `LoadGrammar()`
10. Registra `_on_speech_recognized` como delegado del evento `SpeechRecognized`

**Manejo de errores**:
- Detecta en el mensaje de error si la causa es:
  - Idioma no soportado (contiene `"culture"` o `"language"`) → sugiere instalar paquete de voz
  - Dispositivo de audio no encontrado (contiene `"device"` o `"audio"`) → sugiere conectar micrófono
  - Otros → muestra el error genérico
- Envuelve el error en `RuntimeError` para que `main.py` lo capture

##### `_on_speech_recognized(self, sender: Any, args: Any) -> None`

**Propósito**: Callback ejecutado por SAPI cuando se reconoce una frase.

**Parámetros**:
- `sender`: Objeto `SpeechRecognitionEngine` que disparó el evento
- `args`: `SpeechRecognizedEventArgs` con el resultado

**Flujo**:
1. Verifica que `args`, `args.Result` y `args.Result.Text` no sean `None`
2. Extrae el texto y elimina espacios con `.strip()`
3. Si hay texto, llama a `console_utils.print_recognized(texto)`
4. Libera `args.Result.Dispose()` para evitar fugas de memoria COM

> **Importante**: Este callback se ejecuta en un hilo de .NET, no en el hilo principal de Python. No debe realizar operaciones bloqueantes ni acceder a recursos no thread-safe sin sincronización.

##### `start(self) -> None`

**Propósito**: Inicia el reconocimiento de voz asíncrono continuo.

**Flujo**:
1. Verifica que `self._engine` no sea `None`
2. Llama a `self._engine.RecognizeAsync(self._RecognizeMode.Multiple)` — modo de reconocimiento continuo
3. Imprime mensajes informativos en consola

**`RecognizeMode.Multiple`**: Indica a SAPI que continúe reconociendo frases de forma indefinida hasta que se cancele explícitamente.

##### `stop(self) -> None`

**Propósito**: Detiene el reconocimiento y libera recursos.

**Flujo**:
1. Si `self._engine` no es `None`:
   - Llama a `RecognizeAsyncCancel()` — detiene el reconocimiento
   - Llama a `engine.Dispose()` — libera recursos COM/.NET
   - Establece `self._engine = None` y `self._grammar = None`
2. Si ya estaba detenido, imprime mensaje informativo

> Es seguro llamar a `stop()` múltiples veces.

---

### console_utils.py

**Propósito**: Utilidades de formateo de salida para la consola con timestamp, colores y manejo de `colorama` opcional.

#### Variables de módulo

```python
COLORAMA_DISPONIBLE: bool
```
Indica si `colorama` está instalado. Se determina en tiempo de importación.

Si `colorama` no está disponible, se definen clases `Fore` y `Style` con atributos vacíos como fallback.

#### Funciones

##### `_formato_timestamp() -> str`
- **Propósito**: Genera el timestamp actual en formato `[HH:MM:SS]`.
- **Privada** (por convención `_` prefijo): uso interno del módulo.

##### `print_timestamp(color: str = Fore.CYAN, stream=sys.stdout) -> None`
- **Propósito**: Imprime el timestamp formateado en el color y flujo especificados.
- **Parámetros**:
  - `color`: Color `Fore` de colorama (default: cian)
  - `stream`: Flujo de salida (default: `sys.stdout`)
- Realiza `flush()` para garantizar salida en tiempo real.
- Si `colorama` no está disponible, imprime sin color.

##### `print_recognized(text: str) -> None`
- **Propósito**: Imprime texto reconocido con timestamp en cian y texto en blanco brillante.
- **Formato**: `[HH:MM:SS] texto reconocido`
- Usa `sys.stdout`.

##### `print_info(message: str) -> None`
- **Propósito**: Imprime mensaje informativo con timestamp en verde.
- **Formato**: `[HH:MM:SS] mensaje`
- Usa `sys.stdout`.

##### `print_error(message: str) -> None`
- **Propósito**: Imprime mensaje de error con timestamp en rojo.
- **Formato**: `[HH:MM:SS] ERROR: mensaje`
- Usa `sys.stderr` (todo unificado en el mismo flujo).

#### Diseño de `colorama` opcional

```python
try:
    from colorama import init, Fore, Style
    init()
    COLORAMA_DISPONIBLE = True
except ImportError:
    COLORAMA_DISPONIBLE = False
    class Fore:
        CYAN = ""; GREEN = ""; RED = ""; WHITE = ""; RESET = ""
    class Style:
        BRIGHT = ""; RESET_ALL = ""
```

Esto permite que el programa funcione sin `colorama` instalado (sin colores, pero con toda la funcionalidad intacta).

---

## APIs utilizadas

### pythonnet (`clr`)

- **Documentación**: https://github.com/pythonnet/pythonnet
- **Versión**: ≥ 3.0.0
- **Uso**: `import clr` → `clr.AddReference("System.Speech")` → importar tipos .NET como módulos Python
- **Mecanismo**: Carga el runtime de .NET en el proceso de Python y expone los assemblies administrados como módulos Python.

### System.Speech (.NET Framework 4.7.2+)

- **Assembly**: `System.Speech.dll` (incluido en Windows)
- **Namespace**: `System.Speech.Recognition`
- **Clases utilizadas**:

| Clase .NET | Propósito |
|---|---|
| `SpeechRecognitionEngine` | Motor principal de reconocimiento. Gestiona la configuración, inicio y parada. |
| `DictationGrammar` | Gramática de dictado libre (reconoce palabras sueltas, no comandos). |
| `RecognizeMode` | Enum con valores `Single` (una frase) y `Multiple` (continuo). |
| `CultureInfo` (de `System.Globalization`) | Configura el idioma/región para el motor de reconocimiento. |

### signal (biblioteca estándar)

- **Propósito**: Manejar `SIGINT` (Ctrl+C) y `SIGTERM` para finalización limpia.
- **Limitación Windows**: `signal.pause()` no está disponible. Se usa fallback con `time.sleep(0.5)`.

### colorama

- **Documentación**: https://github.com/tartley/colorama
- **Propósito**: Secuencias de escape ANSI para colores en consola Windows.
- **Uso**: `init()` al importar, luego `Fore.CYAN`, `Fore.GREEN`, `Fore.RED`, `Fore.WHITE`, `Style.BRIGHT`, `Style.RESET_ALL`.

---

## Flujo de eventos

### Diagrama de secuencia

```
main.py                    SapiEngine                  System.Speech
  │                            │                            │
  ├── main()                   │                            │
  │   ├── signal.signal()      │                            │
  │   ├── SapiEngine("es-ES")──┤                            │
  │   │                        ├── clr.AddReference()──────▶│
  │   │                        ├── SpeechRecognitionEngine()│
  │   │                        ├── SetInputToDefaultAudio()▶│
  │   │                        ├── LoadGrammar(Dictation)──▶│
  │   │                        ├── event += callback         │
  │   │                        │◀── Inicializado            │
  │   ├── engine.start()───────┤                            │
  │   │                        ├── RecognizeAsync(Multiple)▶│
  │   │                        │◀── Escuchando...           │
  │   │◀── "Iniciando..."      │                            │
  │   │                        │                            │
  │   ├── bucle espera ◀───────┤                            │
  │   │                        │                            │
  │   │                  (usuario habla)                    │
  │   │                        │◀── SpeechRecognized event  │
  │   │                        ├── _on_speech_recognized()  │
  │   │                        │   ├── args.Result.Text     │
  │   │                        │   ├── print_recognized()   │
  │   │                        │   └── args.Result.Dispose()│
  │   │                        │                            │
  │   │                  (usuario presiona Ctrl+C)          │
  │   ├── signal_handler()     │                            │
  │   │   ├── engine.stop()────┤                            │
  │   │   │                    ├── RecognizeAsyncCancel()──▶│
  │   │   │                    ├── Dispose()───────────────▶│
  │   │   │                    │◀── Recursos liberados      │
  │   │   ├── sys.exit(0)      │                            │
  │   │                        │                            │
```

### Consideraciones sobre threading

1. **Hilo principal (Python)**: Ejecuta `main.py`, el bucle de espera y los manejadores de señal.
2. **Hilo de .NET**: SAPI ejecuta el reconocimiento en hilos administrados por .NET.
3. **Callback `_on_speech_recognized`**: Se ejecuta en un hilo de .NET. El código Python en el callback debe ser thread-safe.
4. **Salida a consola**: `print_recognized()` incluye `flush()` para garantizar visualización inmediata desde cualquier hilo.

No hay condiciones de carrera críticas porque:
- El callback solo escribe en consola (operación atómica a nivel de syscall)
- `engine.stop()` se llama desde el manejador de señal (hilo principal)
- La variable `_engine` se establece a `None` solo después de `Dispose()`

---

## Manejo de errores

### Estrategia general

- **Errores predecibles**: Se capturan específicamente con `try/except` y se muestran mensajes descriptivos.
- **Errores impredecibles**: Se capturan con `except Exception` genérico y se relanzan como `RuntimeError`.
- **Degradación elegante**: El programa nunca termina abruptamente sin mensaje.

### Jerarquía de captura en `main()`

```
try:
    _engine = SapiEngine("es-ES")
    _engine.start()
    while True: ...
except ImportError:      ← Dependencia faltante (pythonnet)
    → sys.exit(1)
except RuntimeError:     ← Error SAPI (idioma, audio)
    → sys.exit(2)
except Exception:        ← Error inesperado
    → sys.exit(3)
```

### Errores detectados en `sapi_engine.py`

| Condición | Detección | Mensaje |
|---|---|---|
| `pythonnet` no instalado | `except ImportError` | `pip install -r requirements.txt` |
| Idioma no soportado | `"culture"` o `"language"` en mensaje | Instalar paquete de voz español |
| Sin dispositivo de audio | `"device"` o `"audio"` en mensaje | Conectar micrófono/line-in |
| Error genérico SAPI | `except Exception` | Mensaje original de .NET |

### Errores en callback

El callback `_on_speech_recognized` envuelve todo en `try/except` para evitar que un error en el procesamiento de una frase detenga el programa completo. La liberación de `args.Result.Dispose()` también tiene su propio `try/except`.

---

## Guía para extender/modificar

### Agregar un nuevo idioma

Modificar en `main.py`:

```python
_engine = SapiEngine(language="es-MX")  # Español mexicano
# o
_engine = SapiEngine(language="en-US")  # Inglés americano
```

### Usar un dispositivo de entrada específico

Actualmente `SetInputToDefaultAudioDevice()` usa el dispositivo predeterminado. Para seleccionar uno específico:

```python
# En sapi_engine.py, reemplazar:
self._engine.SetInputToDefaultAudioDevice()

# Con:
# Obtener dispositivos de audio disponibles
recognizers = SpeechRecognitionEngine.InstalledRecognizers()
# ... seleccionar uno específico
```

### Guardar transcripción en archivo

Agregar en `main.py` un manejador de archivo:

```python
import sys
from sapi_engine import SapiEngine
from datetime import datetime

archivo_log = open(f"transcripcion_{datetime.now():%Y%m%d_%H%M%S}.txt", "w", encoding="utf-8")

class SapiEngineConLog(SapiEngine):
    def _on_speech_recognized(self, sender, args):
        super()._on_speech_recognized(sender, args)
        if args and args.Result and args.Result.Text:
            archivo_log.write(f"{args.Result.Text}\n")
            archivo_log.flush()
```

### Agregar filtrado o post-procesamiento

Modificar `_on_speech_recognized` en `sapi_engine.py`:

```python
def _on_speech_recognized(self, sender, args):
    try:
        if args and args.Result and args.Result.Text:
            texto = args.Result.Text.strip()
            texto = texto.lower()  # Normalizar a minúsculas
            # Aplicar reglas de negocio...
            if texto and len(texto) > 2:  # Ignorar fragmentos muy cortos
                console_utils.print_recognized(texto)
    except Exception as e:
        console_utils.print_error(f"Error en callback: {e}")
```

### Agregar un modo de solo comandos

Reemplazar `DictationGrammar` con `Grammar` basado en lista de frases:

```python
# En lugar de DictationGrammar:
from System.Speech.Recognition import Choices, GrammarBuilder, Grammar

choices = Choices()
choices.Add("iniciar grabación")
choices.Add("detener grabación")
choices.Add("pausar")
choices.Add("continuar")

builder = GrammarBuilder(choices)
grammar = Grammar(builder)
self._engine.LoadGrammar(grammar)
```

### Manejar evento `SpeechRecognitionRejected`

Para detectar cuándo SAPI intentó reconocer pero no tuvo suficiente confianza:

```python
self._engine.SpeechRecognitionRejected += self._on_speech_rejected

def _on_speech_rejected(self, sender, args):
    console_utils.print_info("[Ruido detectado, no se reconoció voz clara]")
```

---

## Pruebas

### Ejecución manual

```batch
python src/main.py
```

### Casos de prueba sugeridos

| ID | Escenario | Acción | Resultado esperado |
|---|---|---|---|
| CP-01 | Inicio normal | Ejecutar programa | "Iniciando reconocimiento..." |
| CP-02 | Reconocimiento | Hablar frente al micrófono | Texto aparece con timestamp |
| CP-03 | Finalización | Ctrl+C | "Programa terminado." + código 0 |
| CP-04 | Sin audio | Deshabilitar micrófono | Error sobre dispositivo de audio |
| CP-05 | Sin paquete español | Desinstalar voz español | Error sobre idioma no soportado |
| CP-06 | Sin pythonnet | Desinstalar pythonnet | Error sobre dependencia faltante |

---

## Notas de implementación

- **pythonnet y recolector de basura**: Los objetos .NET creados via `clr` son administrados por el GC de .NET. Es recomendable llamar a `Dispose()` explícitamente en objetos que implementan `IDisposable` (como `SpeechRecognitionEngine`).
- **UTF-8 en Windows**: La consola de Windows usa por defecto la página de códigos activa (generalmente 850 o 1252). `run.bat` ejecuta `chcp 65001` para cambiar a UTF-8. El código Python también intenta `sys.stdout.reconfigure(encoding="utf-8")`.
- **signal.pause()**: No disponible en Windows. El fallback `time.sleep(0.5)` introduce una latencia de hasta 500ms en la detección de Ctrl+C, pero es suficiente para una herramienta CLI.

---

## Referencias

- [pythonnet — Documentación oficial](https://pythonnet.github.io/)
- [System.Speech.Recognition (.NET Framework) — Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/api/system.speech.recognition)
- [SAPI (Microsoft Speech API) — Microsoft Docs](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/ms723627(v=vs.85))
- [colorama — PyPI](https://pypi.org/project/colorama/)
