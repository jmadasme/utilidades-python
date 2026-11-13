# 📋 Datasheet — SpeechToTextAuxPy

> Ficha técnica para usuario final

---

## Identificación

| Campo | Valor |
|---|---|
| **Nombre** | SpeechToTextAuxPy |
| **Versión** | 1.0.0 |
| **Tipo** | Herramienta de línea de comandos (CLI) |
| **Propósito** | Reconocimiento de voz a texto en tiempo real desde entrada auxiliar (line-in) |
| **Plataforma** | Windows 10 |
| **Idioma de salida** | Español (es-ES) |
| **Última actualización** | Junio 2026 |

---

## Especificación técnica

### Requisitos de hardware

| Componente | Especificación mínima |
|---|---|
| **Procesador** | 1 GHz o superior |
| **Memoria RAM** | 512 MB libres para el proceso |
| **Almacenamiento** | ~10 MB (código fuente + dependencias) |
| **Audio** | Micrófono o entrada auxiliar (line-in) funcional |

### Requisitos de software

| Componente | Versión |
|---|---|
| **Sistema operativo** | Windows 10 (build 10240+) |
| **Python** | 3.10, 3.11, 3.12 o 3.13 |
| **.NET Framework** | 4.7.2 o superior (incluido en Windows 10) |
| **SAPI** | System.Speech (componente de Windows) |
| **Paquete de voz** | Español (España) — `es-ES` |

### Dependencias Python

| Paquete | Versión | Propósito |
|---|---|---|
| `pythonnet` | ≥ 3.0.0 | Puente Python/.NET para acceder a System.Speech |
| `colorama` | ≥ 0.4.6 | Colores en consola Windows (opcional, con fallback) |

### Archivos del producto

| Archivo | Tamaño aprox. | Descripción |
|---|---|---|
| `src/main.py` | 4 KB | Punto de entrada |
| `src/sapi_engine.py` | 8 KB | Motor SAPI vía pythonnet |
| `src/console_utils.py` | 4 KB | Utilidades de impresión |
| `requirements.txt` | < 1 KB | Dependencias |
| `run.bat` | 1 KB | Lanzador Windows |

---

## Comportamiento esperado

### Arranque normal

```
[10:15:32] === SpeechToTextAuxPy ===
[10:15:32] Reconocimiento de voz continuo mediante SAPI (System.Speech)
[10:15:32] Motor SAPI inicializado correctamente (idioma: es-ES)
[10:15:32] Iniciando reconocimiento de voz en español...
[10:15:32] Habla al micrófono/entrada auxiliar. Ctrl+C para salir.
```

### Durante la operación

- Cada frase reconocida se imprime en la consola precedida de un timestamp `[HH:MM:SS]`
- El timestamp aparece en **cian**, el texto reconocido en **blanco brillante**
- Los mensajes informativos aparecen en **verde**
- Los mensajes de error aparecen en **rojo** y se dirigen a `stderr`
- El programa se mantiene ejecutándose de forma continua, sin límite de duración

### Finalización

Al presionar `Ctrl+C`:

```
[10:15:52] Ctrl+C recibido. Deteniendo...
[10:15:52] Reconocimiento de voz detenido.
[10:15:52] Programa terminado.
```

El programa libera todos los recursos COM/.NET y termina con código de salida `0`.

---

## Códigos de salida

| Código | Significado |
|---|---|
| `0` | Finalización normal (Ctrl+C) |
| `1` | Dependencia faltante (pythonnet, colorama) |
| `2` | Error en inicialización de SAPI (idioma no soportado, sin audio) |
| `3` | Error inesperado durante la ejecución |

---

## Casos de uso

### Transcripción de clases o conferencias
Conecta la salida de audio de un grabador o mixer a la entrada line-in del PC. SpeechToTextAuxPy transcribirá el audio en tiempo real.

### Asistencia para subtitulado en vivo
Útil para generar subtítulos en vivo durante presentaciones o eventos, redirigiendo la salida a un archivo o programa de terceros.

### Pruebas de equipos de audio
Verifica rápidamente si un micrófono o entrada auxiliar funciona correctamente hablando y viendo el texto aparecer en pantalla.

### Automatización de entrada de texto
Integra la salida del programa con otras herramientas mediante redirección de consola (`python src/main.py > salida.txt`).

---

## Solución de problemas comunes

### El programa no arranca

| Síntoma | Causa probable | Solución |
|---|---|---|
| `pythonnet no está instalado` | Falta instalar dependencias | `pip install -r requirements.txt` |
| `'python' no se reconoce` | Python no está en PATH | Reinstala Python marcando *Add Python to PATH* |
| Error de .NET | .NET Framework no disponible | Windows 10 incluye .NET 4.7.2+; actualiza Windows |

### El programa arranca pero no reconoce voz

| Síntoma | Causa probable | Solución |
|---|---|---|
| No aparece ningún texto al hablar | Dispositivo de entrada incorrecto | Verifica en *Configuración → Sonido* que el dispositivo correcto sea el predeterminado |
| No aparece ningún texto al hablar | Volumen demasiado bajo | Sube el volumen en *Propiedades del dispositivo → Niveles* |
| `El idioma 'es-ES' no está soportado` | Paquete de voz español no instalado | *Configuración → Hora e idioma → Voz → Agregar voces* |
| El reconocimiento es muy impreciso | Ruido ambiente o mala calidad de audio | Acerca el micrófono o reduce el ruido de fondo |
| Se reconoce en otro idioma | Cultura incorrecta | Verifica que `SapiEngine` se instancia con `language="es-ES"` |

### Problemas con la consola

| Síntoma | Causa probable | Solución |
|---|---|---|
| Caracteres extraños (Ã±, Ã©) | Codificación incorrecta | Usa `run.bat` que ejecuta `chcp 65001` |
| Sin colores en la salida | `colorama` no instalado | Instala con `pip install colorama>=0.4.6` (el programa funciona igual sin colores) |
| La salida no se ve en tiempo real | Buffer de consola | El programa hace `flush()` después de cada mensaje; verifica que la consola no esté en modo pipeline |

---

## Limitaciones conocidas

1. **Solo Windows**: Depende de `System.Speech.dll` de .NET Framework, no disponible en Linux/macOS.
2. **Sin segmentación**: No distingue entre hablantes ni separa por oraciones. Todo el texto reconocido se imprime como una secuencia continua.
3. **Sin persistencia**: No guarda el texto en archivo. Para ello, redirige la salida: `python src/main.py > transcripcion.txt`.
4. **Sin pausa/reamudación**: Solo se puede detener con `Ctrl+C`. No hay comando para pausar/reanudar la escucha.
5. **Sin entrada selectiva**: Usa el dispositivo de audio predeterminado del sistema. No permite seleccionar un dispositivo específico por nombre.
6. **Reconocimiento offline**: SAPI funciona totalmente offline, no requiere conexión a internet.
