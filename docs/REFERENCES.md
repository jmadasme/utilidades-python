# Referencias — utilidades-python

> Fuentes técnicas usadas para escribir y verificar la documentación del archivo.
> Se priorizan estándares, documentación oficial y especificaciones, no blogs.

---

## Índice

1. [Cómo leer esta lista](#1-cómo-leer-esta-lista)
2. [SpeechToTextAuxPy](#2-speechtotextauxpy)
3. [Criptografía: estándares](#3-criptografía-estándares)
4. [Bibliotecas de criptografía](#4-bibliotecas-de-criptografía)
5. [Biblioteca estándar de Python](#5-biblioteca-estándar-de-python)
6. [Herramientas de sistema para vectores](#6-herramientas-de-sistema-para-vectores)
7. [GitHub Actions](#7-github-actions)
8. [Tabla de trazabilidad](#8-tabla-de-trazabilidad)

---

## 1. Cómo leer esta lista

Cada entrada indica **qué se usó para qué**. La columna *Uso en el proyecto* conecta la fuente con el archivo o la afirmación que respalda. Si una afirmación del README no tiene fuente aquí, es observación empírica del código, no una cita.

---

## 2. SpeechToTextAuxPy

### 2.1 Microsoft Learn — System.Speech

| Recurso | URL | Uso en el proyecto |
|---|---|---|
| `SpeechRecognitionEngine` (clase) | https://learn.microsoft.com/en-us/dotnet/api/system.speech.recognition.speechrecognitionengine | Documentar `SetInputToDefaultAudioDevice`, `LoadGrammar`, `RecognizeAsync`, `RecognizeAsyncCancel`, `Dispose` |
| `DictationGrammar` (clase) | https://learn.microsoft.com/en-us/dotnet/api/system.speech.recognition.dictationgrammar | Explicar la gramática de texto libre |
| `RecognizeMode` (enumeración) | https://learn.microsoft.com/en-us/dotnet/api/system.speech.recognition.recognizemode | Justificar `RecognizeMode.Multiple` para escucha continua |
| `SpeechRecognizedEventArgs` | https://learn.microsoft.com/en-us/dotnet/api/system.speech.recognition.speechrecognizedeventargs | Explicar `args.Result.Text` |
| `CultureInfo` (clase) | https://learn.microsoft.com/en-us/dotnet/api/system.globalization.cultureinfo | Justificar el código `es-ES` |
| .NET Framework 4.7.2 (descarga) | https://dotnet.microsoft.com/download/dotnet-framework/net472 | Requisito de versión mínima |

### 2.2 SAPI y reconocimiento de voz

| Recurso | URL | Uso |
|---|---|---|
| Windows Speech Recognition (soporte) | https://support.microsoft.com/windows | Instalación del paquete de voz desde Configuración |
| Descripción general de SAPI | https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ms723627(v=vs.85) | Contexto histórico de SAPI 5 |

> **Nota.** `System.Speech` envuelve los reconocedores SAPI 5 que instala Windows. No existe como componente de .NET (Core) 8+ ni funciona fuera de Windows, lo que explica la limitación de plataforma de la utilidad.

### 2.3 pythonnet

| Recurso | URL | Uso |
|---|---|---|
| Sitio oficial | https://pythonnet.github.io/ | Arquitectura del puente Python ↔ .NET |
| Repositorio | https://github.com/pythonnet/pythonnet | `clr.AddReference`, compatibilidad por versión |
| PyPI | https://pypi.org/project/pythonnet/ | Versión mínima `3.0.0` |
| Guía de interoperabilidad | https://pythonnet.github.io/pythonnet/python.html | Manejo de tipos .NET desde Python |

### 2.4 colorama

| Recurso | URL | Uso |
|---|---|---|
| Repositorio | https://github.com/tartley/colorama | `init()`, `Fore`, `Style`, comportamiento de *strip* cuando no hay TTY |
| PyPI | https://pypi.org/project/colorama/ | Versión mínima `0.4.6` |

**Punto verificado:** `colorama.init()` con `strip=None` elimina las secuencias ANSI cuando el stream no es una terminal. Eso hace que la redirección a archivo produzca texto limpio, afirmación usada en README §4.5.

### 2.5 Python — señales y consola

| Recurso | URL | Uso |
|---|---|---|
| `signal` — documentación | https://docs.python.org/3/library/signal.html | `SIGINT`, `SIGTERM`, ausencia de `pause()` en Windows |
| `sys` — `reconfigure` | https://docs.python.org/3/library/sys.html#sys.stdin | Cambio de codificación a UTF-8 |
| `time.sleep` | https://docs.python.org/3/library/time.html#time.sleep | Espera en el bucle principal |

---

## 3. Criptografía: estándares

### 3.1 Simétricos

| Estándar | Título | URL | Scripts |
|---|---|---|---|
| **FIPS 197** | Advanced Encryption Standard (AES) | https://csrc.nist.gov/pubs/fips/197/final | `_aes.py`, `_AES-256.py` |
| **NIST SP 800-38A** | Recommendation for Block Cipher Modes of Operation | https://csrc.nist.gov/pubs/sp/800/38/a/final | CBC en `_aes.py`, `_AES-256.py` |
| **NIST SP 800-38D** | Galois/Counter Mode (GCM) | https://csrc.nist.gov/pubs/sp/800/38/d/final | Alternativa recomendada (AEAD) |
| **RFC 5116** | An Interface and Algorithms for Authenticated Encryption | https://www.rfc-editor.org/rfc/rfc5116 | Justificación de AEAD frente a CBC |

**Dato usado en la documentación:** AES bloquea en 128 bits; la clave puede ser de 128, 192 o 256 bits. `_aes.py` usa 16 bytes (AES-128) y `_AES-256.py` deriva 32 (AES-256).

### 3.2 Asimétricos

| Estándar | Título | URL | Scripts |
|---|---|---|---|
| **RFC 8017** | PKCS #1 v2.2: RSA Cryptography Specifications | https://www.rfc-editor.org/rfc/rfc8017 | `_rsa.py` (OAEP) |
| **NIST SP 800-56A** | Pair-Wise Key-Establishment Schemes (ECDH) | https://csrc.nist.gov/pubs/sp/800/56/a/r3/final | `_AES-256.py` |
| **SEC 2** | Recommended Elliptic Curve Domain Parameters (P-256) | https://www.secg.org/sec2-v2.pdf | `ec.SECP256R1()` |

**Dato usado:** con RSA-2048 y OAEP-SHA1, el mensaje máximo es de 214 bytes (`256 − 2·20 − 2`). Ver README §5.4.

### 3.3 Hash y derivación de claves

| Estándar | Título | URL | Scripts |
|---|---|---|---|
| **FIPS 180-4** | Secure Hash Standard (SHA-1, SHA-2) | https://csrc.nist.gov/pubs/fips/180-4/upd1/final | `_sha256.py` |
| **RFC 1321** | The MD5 Message-Digest Algorithm | https://www.rfc-editor.org/rfc/rfc1321 | `_md5.py` |
| **RFC 2104** | HMAC: Keyed-Hashing for Message Authentication | https://www.rfc-editor.org/rfc/rfc2104 | Base de PBKDF2 |
| **RFC 2898 / PKCS #5** | Password-Based Cryptography Specification (PBKDF2) | https://www.rfc-editor.org/rfc/rfc2898 | `_AES-256.py` |
| **RFC 7914** | The scrypt Password-Based KDF | https://www.rfc-editor.org/rfc/rfc7914 | Alternativa recomendada en README §9.3 |
| **RFC 9106** | Argon2 Memory-Hard Function | https://www.rfc-editor.org/rfc/rfc9106 | Alternativa recomendada |

**Dato usado:** MD5 está roto para seguridad desde 2004 (colisiones prácticas). SHA-256 sigue vigente para integridad. Ninguno de los dos sirve para almacenar contraseñas.

### 3.4 Codificación

| Estándar | Título | URL | Scripts |
|---|---|---|---|
| **RFC 4648** | The Base16, Base32, and Base64 Data Encodings | https://www.rfc-editor.org/rfc/rfc4648 | `_base16.py`, `_64.py` |

**Dato usado:** Base16 es hexadecimal; Base64 usa 64 símbolos y añade relleno `=`. Ninguno cifra.

### 3.5 Cifrados clásicos

| Recurso | URL | Scripts |
|---|---|---|
| Cifrado César — referencia histórica | https://en.wikipedia.org/wiki/Caesar_cipher | `_cesar.py`, `_xxd.py` |
| Cifrado por transposición | https://en.wikipedia.org/wiki/Transposition_cipher | `_transposicion.py` |
| ROT13 | https://en.wikipedia.org/wiki/ROT13 | `_rot13.py` (nombre, no contenido) |
| Cifrado XOR | https://en.wikipedia.org/wiki/XOR_cipher | `_xor.py`, `_rot13.py` |

> Estas fuentes son enciclopédicas, no normativas. Se usan solo para terminología e historia; los cifrados clásicos no tienen estándar vigente.

---

## 4. Bibliotecas de criptografía

### 4.1 PyCryptodome

| Recurso | URL | Uso |
|---|---|---|
| Documentación | https://www.pycryptodome.org/ | API de `AES`, `RSA`, `PKCS1_OAEP`, `Padding` |
| Módulo AES | https://www.pycryptodome.org/src/cipher/aes | `AES.new(clave, AES.MODE_CBC, iv)` |
| Módulo RSA | https://www.pycryptodome.org/src/public_key/rsa | `RSA.generate(2048)`, `export_key`, `import_key` |
| PKCS#1 OAEP | https://www.pycryptodome.org/src/cipher/pkcs1_oaep | `PKCS1_OAEP.new(clave)` |
| Utilidades de padding | https://www.pycryptodome.org/src/util/padding | `pad`, `unpad` |
| PyPI | https://pypi.org/project/pycryptodome/ | Instalación |

**Advertencia documentada:** `pycryptodome` expone el paquete `Crypto`. El proyecto `pycrypto`, con el mismo nombre de paquete, está abandonado desde 2013 y tiene vulnerabilidades sin parchear. No confundirlos.

### 4.2 cryptography

| Recurso | URL | Uso |
|---|---|---|
| Documentación | https://cryptography.io/ | API `hazmat` |
| Cifrados simétricos | https://cryptography.io/en/latest/hazmat/primitives/symmetric-encryption/ | `Cipher`, `algorithms.AES`, `modes.CBC` |
| AEAD (AES-GCM) | https://cryptography.io/en/latest/hazmat/primitives/aead/ | Ejemplo correcto del README §9.3 |
| Derivación de claves | https://cryptography.io/en/latest/hazmat/primitives/key-derivation-functions/ | `PBKDF2HMAC`, `Scrypt`, `HKDF` |
| Intercambio de claves | https://cryptography.io/en/latest/hazmat/primitives/asymmetric/ec/ | `ec.generate_private_key`, `exchange(ec.ECDH(), ...)` |
| Serialización | https://cryptography.io/en/latest/hazmat/primitives/asymmetric/serialization/ | `load_pem_private_key`, `load_pem_public_key` |
| PyPI | https://pypi.org/project/cryptography/ | Instalación |

---

## 5. Biblioteca estándar de Python

| Módulo | URL | Uso en el proyecto |
|---|---|---|
| `hashlib` | https://docs.python.org/3/library/hashlib.html | `md5`, `sha256`, `hexdigest`; nota sobre FIPS |
| `base64` | https://docs.python.org/3/library/base64.html | `b64encode`, `b64decode` |
| `binascii` | https://docs.python.org/3/library/binascii.html | `hexlify` |
| `random` | https://docs.python.org/3/library/random.html | `random.sample`; advertencia de que no es criptográfico |
| `string` | https://docs.python.org/3/library/string.html | `ascii_letters`, `digits`, `punctuation`, `printable` |
| `os` | https://docs.python.org/3/library/os.html | `os.urandom` |
| `signal` | https://docs.python.org/3/library/signal.html | `SIGINT`, `SIGTERM`, `pause` |
| `sys` | https://docs.python.org/3/library/sys.html | `platform`, `reconfigure`, `exit` |
| `datetime` | https://docs.python.org/3/library/datetime.html | `strftime("%H:%M:%S")` |
| `importlib.util` | https://docs.python.org/3/library/importlib.html | Carga de módulos sin ejecutar la demo (README §4.4 de developer) |
| `contextlib.redirect_stdout` | https://docs.python.org/3/library/contextlib.html | Silenciar la demo al importar |
| `argparse` | https://docs.python.org/3/library/argparse.html | Extensión CLI propuesta |

**Punto verificado:** `random` usa el generador Mersenne Twister, determinista y no apto para criptografía. La documentación oficial lo dice explícitamente; por eso `_ofusca.py` y `_cifrado_de_flujo_bytes.py` se clasifican como ofuscación, no cifrado.

---

## 6. Herramientas de sistema para vectores

| Herramienta | Comando | Uso |
|---|---|---|
| `sha256sum` | `printf 'hola' \| sha256sum` | Vector de referencia para `_sha256.py` |
| `sha256sum` | `sha256sum archivo.iso` | Verificación de integridad (README §9.2) |
| `md5sum` | `printf '1234' \| md5sum` | Vector para `_md5.py` |
| `base64` | `printf 'hola' \| base64` | Vector para `_64.py` → `aG9sYQ==` |
| `xxd` | `printf 'hola' \| xxd -p` | Vector para `_base16.py` → `686f6c61` |
| `openssl` | `openssl dgst -sha256` | Alternativa de verificación |

**Por qué importa:** los scripts no tienen tests. La única forma de comprobar que un algoritmo está bien implementado es comparar su salida con una herramienta independiente. Ese contraste es el que falta en el repositorio.

---

## 7. GitHub Actions

| Recurso | URL | Uso |
|---|---|---|
| Eventos de workflow (`schedule`) | https://docs.github.com/en/actions/reference/events-that-trigger-workflows#schedule | Interpretar `*/10 * * * *` |
| Sintaxis de workflows | https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax | Estructura de `main.yml.stop` |
| Secretos cifrados | https://docs.github.com/en/actions/security-guides/encrypted-secrets | `secrets.GH_TOKEN` en `cmdPbl` |
| `actions/checkout` | https://github.com/actions/checkout | Checkout de otro repositorio |
| `actions/setup-python` | https://github.com/actions/setup-python | Versión 3.10 del workflow |
| Tokens de acceso con permisos finos | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | Recomendación de reemplazo del token |
| Cron sintaxis | https://docs.github.com/en/actions/reference/events-that-trigger-workflows#schedule | `*/10` = cada 10 minutos |

**Cálculo usado en la documentación:** `*/10 * * * *` son 6 ejecuciones por hora × 24 × 30 = 4 320 al mes.

**Dato verificado:** GitHub Actions solo procesa archivos `.yml` y `.yaml` dentro de `.github/workflows/`. Un `.stop` se ignora, que es exactamente el mecanismo de desactivación usado en `cmdPbl`.

---

## 8. Tabla de trazabilidad

| Afirmación del README | Fuente |
|---|---|
| SAPI es Windows-only y no está en .NET 8 | Microsoft Learn, `System.Speech` |
| `pythonnet` ≥ 3.0 es necesario | PyPI / repositorio pythonnet |
| `colorama` elimina ANSI sin TTY | Repositorio colorama, `init(strip=None)` |
| AES bloquea a 128 bits | FIPS 197 |
| CBC no aporta integridad | NIST SP 800-38A / RFC 5116 |
| OAEP admite 214 bytes con RSA-2048 | RFC 8017, cálculo de longitud |
| MD5 está roto para seguridad | RFC 1321 + literatura de colisiones (2004) |
| SHA-256 sigue vigente para integridad | FIPS 180-4 |
| Base64 no es cifrado | RFC 4648 |
| `random` no es criptográfico | Documentación de `random` |
| P-256 es una curva recomendada | SEC 2 |
| PBKDF2 con 100 000 iteraciones | RFC 2898 |
| GCM es la alternativa autenticada | NIST SP 800-38D |
| Cron `*/10` = 4 320/mes | GitHub Actions, sintaxis cron |
| `.stop` desactiva el workflow | GitHub Actions, sintaxis de workflows |

---

## 9. Fuentes descartadas

| Fuente | Motivo |
|---|---|
| Blogs sobre "cifrado en Python" | Sin estándar; repiten errores de clave fija |
| Tutoriales de César con `isalpha()` | Propagan el bug de acentos descrito en BITACORA §2.1 |
| Ejemplos de AES-CBC sin MAC | Enseñan una construcción insegura |
| `pycrypto` | Abandonado, vulnerabilidades sin parchear |
| Respuestas de foro sin fecha | No verificables |

La regla fue: si una afirmación no se puede respaldar con un estándar, la documentación oficial o una reproducción propia, no se escribe.

---

*Referencias del archivo `utilidades-python` · portfolio `AG.CreadorContenido` · 2026.*
