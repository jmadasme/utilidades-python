# Bitácora — utilidades-python

> Registro de decisiones, problemas encontrados y lecciones aprendidas durante el desarrollo y la documentación de las cuatro utilidades.
> El objetivo es que un proyecto futuro no repita los mismos errores.

---

## 1. Contexto

Estas utilidades se escribieron en momentos distintos, sin un plan común. La bitácora se reconstruye a partir del código y de lo que la ejecución demuestra: no hay un historial de commits disponible en el snapshot, así que los hallazgos se verificaron ejecutando los scripts o analizando la lógica.

Método usado:

1. Leer cada archivo completo.
2. Ejecutar los scripts de biblioteca estándar.
3. Reproducir las funciones críticas en un intérprete para confirmar el comportamiento.
4. Contrastar con los vectores de referencia del sistema (`sha256sum`, `base64`, `md5sum`).

---

## 2. Problemas encontrados y cómo se resolvieron

### 2.1 `_cesar.py` se rompe con acentos

**Síntoma.** El cifrado César produce caracteres que no son letras cuando el texto lleva tildes o eñes.

**Causa.** `str.isalpha()` devuelve `True` para `á`, `ñ` y cualquier letra Unicode. El código suma el desplazamiento al código del carácter y luego compara contra `ord('z')`, que es 122. Para `á` (225), la comparación da verdadera y resta 26, produciendo otro símbolo fuera del alfabeto.

**Reproducción:**

```python
cifrado_cesar("año", 3)
# 'dÚr'  — debería ser 'dsq' o similar, no caracteres acentuados
```

**Solución aplicada.** Ninguna: se documenta como limitación. La corrección sería restringir a `a-zA-Z` y usar aritmética modular:

```python
def cifrado_cesar_seguro(texto, k):
    k %= 26
    out = []
    for c in texto:
        if 'a' <= c <= 'z':
            out.append(chr((ord(c) - 97 + k) % 26 + 97))
        elif 'A' <= c <= 'Z':
            out.append(chr((ord(c) - 65 + k) % 26 + 65))
        else:
            out.append(c)
    return "".join(out)
```

**Lección.** `isalpha()` no significa "letra ASCII". En Python 3, todo el rango Unicode entra. Para criptografía clásica hay que acotar el alfabeto explícitamente.

---

### 2.2 `_cesar.py` falla con desplazamientos negativos

**Síntoma.** `cifrado_cesar("abc", -3)` devuelve `` `^_ `` en lugar de `xyz`.

**Causa.** Solo hay una comprobación de desborde hacia arriba (`> ord('z')`). Nunca se resta 26 cuando el resultado queda por debajo de `a`.

**Reproducción:**

```python
cifrado_cesar("abc", -3)   # '`^_'
```

**Solución aplicada.** Documentada. La corrección es el módulo de la solución anterior, que maneja ambos signos.

**Lección.** El cifrado César se suele enseñar con desplazamientos positivos. Probar con negativos es un caso de borde barato que revela la mitad de la lógica faltante.

---

### 2.3 `_rot13.py` no contiene ROT13

**Síntoma.** El archivo se llama `_rot13.py`, pero al leerlo aparece una implementación de XOR.

**Causa.** Probablemente se copió el archivo de XOR y se renombró sin reemplazar el contenido, o se pensaba agrupar varias técnicas en un archivo y quedó solo una.

**Impacto.** Un lector que busque ROT13 no lo encuentra. Un lector que busque XOR lo encuentra dos veces (aquí y en `_xor.py`).

**Solución aplicada.** Se documenta la discrepancia en el catálogo del README y en el datasheet. No se renombra el archivo para no alterar el snapshot.

**Lección.** El nombre de un archivo es documentación. Si el contenido cambia, el nombre también. Un `git mv` es más barato que la confusión que genera.

---

### 2.4 `_AES-256.py`: el acuerdo ECDH no converge

**Síntoma.** El script imprime `Secretos coinciden: False` y el descifrado falla.

**Causa.** `obtener_secreto_compartido()` genera una `salt` aleatoria **dentro** de la función, distinta para cada parte:

```python
def obtener_secreto_compartido(private_key_pem, public_key_pem):
    shared_secret = private_key.exchange(ec.ECDH(), public_key)
    salt = os.urandom(16)                    # ← distinta en cada llamada
    kdf = PBKDF2HMAC(..., salt=salt, ...)
    return kdf.derive(shared_secret), salt
```

Dos derivaciones del mismo secreto con sales distintas producen claves distintas. Es el error clásico de confundir "valor aleatorio" con "valor compartido".

**Solución aplicada.** Documentada, con el ejemplo correcto en el README (§9.4), que usa HKDF con una `salt` de protocolo.

**Lección.** En un KDF, la `salt` es un parámetro **del protocolo**: viaja junto al mensaje o se deriva de un identificador de sesión. Si cada parte la inventa por su cuenta, no hay acuerdo posible.

---

### 2.5 `_AES-256.py`: doble derivación al descifrar

**Síntoma.** Incluso compartiendo la `salt`, el descifrado no devuelve el texto.

**Causa.** Al cifrar se usa `secret_key` directamente como clave AES; al descifrar se aplica PBKDF2 **otra vez** sobre `secret_key`:

```python
# cifrar
cipher = Cipher(algorithms.AES(secret_key), modes.CBC(iv), ...)

# descifrar
derived_key = kdf.derive(secret_key)          # ← segunda derivación
cipher = Cipher(algorithms.AES(derived_key), modes.CBC(iv), ...)
```

**Solución aplicada.** Documentada. La regla es simple: una clave se deriva una vez, en un solo lugar, y ambas partes usan el mismo procedimiento.

**Lección.** Si una función se llama `derivar`, no debe recibir una clave ya derivada. Nombrar las variables `master_key` / `session_key` ayuda a no encadenar derivaciones por accidente.

---

### 2.6 `_aes.py`: la clave no mide lo que dice el comentario

**Síntoma.** El comentario afirma "16 bytes (128 bits)", pero el literal mide 15.

**Causa.**

```python
clave = b'EstaEsMiClave12'   # 15 bytes, no 16
clave = clave.ljust(16, b'\0')
```

Se compensa con `ljust`, pero el relleno introduce un byte nulo que debilita la entropía efectiva y confunde la lectura.

**Solución aplicada.** Documentada. Lo correcto es usar una clave de exactamente 16/24/32 bytes, preferentemente derivada con scrypt o Argon2id.

**Lección.** No confiar en el comentario: contar con `len(clave)`. Un test de una línea lo habría detectado.

---

### 2.7 `_ofusca.py` y `_cifrado_de_flujo_bytes.py`: duplicación

**Síntoma.** Dos archivos con la misma lógica de sustitución por permutación aleatoria.

**Causa.** Evolución incremental: se creó una versión, luego otra con un alfabeto distinto (`string.printable` en lugar de `ascii_letters + digits + punctuation + space`), y quedaron las dos.

**Problema adicional.** El alfabeto de destino tiene 95 símbolos y el de origen 100 (`string.printable` incluye `\t\n\r\x0b\x0c`). `zip()` trunca al más corto, así que los cinco caracteres de control pasan sin transformar y el mapeo no es biyectivo sobre todo el dominio.

**Solución aplicada.** Documentada. Se podría unificar en un solo módulo con el alfabeto como parámetro.

**Lección.** `zip()` no avisa cuando las secuencias tienen distinta longitud: corta en silencio. Cuando el mapeo debe ser total, verificar `len(a) == len(b)` o usar `zip(..., strict=True)` (Python 3.10+).

---

### 2.8 El cifrado de sustitución no sobrevive a la ejecución

**Síntoma.** El texto ofuscado no se puede recuperar en otra corrida.

**Causa.** `generar_mapeo()` usa `random.sample`, que depende del estado del generador Mersenne Twister. El mapeo se crea en memoria y no se guarda.

**Solución aplicada.** Documentada. Si se necesitara persistencia, habría que serializar el mapeo o derivarlo de una semilla secreta guardada aparte. Ninguna de las dos es criptografía: una permutación aleatoria es un cifrado de sustitución monoalfabético, quebrable por análisis de frecuencia.

**Lección.** Un cifrado sin gestión de claves no es un cifrado: es una transformación de un solo uso.

---

### 2.9 `_bin.py` solo admite ASCII

**Síntoma.** `format(valor, '08b')` lanza un error con caracteres fuera del rango 0–255.

**Causa.** La función asume 8 bits por carácter, válido para ASCII y Latin-1, inválido para Unicode.

**Detalle interesante.** La inversión de bits es una involución: aplicarla dos veces devuelve el original. Por eso el script no necesita función de descifrado, aunque no lo dice.

**Solución aplicada.** Documentada.

**Lección.** Un cifrado que es su propia inversa es elegante y frágil a la vez: si se rompe en el medio, no hay forma de detectarlo.

---

### 2.10 `_transposicion.py` es irreversible

**Síntoma.** No hay función de descifrado y el relleno se pierde.

**Causa.** `texto.replace(' ', '')` elimina espacios y `texto += 'X'` agrega relleno sin marcar cuántos caracteres se agregaron. Al revertir no se sabe cuántas `X` eran relleno y cuántas parte del mensaje.

**Solución aplicada.** Documentada. Un formato reversible necesitaría guardar la longitud original.

**Lección.** Todo cifrado debe poder responder: ¿cuánto media el original? Sin ese dato, el relleno es ambiguo.

---

### 2.11 Los scripts no tienen guarda `__main__`

**Síntoma.** `import _rsa` genera un par de claves RSA de 2048 bits y ejecuta una demo.

**Causa.** El código de ejemplo está en el nivel superior del módulo.

**Solución aplicada.** Documentada con dos alternativas (copiar la función o cargar con `importlib` filtrando `stdout`). Ninguna es ideal.

**Lección.** Separar siempre definición y demostración:

```python
if __name__ == "__main__":
    demo()
```

Es una línea y evita que el módulo tenga efectos secundarios al importarse.

---

### 2.12 `main.py` captura `AttributeError` en cada iteración

**Síntoma.** En Windows, el bucle principal lanza y captura una excepción cada 500 ms.

**Causa.**

```python
while True:
    try:
        signal.pause()          # no existe en Windows
    except AttributeError:
        time.sleep(0.5)
```

`signal.pause` no está definido en Windows, así que el `AttributeError` se produce en cada vuelta.

**Impacto.** Ninguno funcional: el programa espera correctamente. El costo es ruido y una excepción por segundo y medio.

**Solución aplicada.** Documentada en `developer.md` con la corrección (resolver `getattr` una vez fuera del bucle). No se aplicó para no alterar el código original.

**Lección.** Las comprobaciones de capacidad del sistema se hacen una vez, al arrancar, no dentro del bucle caliente.

---

### 2.13 `cmdPbl`: el workflow no puede ejecutarse

**Síntoma.** El único archivo es `main.yml.stop`, y los scripts que invoca no existen.

**Causa.** Se archivó el workflow renombrándolo, pero no se conservaron `scriptAgrega.py` ni `firma.py`. Además, el `checkout` apunta a otro repositorio y usa un token que no está.

**Solución aplicada.** Documentado como stub en README, datasheet y developer. No se reactiva.

**Lección.** Desactivar por extensión es una buena práctica de archivo, pero solo si se conserva lo necesario para entenderlo. Un workflow sin sus scripts es un plano sin medidas.

---

## 3. Enfoques descartados y por qué

### 3.1 Empaquetar todo como una librería

**Idea.** Crear `utilidades_python/` con `__init__.py`, submódulos y un `pyproject.toml`.

**Por qué se descartó.**
- Las cuatro versiones no comparten dominio: voz, cripto, suma y un workflow.
- Empaquetar obligaría a reescribir los quince scripts de `EncrypPython` para sacar las demos del nivel superior.
- El snapshot debe conservar el código original. Un paquete implica modificarlo.

**Consecuencia.** Se documenta el uso por copia y se explica por qué no hay `pip install`.

### 3.2 Corregir los bugs en el código

**Idea.** Arreglar `_AES-256.py`, el César, la guarda `__main__`.

**Por qué se descartó.** El mandato del archivo es preservar las versiones. Cambiar el código convertiría el snapshot en una reescritura y rompería la trazabilidad con los repositorios de origen. Los bugs se documentan con reproducción y corrección propuesta, que es más útil para aprender que un arreglo silencioso.

**Consecuencia.** `BITACORA.md` y `developer.md` concentran los hallazgos.

### 3.3 Un README por versión, sin documentación transversal

**Idea.** Dejar que cada carpeta tenga su README y no escribir nada en la raíz.

**Por qué se descartó.** `CSJE` tiene un README de una línea, `cmdPbl` no tiene README y `EncrypPython` solo tiene `VERSION.md`. Sin una vista transversal, el lector no puede comparar ni decidir qué usar.

**Consecuencia.** El README raíz es el documento principal; los README internos se conservan como están.

### 3.4 Generar los diagramas con una herramienta gráfica

**Idea.** Usar un editor vectorial y exportar PNG.

**Por qué se descartó.** Los SVG escritos a mano son texto: se versionan bien, se leen en el diff y no dependen de binarios. Además permiten mantener la paleta exacta sin depender de un editor.

**Consecuencia.** Tres SVG en `docs/assets/`, con la paleta `#06141b`, `#2A9D8F`, `#457B9D`, `#5B7C99`.

### 3.5 Documentar `cmdPbl` como "próximamente"

**Idea.** Suavizar el estado del stub para que el portfolio no parezca incompleto.

**Por qué se descartó.** Un stub presentado como proyecto genera desconfianza cuando el lector abre la carpeta y encuentra un archivo. La honestidad sobre el estado es parte de la documentación técnica.

**Consecuencia.** `cmdPbl` se etiqueta como stub en todas las tablas, con explicación de qué contenía y por qué no ejecuta.

---

## 4. Decisiones técnicas y su justificación

### 4.1 Estructura `versions/` sin tocar

| Decisión | Alternativa | Motivo |
|---|---|---|
| Copiar las versiones tal cual | Reorganizar en `src/` | Preservar trazabilidad con los repos de origen |
| Un `VERSION.md` por carpeta | Un changelog global | Cada versión declara su procedencia |
| Documentación en la raíz | Solo docs internos | Permite comparar las cuatro |

### 4.2 Nombres de los documentos

Se usaron `datasheet.md`, `developer.md`, `REFERENCES.md` y `BITACORA.md`, en minúsculas salvo los acrónimos. La mezcla español/inglés es la convención del portfolio: los nombres de archivo en inglés, el contenido en español.

### 4.3 Paleta de los diagramas

| Color | Uso |
|---|---|
| `#06141b` | Fondo |
| `#2A9D8F` | Acento (estado usable, rutas principales) |
| `#457B9D` | Secundario (estructuras, flujos) |
| `#5B7C99` | Terciario (texto secundario, estados mínimos) |

Se evitó el verde neón y los degradados. Los diagramas usan strokes finos y tipografía monoespaciada para parecer documentación técnica, no una landing page.

### 4.4 Longitud del README

Se apuntó a 25 000–45 000 caracteres. El resultado ronda los 43 000 caracteres (44 900 bytes en UTF-8). La diferencia entre caracteres y bytes importa: los acentos y los caracteres de dibujo ocupan dos o tres bytes cada uno. Cualquier medición automática debe hacerse con `len(texto)`, no con `wc -c`.

**Lección.** Si un requisito habla de caracteres, `wc -c` miente en textos con Unicode.

### 4.5 Sin `docs/index.html`

Se decidió no generar una página HTML. El README y los markdown son el entregable; un `index.html` duplicaría contenido y quedaría desactualizado. Los SVG se referencian directamente.

---

## 5. Lecciones aprendidas

### 5.1 Sobre criptografía

1. **Codificar no es cifrar.** Base64 y hex son transporte. Si alguien dice "cifré en Base64", no cifró.
2. **Hashear no es cifrar.** Un hash no se revierte; un cifrado sí, con la clave.
3. **La primitiva no es el protocolo.** AES es sólido; AES-CBC con clave fija en el código no protege nada.
4. **Sin MAC no hay integridad.** CBC permite modificar el ciphertext sin que el descifrado lo note. GCM o un HMAC lo evitan.
5. **La salt es del protocolo, no de la parte.** Si cada extremo la inventa, no hay acuerdo.
6. **Derivar dos veces no es más seguro.** Es un bug.
7. **Los vectores de prueba son baratos.** `printf '1234' | md5sum` habría detectado varios errores.

### 5.2 Sobre Python

1. **`isalpha()` es Unicode.** Para ASCII, comparar rangos.
2. **`zip()` trunca.** Usar `strict=True` cuando las longitudes deban coincidir.
3. **`import` ejecuta el módulo.** Sin guarda `__main__`, importar es ejecutar.
4. **`wc -c` no son caracteres.** Es bytes.
5. **`format(v, '08b')` asume 8 bits.** Falla fuera de 0–255.
6. **`signal.pause()` no existe en Windows.** Resolver la capacidad una vez, no en el bucle.

### 5.3 Sobre documentación

1. **El estado real es parte del contenido.** Un stub documentado como stub vale más que un stub disfrazado.
2. **Los bugs son documentación.** Reproducirlos con un ejemplo vale más que ocultarlos.
3. **Las tablas comparan mejor que la prosa.** Una matriz de madurez dice más que tres párrafos.
4. **Los diagramas en texto se versionan.** SVG escrito a mano entra en el diff; un PNG no.
5. **Un enlace de ancla se rompe con un número.** `#5-4-...` no existe; GitHub genera `#54-...`. Verificar los anchors antes de publicar.

### 5.4 Sobre el archivo de proyectos

1. **Conservar el original es una decisión, no una pereza.** Permite comparar y aprender del error.
2. **Un workflow desactivado necesita contexto.** Sin los scripts, es un plano sin medidas.
3. **La documentación transversal es la que falta siempre.** Cada carpeta tiene su README; nadie escribe el mapa.
4. **No mezclar repositorios en un repo.** El publicador lo exige por una razón: un repo por proyecto, con su historia.

---

## 6. Pendientes

| # | Tarea | Prioridad | Nota |
|---|---|---|---|
| 1 | Agregar guarda `__main__` a los 15 scripts | Alta | Rompe el patrón actual; hacerlo en una rama aparte |
| 2 | Corregir `_AES-256.py` (salt compartida, una sola derivación) | Alta | Ver §2.4 y §2.5 |
| 3 | Reemplazar CBC por GCM en los ejemplos | Alta | Requiere `cryptography` |
| 4 | Agregar `requirements.txt` para cripto | Media | `pycryptodome`, `cryptography` |
| 5 | Vectores de prueba para cada algoritmo | Media | Comparar con `sha256sum`, `base64`, `md5sum` |
| 6 | Renombrar `_rot13.py` o implementar ROT13 | Media | Ver §2.3 |
| 7 | Unificar los dos scripts de ofuscación | Baja | Ver §2.7 |
| 8 | Resolver `signal.pause` fuera del bucle | Baja | Ver §2.12 |
| 9 | Documentar la procedencia de `cmdPbl` | Baja | Ya está en `VERSION.md` |

Ninguno de estos pendientes se ejecutó: el snapshot se conserva intacto por decisión de archivo.

---

## 7. Cronología de la documentación

| Fecha | Hito |
|---|---|
| 2026 | Importación de las cuatro versiones al portfolio |
| 2026 | Lectura y verificación de los 19 archivos Python |
| 2026 | Reproducción de los bugs de César, AES-256 y ofuscación |
| 2026 | Redacción del README transversal |
| 2026 | Datasheet, guía de desarrollador, referencias y esta bitácora |
| 2026 | Diagramas SVG y licencia MIT |

---

*Bitácora del archivo `utilidades-python` · portfolio `AG.CreadorContenido` · 2026.*
