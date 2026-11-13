# Datasheet — utilidades-python

> Ficha técnica y guía de usuario del archivo de utilidades.
> Cubre las cuatro versiones: `SpeechToTextAuxPy`, `EncrypPython`, `CSJE` y `cmdPbl`.

---

## Identificación

| Campo | Valor |
|---|---|
| **Producto** | utilidades-python |
| **Tipo** | Archivo de versiones (monorepo de documentación) |
| **Versiones** | 4 |
| **Scripts Python** | 19 |
| **Líneas de código Python** | ≈ 888 |
| **Plataforma** | Windows 10 (una versión) · multiplataforma (tres) |
| **Lenguaje** | Python 3 |
| **Licencia** | MIT |
| **Autor** | jmadasme |
| **Año** | 2026 |
| **Estado** | Archivo. Sin mantenimiento activo |

---

## 1. Resumen ejecutivo

`utilidades-python` reúne cuatro proyectos de madurez desigual bajo un mismo techo documental. No comparten código ni interfaz: comparten autor y propósito didáctico.

| Versión | Función | Madurez | Plataforma | Dependencias |
|---|---|---|---|---|
| **SpeechToTextAuxPy** | Voz a texto continuo en español | Usable | Windows 10 | `pythonnet`, `colorama` |
| **EncrypPython** | 15 demos de criptografía y codificación | Didáctica | Cualquiera | 2 opcionales |
| **CSJE** | Suma de dos valores | Mínima | Cualquiera | Ninguna |
| **cmdPbl** | Workflow de Actions detenido | Stub | GitHub Actions | Ninguna |

**Recomendación de uso:** solo `SpeechToTextAuxPy` está pensada para que alguien la ejecute y obtenga un resultado. El resto es material de lectura.

---

## 2. Especificaciones técnicas

### 2.1 Requisitos de hardware

| Componente | SpeechToTextAuxPy | Resto |
|---|---|---|
| CPU | 1 GHz o superior | Cualquiera con Python |
| RAM | 512 MB libres | < 50 MB |
| Disco | ~10 MB con dependencias | < 1 MB |
| Audio | Micrófono o *line-in* | No aplica |
| Red | No requiere | No requiere (salvo `pip`) |

### 2.2 Requisitos de software

| Componente | Versión mínima | Notas |
|---|---|---|
| Python | 3.10 | 3.8 alcanza para el resto |
| .NET Framework | 4.7.2 | Incluido en Windows 10; solo S2T |
| SAPI / System.Speech | Componente de Windows | Solo S2T |
| Paquete de voz | `es-ES` | Descarga aparte |
| pip | Cualquiera reciente | Para dependencias |

### 2.3 Sistemas operativos soportados

| SO | SpeechToTextAuxPy | EncrypPython | CSJE | cmdPbl |
|---|---|---|---|---|
| Windows 10/11 | Sí | Sí | Sí | N/A |
| Linux | No | Sí | Sí | N/A |
| macOS | No | Sí | Sí | N/A |
| WSL | No (sin audio SAPI) | Sí | Sí | N/A |

---

## 3. Dependencias

### 3.1 Declaradas en `requirements.txt`

El archivo `versions/SpeechToTextAuxPy/requirements.txt` contiene **dos** dependencias:

| Paquete | Versión declarada | Propósito | Obligatoria |
|---|---|---|---|
| `pythonnet` | `>=3.0.0` | Puente Python ↔ .NET (CLR) para invocar `System.Speech` | Sí |
| `colorama` | `>=0.4.6` | Secuencias ANSI en consolas Windows | No (hay *fallback*) |

Contenido exacto:

```
pythonnet>=3.0.0
colorama>=0.4.6
```

### 3.2 Instalación dentro del entorno virtual

```bash
cd versions/SpeechToTextAuxPy

python3 -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows

pip install -r requirements.txt
```

En Windows, que es el único destino real de la utilidad, el flujo completo es:

```bat
cd versions\SpeechToTextAuxPy
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
python src\main.py
```

### 3.3 Dependencias no declaradas

Tres scripts de `EncrypPython` requieren librerías que **no** están en ningún `requirements.txt`:

| Script | Paquete | Versión sugerida | Instalación |
|---|---|---|---|
| `_aes.py` | `pycryptodome` | ≥ 3.20 | `pip install pycryptodome` |
| `_rsa.py` | `pycryptodome` | ≥ 3.20 | `pip install pycryptodome` |
| `_AES-256.py` | `cryptography` | ≥ 42 | `pip install cryptography` |

Los otros doce scripts usan solo biblioteca estándar: `hashlib`, `base64`, `binascii`, `random`, `string`, `os`.

> `pycryptodome` expone el paquete `Crypto`. No instalar `pycrypto`, abandonado y con vulnerabilidades conocidas.

### 3.4 Matriz de dependencias por script

| Script | `hashlib` | `base64` | `binascii` | `random` | `Crypto` | `cryptography` |
|---|---|---|---|---|---|---|
| `_cesar.py` | — | — | — | — | — | — |
| `_rot13.py` | — | — | — | — | — | — |
| `_xor.py` | — | — | — | — | — | — |
| `_base16.py` | — | — | — | — | — | — |
| `_64.py` | — | ✔ | — | — | — | — |
| `_xxd.py` | — | — | ✔ | — | — | — |
| `_bin.py` | — | — | — | — | — | — |
| `_transposicion.py` | — | — | — | — | — | — |
| `_ofusca.py` | — | — | — | ✔ | — | — |
| `_cifrado_de_flujo_bytes.py` | — | — | — | ✔ | — | — |
| `_md5.py` | ✔ | — | — | — | — | — |
| `_sha256.py` | ✔ | — | — | — | — | — |
| `_aes.py` | — | ✔ | — | — | ✔ | — |
| `_AES-256.py` | — | ✔ | — | — | — | ✔ |
| `_rsa.py` | — | ✔ | — | — | ✔ | — |

---

## 4. Modo de operación

### 4.1 SpeechToTextAuxPy

| Paso | Acción |
|---|---|
| 1 | Instalar el paquete de voz `es-ES` en Windows |
| 2 | Seleccionar el dispositivo de entrada (micrófono o *line-in*) como predeterminado |
| 3 | Crear el entorno virtual e instalar `requirements.txt` |
| 4 | Ejecutar `run.bat` o `python src/main.py` |
| 5 | Hablar; cada frase aparece con timestamp |
| 6 | `Ctrl+C` para salir |

**Salida:** texto por `stdout` con formato `[HH:MM:SS] frase`. Errores por `stderr` con prefijo `ERROR:`.

**Códigos de salida:** `0` normal, `1` dependencia faltante, `2` fallo SAPI (idioma/audio), `3` error inesperado.

**Persistencia:** ninguna. Para guardar, redirigir `stdout`.

### 4.2 EncrypPython

| Paso | Acción |
|---|---|
| 1 | Instalar la dependencia del script si la requiere |
| 2 | `cd versions/EncrypPython` |
| 3 | `python3 _nombre.py` |
| 4 | Leer el resultado en consola |

**Entrada:** el texto está escrito dentro de cada archivo. Para probar otro texto hay que editarlo.

**Salida:** una o más líneas de resultado.

### 4.3 CSJE

| Paso | Acción |
|---|---|
| 1 | Agregar `versions/CSJE` al `sys.path` |
| 2 | `from sumador import sumar` |
| 3 | `sumar(a, b)` |

### 4.4 cmdPbl

No se opera. El workflow está desactivado por la extensión `.stop` y le faltan los scripts que invoca.

---

## 5. Inventario funcional

### 5.1 SpeechToTextAuxPy

| Módulo | Función pública | Descripción |
|---|---|---|
| `main.py` | `main()` | Punto de entrada; señales y bucle |
| `main.py` | `signal_handler(num, frame)` | Detención limpia |
| `sapi_engine.py` | `SapiEngine(language="es-ES")` | Envoltura del motor SAPI |
| `sapi_engine.py` | `.start()` / `.stop()` | Inicio y fin del reconocimiento |
| `console_utils.py` | `print_recognized(text)` | Texto con timestamp |
| `console_utils.py` | `print_info(msg)` / `print_error(msg)` | Mensajes informativos y de error |

### 5.2 EncrypPython

| Script | Función | Firma |
|---|---|---|
| `_cesar.py` | `cifrado_cesar` | `(texto, desplazamiento) -> str` |
| `_rot13.py` | `ofuscar_xor`, `desofuscar_xor` | `(texto, clave) -> str` |
| `_xor.py` | `cifrado_xor` | `(texto, clave) -> str` |
| `_base16.py` | — (inline) | — |
| `_64.py` | — (inline) | — |
| `_xxd.py` | `cifrado_cesar` | `(texto, desplazamiento) -> str` |
| `_bin.py` | `invertir_binario`, `cifrado_inversion_binaria` | `(valor) -> int`, `(texto) -> str` |
| `_transposicion.py` | `transposicion` | `(texto, clave) -> str` |
| `_ofusca.py` | `generar_mapeo`, `ofuscar`, `desofuscar` | `() -> dict`, `(texto, mapeo) -> str` |
| `_cifrado_de_flujo_bytes.py` | `crear_alfabeto_personalizado`, `ofuscar_...`, `desofuscar_...` | idem |
| `_md5.py` | `md5_hash` | `(texto) -> str` |
| `_sha256.py` | — (inline) | — |
| `_aes.py` | `cifrar_aes`, `descifrar_aes` | `(texto, clave) -> str`, `(b64, clave) -> str` |
| `_AES-256.py` | `generar_claves_ecc`, `obtener_secreto_compartido`, `cifrar_aes`, `descifrar_aes` | ver developer.md |
| `_rsa.py` | `cifrar_rsa`, `descifrar_rsa` | `(texto, pem) -> str` |

### 5.3 CSJE

| Función | Firma | Retorno |
|---|---|---|
| `sumar` | `(a, b)` | `a + b` |

---

## 6. Rendimiento

| Utilidad | Métrica | Valor observado |
|---|---|---|
| SpeechToTextAuxPy | Latencia de transcripción | ~1–3 s por frase (depende del modelo) |
| SpeechToTextAuxPy | RAM | ~50–150 MB con CLR cargado |
| SpeechToTextAuxPy | CPU en reposo | < 2 % |
| `_sha256.py` | Hash de string corto | < 1 ms |
| `_rsa.py` | Generación de par 2048 | 50–500 ms |
| `_AES-256.py` | PBKDF2 100 000 iteraciones | 30–100 ms por derivación |

No hay benchmarks formales. Los valores son órdenes de magnitud, no mediciones reproducibles.

---

## 7. Compatibilidad y limitaciones

### 7.1 Limitaciones por diseño

| Limitación | Afecta a | Consecuencia |
|---|---|---|
| Sin CLI ni argumentos | EncrypPython | Datos *hardcodeados* |
| Sin guarda `__main__` | 15 scripts | Importar ejecuta la demo |
| Sin tests | Todo | No hay verificación automática |
| Sin empaquetado | Todo | No se puede `pip install` |
| Sin configuración externa | Todo | Cambiar requiere editar código |
| Windows-only | SpeechToTextAuxPy | No corre en Linux/macOS |
| Dependencia de .NET Framework | SpeechToTextAuxPy | No funciona en .NET 8 |

### 7.2 Compatibilidad de versiones de Python

| Versión | Estado |
|---|---|
| 3.8 | Funciona el resto; S2T no probado |
| 3.9 | Ídem |
| 3.10 | Recomendada |
| 3.11 / 3.12 | Probable; `pythonnet` requiere ≥ 3.0 |
| 3.13+ | `pythonnet` puede requerir compilación |

---

## 8. Guía de solución de problemas

### 8.1 SpeechToTextAuxPy

| Síntoma | Causa | Solución |
|---|---|---|
| `pythonnet no está instalado` | Falta dependencia | `pip install -r requirements.txt` |
| `El idioma 'es-ES' no está soportado` | Paquete de voz ausente | Configuración → Hora e idioma → Voz |
| `No se encontró dispositivo de entrada` | Micrófono desconectado | Panel de sonido → Grabación |
| No reconoce nada | Nivel bajo o dispositivo erróneo | Subir ganancia; verificar predeterminado |
| `System.Speech` no se encuentra | .NET < 4.7.2 o SO no Windows | Actualizar; no hay soporte Linux |
| Acentos corruptos | Página de códigos | `chcp 65001` |
| No termina con `Ctrl+C` | Señal interceptada | `taskkill /PID <pid>` |

### 8.2 EncrypPython

| Síntoma | Causa | Solución |
|---|---|---|
| `ModuleNotFoundError: Crypto` | Falta `pycryptodome` | `pip install pycryptodome` |
| `ModuleNotFoundError: cryptography` | Falta `cryptography` | `pip install cryptography` |
| `ValueError: Data must be padded` | Clave/IV incorrectos | Verificar longitud (16/24/32) |
| `UnicodeDecodeError` | Clave distinta | Revisar derivación |
| `Secretos coinciden: False` | Bug de `_AES-256.py` | Ver BITACORA.md |
| `TypeError` con emoji en `_bin.py` | `format(..., '08b')` solo 0–255 | Usar ASCII |
| Acentos mal cifrados | `_cesar.py` opera sobre Unicode | Usar solo ASCII |

### 8.3 Entorno

| Síntoma | Causa | Solución |
|---|---|---|
| `pip` instala global | *venv* inactivo | Activar el entorno |
| `python` no encontrado | PATH | Usar `python3` o reinstalar |
| Permisos en Linux | `pip` sin `--user` | Usar *venv* |

---

## 9. Advertencias

1. **No uses `EncrypPython` para proteger datos reales.** Las claves están en el código y varios algoritmos son obsoletos.
2. **`_AES-256.py` no descifra correctamente** por un doble PBKDF2 y una *salt* no compartida.
3. **`cmdPbl` no se debe reactivar** sin escribir antes los scripts que invoca y rotar el token.
4. **SpeechToTextAuxPy envía audio al reconocedor de Windows.** No hay garantía de privacidad local: el reconocimiento puede usar servicios en línea según la configuración del sistema.
5. **El repositorio no incluye secretos**, pero tampoco incluye un `.gitignore` global: revisar antes de agregar archivos.

---

## 10. Glosario

| Término | Definición |
|---|---|
| **SAPI** | Speech Application Programming Interface, API de voz de Windows |
| **System.Speech** | Espacio de nombres de .NET Framework para síntesis y reconocimiento |
| **pythonnet** | Biblioteca que carga el CLR de .NET en un proceso Python |
| **CLR** | Common Language Runtime, máquina virtual de .NET |
| **DictationGrammar** | Gramática de texto libre de SAPI |
| **AEAD** | Cifrado autenticado con datos asociados (ej. AES-GCM) |
| **OAEP** | Optimal Asymmetric Encryption Padding, relleno para RSA |
| **PBKDF2** | Derivación de clave basada en contraseña con iteraciones |
| **ECDH** | Intercambio de claves Diffie-Hellman sobre curvas elípticas |
| **IV / nonce** | Valor de inicialización; debe ser único por mensaje |
| **Salt** | Valor aleatorio que impide tablas precomputadas |
| **Stub** | Implementación incompleta conservada como registro |

---

## 11. Referencias rápidas

| Recurso | Ubicación |
|---|---|
| Documentación de usuario | `README.md` |
| Guía de desarrollador | `docs/developer.md` |
| Fuentes y estándares | `docs/REFERENCES.md` |
| Decisiones y bugs | `docs/BITACORA.md` |
| Diagramas | `docs/assets/` |
| Licencia | `LICENSE` |

---

*Documento generado para el portfolio `AG.CreadorContenido` · 2026.*
