# Guía de desarrollador — utilidades-python

> Cómo está escrito el código, por qué está así, y qué tocar para extenderlo.
> Cubre las cuatro versiones del archivo.

---

## Índice

1. [Filosofía del repositorio](#1-filosofía-del-repositorio)
2. [Arquitectura general](#2-arquitectura-general)
3. [SpeechToTextAuxPy](#3-speechtotextauxpy)
4. [EncrypPython](#4-encryppython)
5. [CSJE](#5-csje)
6. [cmdPbl](#6-cmdpbl)
7. [API de referencia](#7-api-de-referencia)
8. [Guía de extensión](#8-guía-de-extensión)
9. [Calidad y deuda técnica](#9-calidad-y-deuda-técnica)
10. [Convenciones](#10-convenciones)

---

## 1. Filosofía del repositorio

El código se escribió para **aprender y demostrar**, no para producción. Eso explica tres decisiones que un revisor profesional marcaría de inmediato:

1. **Demos en el nivel superior del módulo** en `EncrypPython`. Cada script imprime su resultado al ejecutarse. No hay guarda `if __name__ == "__main__":`. Es cómodo para leer un archivo de arriba a abajo y ver el resultado, incómodo para reutilizar.
2. **Sin empaquetado.** No hay `__init__.py`, `setup.py` ni `pyproject.toml`. Las carpetas no son paquetes importables. Se asume copiar y pegar.
3. **Sin abstracción.** Cada utilidad resuelve su problema en el menor número de archivos posible. `CSJE` es literalmente dos líneas.

La documentación no pretende corregir estas decisiones, sino hacerlas explícitas para que quien lea sepa qué esperar.

---

## 2. Arquitectura general

```
utilidades-python/
│
├── versions/                    código, intocado
│   ├── SpeechToTextAuxPy/       aplicación Windows de 3 módulos
│   ├── EncrypPython/            15 scripts independientes
│   ├── CSJE/                    1 módulo, 1 función
│   └── cmdPbl/                  1 workflow detenido
│
└── docs/                        documentación transversal
```

No hay dependencias entre versiones. Cada carpeta se puede extraer y funciona sola (salvo que `EncrypPython` necesita sus dos librerías).

### 2.1 Modelo de capas

| Capa | Dónde | Qué contiene |
|---|---|---|
| Presentación | `main.py`, `console_utils.py` | Entrada/salida de consola |
| Dominio | `sapi_engine.py` | Lógica de reconocimiento |
| Adaptadores | `clr` (pythonnet) | Puente a .NET |
| Infraestructura | Windows / SAPI | Motor propietario |

Es una arquitectura en capas muy delgada: tres archivos, una clase de dominio y dos módulos de apoyo. Para una utilidad de este tamaño, es proporcionada.

---

## 3. SpeechToTextAuxPy

### 3.1 Flujo de ejecución

```
main()
 ├─ sys.stdin/stdout/stderr.reconfigure("utf-8")   [solo win32]
 ├─ signal.signal(SIGINT, signal_handler)
 ├─ signal.signal(SIGTERM, signal_handler)          [si existe]
 ├─ SapiEngine(language="es-ES")
 │   ├─ import clr
 │   ├─ clr.AddReference("System.Speech")
 │   ├─ CultureInfo("es-ES")
 │   ├─ SpeechRecognitionEngine(cultura)
 │   ├─ SetInputToDefaultAudioDevice()
 │   ├─ LoadGrammar(DictationGrammar())
 │   └─ SpeechRecognized += _on_speech_recognized
 ├─ engine.start()  →  RecognizeAsync(RecognizeMode.Multiple)
 └─ while True: signal.pause()  /  time.sleep(0.5)
```

Al reconocer una frase, el CLR invoca `_on_speech_recognized`, que imprime el texto. Al recibir `Ctrl+C`, `signal_handler` llama a `engine.stop()` y sale con código `0`.

### 3.2 `main.py`

**Responsabilidad:** orquestar el ciclo de vida.

| Elemento | Detalle |
|---|---|
| `_engine` | Global de módulo. Necesaria porque el manejador de señales no recibe contexto |
| `signal_handler(num, frame)` | Detiene el motor, imprime y hace `sys.exit(0)` |
| `main()` | Configura señales, crea el motor, arranca y espera |
| Bloque `try/except` | Traduce excepciones a códigos de salida |

**Detalle a tener en cuenta:** `signal.pause()` no existe en Windows. El `try/except AttributeError` está *dentro* del `while`, así que en Windows se lanza y captura una excepción cada 500 ms. Funciona, pero es ruido innecesario. La corrección es detectar una vez:

```python
_pause = getattr(signal, "pause", None)
if _pause is None:
    while True:
        time.sleep(0.5)
else:
    while True:
        _pause()
```

No se aplicó para no modificar el código original del snapshot.

### 3.3 `sapi_engine.py`

**Responsabilidad:** encapsular `SpeechRecognitionEngine`.

| Miembro | Tipo | Descripción |
|---|---|---|
| `__init__(language="es-ES")` | constructor | Importa `clr`, crea el motor, carga gramática, registra evento |
| `_language` | atributo | Código ISO del idioma |
| `_engine` | atributo | Instancia .NET o `None` |
| `_grammar` | atributo | `DictationGrammar` |
| `_RecognizeMode` | atributo | Clase `RecognizeMode`, guardada para evitar reimportar |
| `_on_speech_recognized(sender, args)` | callback | Extrae `args.Result.Text`, imprime y libera |
| `start()` | método | `RecognizeAsync(Multiple)` |
| `stop()` | método | `RecognizeAsyncCancel()` + `Dispose()`; idempotente |

**Manejo de errores del constructor:** inspecciona el texto de la excepción buscando `"culture"`, `"language"`, `"device"` o `"audio"` para dar un mensaje específico. Es una heurística frágil —depende de los mensajes de .NET, que están localizados— pero mejora mucho la experiencia frente a un `traceback` crudo.

**Ciclo de vida de recursos:** `stop()` libera el motor y la gramática y pone ambos en `None`. Llamarlo dos veces imprime "El motor ya estaba detenido" en lugar de fallar. Es correcto para un recurso COM.

### 3.4 `console_utils.py`

**Responsabilidad:** formatear la salida.

| Función | Stream | Color |
|---|---|---|
| `print_timestamp(color, stream)` | el indicado | el indicado |
| `print_recognized(text)` | `stdout` | cian + blanco brillante |
| `print_info(message)` | `stdout` | verde |
| `print_error(message)` | `stderr` | rojo |

**Degradación:** si `colorama` no está, define clases `Fore` y `Style` con atributos vacíos. El código que las usa no necesita ramificar: escribir una cadena vacía no tiene efecto. Es un patrón simple y robusto.

**Flush explícito:** todas las funciones hacen `flush()` después de escribir. Es necesario porque `stdout` puede estar *bufferizado* cuando no es una terminal, y sin flush la transcripción aparecería a saltos.

### 3.5 Dependencias entre módulos

```
main.py ──▶ console_utils
   │
   └──────▶ sapi_engine ──▶ console_utils
                    │
                    └──────▶ clr (pythonnet) ──▶ System.Speech (.NET)
```

No hay imports circulares: `console_utils` no importa nada del proyecto.

---

## 4. EncrypPython

### 4.1 Estructura

Quince archivos planos, sin paquete, sin `__init__.py`. Cada uno sigue el mismo patrón:

```python
# 1. imports
# 2. definición de funciones
# 3. demo en el nivel superior
```

Ese patrón es la causa de que importar un script ejecute la demo.

### 4.2 Las cuatro familias

#### Codificación

| Script | Implementación | Notas |
|---|---|---|
| `_base16.py` | `texto.encode('utf-8').hex()` y `bytes.fromhex(...)` | Correcto |
| `_64.py` | `base64.b64encode` | Correcto |
| `_xxd.py` | César + `binascii.hexlify` | Combina dos pasos |

#### Hash

| Script | Implementación | Notas |
|---|---|---|
| `_md5.py` | `hashlib.md5(...).hexdigest()` | Sin `usedforsecurity=False`; falla en modo FIPS |
| `_sha256.py` | `hashlib.sha256(...).hexdigest()` | Correcto |

#### Clásico

| Script | Implementación | Algoritmo |
|---|---|---|
| `_cesar.py` | `ord`/`chr` con desplazamiento | César |
| `_xor.py` | `ord(c) ^ clave` | XOR de 1 byte |
| `_rot13.py` | `ord(c) ^ clave` | **XOR, no ROT13** |
| `_bin.py` | `format(v,'08b')[::-1]` | Inversión de bits |
| `_transposicion.py` | `texto[i::clave]` | Transposición por columnas |

#### Moderno

| Script | Primitivas | Modo |
|---|---|---|
| `_aes.py` | `Crypto.Cipher.AES` | CBC + PKCS#7 |
| `_AES-256.py` | `cryptography.hazmat` | ECDH + PBKDF2 + CBC |
| `_rsa.py` | `Crypto.PublicKey.RSA` | PKCS#1 OAEP |

#### Ofuscación

`_ofusca.py` y `_cifrado_de_flujo_bytes.py` son variantes del mismo código: permutan un alfabeto con `random.sample` y sustituyen carácter por carácter. La diferencia está en el alfabeto de origen (95 vs 100 símbolos) y en que el segundo usa `string.printable`.

### 4.3 Análisis de los scripts modernos

#### `_aes.py`

```python
clave = b'EstaEsMiClave12'      # 15 bytes
clave = clave.ljust(16, b'\0')  # 16 bytes → AES-128
```

Flujo: `iv = get_random_bytes(16)` → `pad(texto, 16)` → `AES.new(clave, CBC, iv)` → `base64(iv + ct)`.

**Correcto:** IV aleatorio por mensaje, padding estándar, formato de transporte que incluye el IV.
**Incorrecto para producción:** clave fija en el código; CBC sin MAC (sin integridad); sin KDF.

#### `_AES-256.py`

Flujo previsto: dos pares P-256 → ECDH → PBKDF2-HMAC-SHA256 (100 000 iteraciones, 32 bytes) → AES-256-CBC.

**Dos errores de construcción:**

```python
# Error 1: cada parte genera su propia salt
salt = os.urandom(16)          # dentro de obtener_secreto_compartido()
secret_key, salt = ...         # por lo tanto secret_key_1 != secret_key_2

# Error 2: descifrar deriva otra vez
derived_key = kdf.derive(secret_key)   # PBKDF2(PBKDF2(shared, salt2), salt1)
```

El primero rompe el acuerdo de claves. El segundo hace que la clave de descifrado no coincida con la de cifrado. Consecuencia: el script imprime `Secretos coinciden: False` y el descifrado falla o devuelve basura.

**Además:** padding propio sobre `str` (`chr(n) * n`) antes de `.encode()`, que se corrompe con caracteres multibyte; CBC sin MAC.

#### `_rsa.py`

```python
key = RSA.generate(2048)
cipher = PKCS1_OAEP.new(public_key)
```

**Correcto:** OAEP es el relleno recomendado para RSA; 2048 bits es el mínimo aceptable.
**Limitación:** OAEP-SHA1 con 2048 bits admite 214 bytes. No sirve para archivos.
**Detalle:** las claves se generan al importar el módulo, no dentro de una función.

### 4.4 Cómo reutilizar una función

Como importar ejecuta la demo, hay dos caminos:

**A. Copiar la función** a un archivo propio (lo más simple y lo que el código asume).

**B. Cargar el módulo evitando la demo** con `importlib` y filtrando la salida:

```python
import contextlib, importlib.util, io

spec = importlib.util.spec_from_file_location("cesar", "versions/EncrypPython/_cesar.py")
mod = importlib.util.module_from_spec(spec)
with contextlib.redirect_stdout(io.StringIO()):
    spec.loader.exec_module(mod)

print(mod.cifrado_cesar("abc", 1))
```

No se recomienda: es frágil y no evita efectos secundarios como la generación de claves de `_rsa.py`.

---

## 5. CSJE

```python
def sumar(a, b):
    return a + b
```

| Aspecto | Detalle |
|---|---|
| Líneas | 2 |
| Fin de línea | CRLF (Windows) |
| Guarda `__main__` | No |
| Type hints | No |
| Validación | Ninguna; `sumar("2", 3)` lanza `TypeError` |
| Tests | No |

**Uso como módulo:**

```python
import sys
sys.path.append("versions/CSJE")
from sumador import sumar
```

**Si se quisiera convertir en herramienta CLI**, el cambio mínimo sería:

```python
import argparse

def sumar(a, b):
    return a + b

if __name__ == "__main__":
    p = argparse.ArgumentParser()
    p.add_argument("a", type=float)
    p.add_argument("b", type=float)
    args = p.parse_args()
    print(sumar(args.a, args.b))
```

Eso excede el alcance del snapshot, que conserva el original.

---

## 6. cmdPbl

### 6.1 Contenido

Un único archivo: `.github/workflows/main.yml.stop`, 36 líneas.

### 6.2 Estructura del workflow

| Clave | Valor |
|---|---|
| `name` | Leer y borrar archivos .txt |
| `on.schedule.cron` | `*/10 * * * *` |
| `on.workflow_dispatch` | presente |
| `runs-on` | `ubuntu-latest` |
| `actions/checkout@v3` | repositorio `jmadasme/cmd`, token `secrets.GH_TOKEN` |
| `actions/setup-python@v4` | 3.10 |
| Scripts | `scriptAgrega.py`, `firma.py` |

### 6.3 Por qué no se ejecuta

1. La extensión `.stop` no la reconoce GitHub Actions: solo `.yml` y `.yaml`.
2. Los scripts referenciados no existen en el snapshot.
3. El token no está y no debe publicarse.
4. El `checkout` apunta a otro repositorio.

### 6.4 Si se quisiera reconstruir

Orden mínimo:

1. Escribir `scriptAgrega.py` (lee y borra `.txt`).
2. Escribir `firma.py` (hash y archivo `signature`).
3. Crear `requirements.txt` (aunque sea vacío).
4. Sustituir `secrets.GH_TOKEN` por un *fine-grained token* con permiso solo sobre el repo destino.
5. Renombrar a `.yml` y probar con `workflow_dispatch` antes de habilitar el cron.

---

## 7. API de referencia

### 7.1 `SapiEngine`

```python
class SapiEngine:
    def __init__(self, language: str = "es-ES") -> None: ...
    def start(self) -> None: ...
    def stop(self) -> None: ...
    def _on_speech_recognized(self, sender: Any, args: Any) -> None: ...
```

| Método | Excepciones | Efecto secundario |
|---|---|---|
| `__init__` | `ImportError`, `RuntimeError` | Carga el CLR y el motor |
| `start` | `RuntimeError` | Inicia escucha asíncrona |
| `stop` | — | Libera recursos; idempotente |

### 7.2 `console_utils`

```python
def print_timestamp(color: str = Fore.CYAN, stream=sys.stdout) -> None
def print_recognized(text: str) -> None
def print_info(message: str) -> None
def print_error(message: str) -> None
```

### 7.3 `EncrypPython`

```python
# _cesar.py
def cifrado_cesar(texto: str, desplazamiento: int) -> str

# _xor.py / _rot13.py
def cifrado_xor(texto: str, clave: int) -> str
def ofuscar_xor(texto: str, clave: int) -> str
def desofuscar_xor(texto: str, clave: int) -> str

# _base16.py  (inline)
# _64.py      (inline)

# _bin.py
def invertir_binario(valor: int) -> int
def cifrado_inversion_binaria(texto: str) -> str

# _transposicion.py
def transposicion(texto: str, clave: int) -> str

# _ofusca.py
def generar_mapeo() -> dict
def ofuscar(texto: str, mapeo: dict) -> str
def desofuscar(texto: str, mapeo: dict) -> str

# _cifrado_de_flujo_bytes.py
def crear_alfabeto_personalizado() -> dict
def ofuscar_alfabeto_personalizado(texto: str, alfabeto: dict) -> str
def desofuscar_alfabeto_personalizado(texto: str, alfabeto: dict) -> str

# _md5.py
def md5_hash(texto: str) -> str

# _sha256.py  (inline)

# _aes.py
def cifrar_aes(texto: str, clave: bytes) -> str
def descifrar_aes(texto_cifrado_base64: str, clave: bytes) -> str

# _AES-256.py
def generar_claves_ecc() -> tuple[bytes, bytes]
def obtener_secreto_compartido(private_key_pem: bytes, public_key_pem: bytes) -> tuple[bytes, bytes]
def cifrar_aes(texto: str, secret_key: bytes, salt: bytes) -> str
def descifrar_aes(texto_ofuscado: str, secret_key: bytes) -> str

# _rsa.py
def cifrar_rsa(texto: str, public_key: bytes) -> str
def descifrar_rsa(texto_cifrado: str, private_key: bytes) -> str
```

### 7.4 `CSJE`

```python
def sumar(a, b)
```

---

## 8. Guía de extensión

### 8.1 Agregar un script de cifrado

Sigue el patrón existente para no romper la coherencia:

```python
"""_vigenere.py - cifrado de Vigenère (demostración)."""

ALFABETO = "abcdefghijklmnopqrstuvwxyz"


def cifrar_vigenere(texto: str, clave: str) -> str:
    """Cifra con Vigenère; solo letras ASCII, el resto pasa igual."""
    resultado, j = [], 0
    for c in texto:
        if c.lower() in ALFABETO:
            base = ord('a') if c.islower() else ord('A')
            k = ALFABETO.index(clave[j % len(clave)].lower())
            resultado.append(chr((ord(c) - base + k) % 26 + base))
            j += 1
        else:
            resultado.append(c)
    return "".join(resultado)


if __name__ == "__main__":          # <- la mejora que falta en el resto
    print(cifrar_vigenere("hola", "clave"))
```

**Reglas del archivo nuevo:**
- Prefijo `_` y nombre del algoritmo.
- Docstring de una línea al inicio.
- Funciones puras, sin `print` internos.
- Demo bajo `if __name__ == "__main__":`.
- Sin dependencias externas si es posible.

### 8.2 Convertir el catálogo en un paquete

Si el objetivo fuera dejar de copiar y pegar:

```
versions/EncrypPython/
├── __init__.py
├── cifrado_clasico/
│   ├── __init__.py
│   ├── cesar.py
│   └── vigenere.py
├── codificacion/
├── hash/
└── moderno/
```

Y un `pyproject.toml` con `[project] name = "encryppython"`. Implica reescribir los quince scripts para quitar las demos del nivel superior.

### 8.3 Agregar un idioma a SpeechToTextAuxPy

`SapiEngine` acepta `language` en el constructor. Para exponerlo por CLI:

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--lang", default="es-ES")
args = parser.parse_args()
_engine = SapiEngine(language=args.lang)
```

Requiere que el paquete de voz del idioma esté instalado en Windows. Los códigos habituales: `es-ES`, `es-MX`, `en-US`, `pt-BR`.

### 8.4 Guardar la transcripción a archivo

`main.py` no escribe archivos. La forma menos invasiva es un *wrapper*:

```python
import io, sys
from contextlib import redirect_stdout

with open("salida.txt", "w", encoding="utf-8") as f, redirect_stdout(f):
    main()
```

O redirigir en el shell, que es lo documentado en el README.

### 8.5 Reemplazar SAPI por Whisper

SAPI es propietario, Windows-only y limitado. Un reemplazo moderno:

```python
import whisper
modelo = whisper.load_model("small")
resultado = modelo.transcribe("audio.wav", language="es")
print(resultado["text"])
```

Ventajas: multiplataforma, mejor precisión, puntuación. Desventajas: requiere GPU para tiempo real, y el modelo pesa cientos de MB. La arquitectura de `SpeechToTextAuxPy` (motor + presentación) permitiría sustituir `SapiEngine` por un `WhisperEngine` manteniendo `main.py` casi igual. Esa es, quizá, la extensión más valiosa del repositorio.

---

## 9. Calidad y deuda técnica

### 9.1 Lo que está bien

| Aspecto | Dónde |
|---|---|
| Separación en módulos | SpeechToTextAuxPy |
| Docstrings completos | SpeechToTextAuxPy |
| Degradación elegante | `console_utils` sin `colorama` |
| Códigos de salida diferenciados | `main.py` |
| Liberación de recursos | `SapiEngine.stop()` |
| Mensajes de error accionables | `sapi_engine` |

### 9.2 Lo que falta

| Deuda | Impacto | Prioridad |
|---|---|---|
| Guarda `__main__` | Importar ejecuta demos | Alta |
| Tests | Sin verificación | Alta |
| Claves fuera del código | Cifrado decorativo | Alta |
| AEAD en los ejemplos AES | Sin integridad | Alta |
| Corrección de `_AES-256.py` | No descifra | Media |
| `requirements.txt` para cripto | `ImportError` | Media |
| César con acentos | Resultado incorrecto | Media |
| `signal.pause()` en bucle | Ruido de excepciones | Baja |
| Nombres de archivo (`_64.py`) | Poco descriptivos | Baja |

### 9.3 Cómo verificar sin tests

```bash
# Sintaxis de todos los scripts
python3 -m compileall -q versions/

# Vectores conocidos
printf 'hola' | sha256sum
printf 'hola' | base64
printf '1234' | md5sum

# Comparar con los scripts
python3 versions/EncrypPython/_sha256.py
```

`compileall` detecta errores de sintaxis, no de lógica. Para lógica hacen falta vectores de prueba, que es exactamente lo que falta.

---

## 10. Convenciones

### 10.1 Estilo observado

| Convención | Uso |
|---|---|
| Nombres de función | `snake_case` en español (`cifrado_cesar`, `ofuscar_xor`) |
| Nombres de clase | `PascalCase` (`SapiEngine`) |
| Constantes | MAYÚSCULAS (`ALFABETO`, `COLORAMA_DISPONIBLE`) |
| Prefijo de archivos cripto | `_` |
| Docstrings | Español, con Args/Raises en S2T |
| Comentarios | Español, explican el porqué |

La mezcla de inglés (`SapiEngine`, `start`, `stop`) y español (`cifrado_cesar`) es deliberada en S2T —la API imita .NET— y accidental en EncrypPython.

### 10.2 Reglas para contribuir al snapshot

1. **No modificar `versions/`** salvo que se esté corrigiendo un bug documentado en `BITACORA.md`.
2. **Documentar primero.** Si un cambio altera el comportamiento, actualizar README y datasheet.
3. **No agregar secretos.** Ni tokens, ni claves, ni rutas personales.
4. **Mantener el estilo.** Un script nuevo debe parecerse a los quince existentes.
5. **No romper la compatibilidad de Python 3.10**, que es el mínimo declarado.

---

## Apéndice A — Mapa de archivos

| Archivo | Líneas | Rol |
|---|---|---|
| `SpeechToTextAuxPy/src/main.py` | 135 | Entrada |
| `SpeechToTextAuxPy/src/sapi_engine.py` | 201 | Motor |
| `SpeechToTextAuxPy/src/console_utils.py` | 108 | Salida |
| `EncrypPython/_AES-256.py` | 117 | Cripto moderna |
| `EncrypPython/_aes.py` | 63 | Cripto moderna |
| `EncrypPython/_xxd.py` | 41 | Codificación + clásico |
| `EncrypPython/_cesar.py` | 34 | Clásico |
| `EncrypPython/_rsa.py` | 32 | Cripto moderna |
| `EncrypPython/_ofusca.py` | 28 | Ofuscación |
| `EncrypPython/_bin.py` | 27 | Clásico |
| `EncrypPython/_cifrado_de_flujo_bytes.py` | 27 | Ofuscación |
| `EncrypPython/_rot13.py` | 17 | Clásico (XOR) |
| `EncrypPython/_64.py` | 14 | Codificación |
| `EncrypPython/_transposicion.py` | 12 | Clásico |
| `EncrypPython/_xor.py` | 8 | Clásico |
| `EncrypPython/_base16.py` | 8 | Codificación |
| `EncrypPython/_md5.py` | 8 | Hash |
| `EncrypPython/_sha256.py` | 6 | Hash |
| `CSJE/sumador.py` | 2 | Suma |
| `cmdPbl/.github/workflows/main.yml.stop` | 36 | Workflow detenido |

**Total Python:** ≈ 888 líneas.

---

*Documentación para desarrolladores · portfolio `AG.CreadorContenido` · 2026.*
