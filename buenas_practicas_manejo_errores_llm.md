# Guía de Buenas Prácticas: Manejo de Errores en Scripts con LLMs

> **Propósito**: Este documento recopila y sistematiza los patrones y prácticas de manejo de errores implementados en este repositorio (`genai`) para scripts que realizan consultas a Modelos de Lenguaje (LLMs como Google Gemini y OpenRouter). Sirve como base técnica para construir una **Skill de Antigravity** que genere o refactorice scripts con estos estándares de resiliencia y robustez.

---

## Índice de Contenidos

1. [Validación Previa y Entorno (Pre-flight Checks)](#1-validación-previa-y-entorno-pre-flight-checks)
2. [Gestión de Errores de API, Red y Rate Limiting (429 / Quotas)](#2-gestión-de-errores-de-api-red-y-rate-limiting-429--quotas)
3. [Estrategia de Model Fallbacks (Modelos de Respaldo en Cascada)](#3-estrategia-de-model-fallbacks-modelos-de-respaldo-en-cascada)
4. [Gestión de Recursos y Limpieza Garantizada (`finally`)](#4-gestión-de-recursos-y-limpieza-garantizada-finally)
5. [Parseo Defensivo y Reparación de Respuestas Estructuradas (JSON)](#5-parseo-defensivo-y-reparación-de-respuestas-estructuradas-json)
6. [Persistencia Atómica y Reanudación de Estado (Checkpointing / Resume)](#6-persistencia-atómica-y-reanudación-de-estado-checkpointing--resume)
7. [Sanitización Determinística y Filtros Pre/Post LLM](#7-sanitización-determinística-y-filtros-prepost-llm)
8. [Plantilla Modelo para Nuevos Scripts](#8-plantilla-modelo-para-nuevos-scripts)
9. [Diseño para la Futura Skill de Antigravity](#9-diseño-para-la-futura-skill-de-antigravity)

---

## 1. Validación Previa y Entorno (Pre-flight Checks)

Antes de iniciar cualquier proceso intensivo o costoso con un LLM, el script debe validar todas las dependencias y precondiciones para fallar rápido (*Fail Fast*) con mensajes claros.

### Patrones Implementados:
- **Verificación de API Keys:** Validación explícita de `os.environ.get("GEMINI_API_KEY")` o `os.environ.get("OPENROUTER_API_KEY")` según el backend seleccionado, alertando al usuario y saliendo con `sys.exit(1)` antes de invocar la API.
- **Validación de Archivos de Entrada:** Comprobación de existencia de insumos (`schema.json`, PDFs, JSONs de pasos anteriores) con sugerencias de qué script debe ejecutarse previamente.
- **Importaciones Opcionales con Flags:** Manejo de paquetes opcionales o nativos (como `pdf_inspector`) mediante bloques `try/except ImportError` y banderas booleanas (`HAS_PDF_INSPECTOR = True/False`).

```python
# Ejemplo de verificación de variables de entorno y archivos
api_key = os.environ.get("GEMINI_API_KEY")
if not api_key:
    print("Error: No se encontró GEMINI_API_KEY en el entorno o archivo .env.")
    sys.exit(1)

if not input_file.exists():
    print(f"Error: No se encontró '{input_file.name}'. Ejecuta primero el paso anterior.")
    sys.exit(1)
```

---

## 2. Gestión de Errores de API, Red y Rate Limiting (429 / Quotas)

Las llamadas a LLMs pueden fallar por límites de tasa (*rate limit / 429*), saturación de cuota (`RESOURCE_EXHAUSTED`), timeouts de red o errores de servidor (5xx).

### Patrones Implementados:
- **Reintentos con Backoff Diferenciado:**
  - Si el error contiene `"429"`, `"RESOURCE_EXHAUSTED"`, `"Rate Limit"` o `"Quota"`, se aplica una espera mayor (ej. 10 a 12 segundos, adaptada a la ventana de cuota por minuto de Google Gemini).
  - Para otros errores transitorios de red o servidor, se usa una espera más corta (ej. 3 a 4 segundos).
- **Control de Intentos Totales:** Uso de bucles con `intentos_totales = max_reintentos + 1` y captura de `ultimo_error` para lanzar una excepción informativa si se agotan todos los reintentos (`RuntimeError`).
- **Timeouts Explícitos en HTTP:** En llamadas directas (ej. OpenRouter con `urllib.request`), se define explícitamente `timeout=120` para evitar bloqueos indefinidos.

```python
def ejecutar_con_reintentos(client, file_ref, prompt, max_reintentos=3, espera_segundos=3.0):
    intentos_totales = max_reintentos + 1
    ultimo_error = None

    for intento in range(1, intentos_totales + 1):
        try:
            return llamar_api_llm(client, file_ref, prompt)
        except Exception as e:
            ultimo_error = e
            err_msg = str(e)
            
            # Backoff extendido para cuota agotada / 429
            tiempo_espera = 12.0 if ("429" in err_msg or "RESOURCE_EXHAUSTED" in err_msg or "Quota" in err_msg) else espera_segundos

            if intento < intentos_totales:
                print(f"\n   [Aviso] Pausa en intento {intento}/{intentos_totales}. Esperando {tiempo_espera}s...", flush=True)
                time.sleep(tiempo_espera)

    raise RuntimeError(f"Fallo persistente tras {intentos_totales} intentos: {ultimo_error}")
```

---

## 3. Estrategia de Model Fallbacks (Modelos de Respaldo en Cascada)

Cuando una cuota de modelo se agota completamente o un modelo experimenta degradación, el script conmuta automáticamente a modelos alternativos compatibles.

### Patrones Implementados:
- **Lista de Modelos en Cascada:**
  En `detectar_paginas_irrelevantes.py`, se define una lista ordenada (`["gemini-2.5-flash", "gemini-3.5-flash", "gemini-2.0-flash", "gemini-1.5-flash"]`).
- **Conmutación Inteligente ante 429:**
  Si el modelo actual arroja error de cuota persistente, se rompe el ciclo interno de ese modelo (`break`) y se avanza de inmediato al siguiente modelo disponible.

```python
modelos = ["gemini-2.5-flash", "gemini-3.5-flash", "gemini-2.0-flash", "gemini-1.5-flash"]
for model_name in modelos:
    for intento in range(1, 3):
        try:
            # Ejecutar llamada con model_name
            return respuesta
        except Exception as e:
            err_msg = str(e)
            if "429" in err_msg or "RESOURCE_EXHAUSTED" in err_msg:
                print(f"Cuota agotada en {model_name}. Conmutando al siguiente modelo...")
                time.sleep(10)
                break  # Pasa al siguiente modelo
            time.sleep(3)
```

---

## 4. Gestión de Recursos y Limpieza Garantizada (`finally`)

Al utilizar APIs multimodales con subida de archivos (como Gemini Files API), los archivos temporales subidos consumen cuota de almacenamiento y deben eliminarse siempre, incluso ante errores no controlados.

### Patrones Implementados:
- **Garantía mediante `try...finally`:**
  El archivo se sube con reintentos antes del proceso principal, y el bloque `finally:` garantiza la llamada a `client.files.delete(name=file_ref.name)`.
- **Silenciamiento Seguro en Limpieza:**
  La eliminación en el bloque `finally` se envuelve en su propio `try/except Exception: pass` para no ocultar la excepción original si la eliminación de red falla.

```python
file_ref = None
try:
    file_ref = subir_pdf_con_reintentos(client, pdf_path)
    # ... operaciones con el LLM ...
finally:
    if file_ref and client:
        try:
            client.files.delete(name=file_ref.name)
            print("Archivo temporal limpiado de Gemini storage.")
        except Exception:
            pass
```

---

## 5. Parseo Defensivo y Reparación de Respuestas Estructuradas (JSON)

Los LLMs pueden devolver JSONs con caracteres no escapados, bloques markdown (````json ... ````), comillas simples o estructuras truncadas por límites de tokens.

### Patrones Implementados:
- **Forzado en la Configuración:**
  Uso de `types.GenerateContentConfig(response_mime_type="application/json")`.
- **Estrategia de Desempaquetado y Reparación Multinivel:**
  1. *Limpieza de Markdown:* Remoción de bloques ````json ... ```` o ```` ... ```` si el modelo los incluyó.
  2. *Intento con Parser Estándar:* `json.loads(texto, strict=False)`.
  3. *Fallback a `json_repair`:* Si `json.loads` lanza excepción, se recurre a `json_repair.repair_json(texto, return_objects=True)`.
  4. *Acceso Seguro con Default:* Extracción de claves esperadas con `.get("clave", fallback_texto)` en lugar de acceso directo por corchetes.

```python
def extraer_limpiar_json(texto: str) -> dict:
    texto_limpio = texto.strip()
    # 1. Remover delimitadores de código markdown
    if texto_limpio.startswith("```"):
        lineas = texto_limpio.splitlines()
        if lineas[0].startswith("```"):
            lineas = lineas[1:]
        if lineas and lineas[-1].startswith("```"):
            lineas = lineas[:-1]
        texto_limpio = "\n".join(lineas).strip()

    # 2. Parseo estándar y fallback con json_repair
    try:
        return json.loads(texto_limpio, strict=False)
    except Exception:
        return json_repair.repair_json(texto_limpio, return_objects=True)
```

---

## 6. Persistencia Atómica y Reanudación de Estado (Checkpointing / Resume)

En pipelines largos que procesan documentos sección por sección o lote por lote, un fallo en el elemento 40 de 50 no debe obligar a reiniciar desde cero.

### Patrones Implementados:
- **Autoguardado Inmediato (Checkpoint tras cada item):**
  Cada vez que un bloque o sección termina de procesarse, se sobrescribe de inmediato el archivo de salida (`schema_completado.json`, `schema_completado_espanol.json`).
- **Detección y Carga de Estado Previo:**
  Al iniciar la ejecución, se verifica si el archivo de salida ya existe:
  - Si existe: Se cargan los datos y se identifican únicamente los nodos/secciones pendientes (`v == ""`).
  - Si todas las secciones están completas: Termina con éxito informando al usuario (`sys.exit(0)`).
- **Protección de Salida ante Errores:**
  Si ocurre una interrupción no controlada (ej. `KeyboardInterrupt` o error de red), se informa que el avance hasta ese momento quedó guardado y se puede reanudar.

```python
# Carga con reanudación
if output_file.exists():
    print("Detectado avance previo. Reanudando...")
    with open(output_file, "r", encoding="utf-8") as f:
        working_data = json.load(f)
else:
    with open(template_file, "r", encoding="utf-8") as f:
        working_data = json.load(f)

# Guardar en cada iteración
for ruta, parent_dict, clave in secciones_pendientes:
    texto = extraer_texto(...)
    parent_dict[clave] = texto
    guardar_progreso(output_file, working_data)  # Persistencia inmediata
```

---

## 7. Sanitización Determinística y Filtros Pre/Post LLM

No se debe depender únicamente de que el LLM obedezca instrucciones negativas ("no incluyas pies de página", "no dupliques títulos").

### Patrones Implementados:
- **Filtros Heurísticos Previos (Evitar llamadas innecesarias):**
  - Identificación de secciones de bibliografía/referencias mediante `es_seccion_referencias(k)` para no gastar tokens ni cuota en extraer o traducir listas de citas.
- **Saneamiento Determinístico Post-LLM (Regex y Limpieza de Texto):**
  - `limpiar_footers_y_headers()`: Expresiones regulares que barren rutas de archivo (`.doc`, `.pdf`), estampas de fecha/hora (`20-07-2014@22.32`), números de página (`Page X of Y`), avisos de derechos de autor y marcas de agua.
  - `remover_titulo_duplicado()`: Remueve el encabezado repetido al principio del texto si el modelo lo insertó por error.

---

## 8. Plantilla Modelo para Nuevos Scripts

A continuación se presenta el esqueleto estándar que reúne todas las buenas prácticas anteriores:

```python
import os
import sys
import json
import time
from pathlib import Path
from dotenv import load_dotenv
from google import genai
from google.genai import types
import json_repair

load_dotenv()

def validar_entorno() -> str:
    """Verifica variables de entorno y devuelve la clave de API."""
    key = os.environ.get("GEMINI_API_KEY")
    if not key:
        print("Error: No se encontró GEMINI_API_KEY en el entorno o archivo .env.")
        sys.exit(1)
    return key

def parsear_json_seguro(texto: str) -> dict:
    """Parsea JSON con soporte para json_repair como fallback."""
    texto_limpio = texto.strip()
    if texto_limpio.startswith("```"):
        lineas = texto_limpio.splitlines()
        texto_limpio = "\n".join(lineas[1:-1] if lineas[-1].startswith("```") else lineas[1:]).strip()
    try:
        return json.loads(texto_limpio, strict=False)
    except Exception:
        res = json_repair.repair_json(texto_limpio, return_objects=True)
        return res if isinstance(res, dict) else {"resultado": str(res)}

def ejecutar_consulta_con_reintentos(client: genai.Client, file_ref, prompt: str, max_reintentos: int = 3) -> dict:
    """Ejecuta una consulta al LLM con reintentos exponenciales y backoff diferenciado."""
    for intento in range(1, max_reintentos + 2):
        try:
            response = client.models.generate_content(
                model="gemini-2.5-flash",
                contents=[file_ref, prompt] if file_ref else [prompt],
                config=types.GenerateContentConfig(
                    response_mime_type="application/json",
                    temperature=0.1
                )
            )
            return parsear_json_seguro(response.text or "")
        except Exception as e:
            err_msg = str(e)
            espera = 12.0 if ("429" in err_msg or "RESOURCE_EXHAUSTED" in err_msg) else 3.0
            if intento <= max_reintentos:
                print(f"   [Aviso] Intento {intento}/{max_reintentos + 1} falló. Esperando {espera}s...", flush=True)
                time.sleep(espera)
            else:
                raise RuntimeError(f"Fallo persistente tras {max_reintentos + 1} intentos: {e}")

def main():
    validar_entorno()
    client = genai.Client()
    file_ref = None

    try:
        # 1. Subida segura si aplica
        # file_ref = subir_pdf_con_reintentos(client, ruta_pdf)
        
        # 2. Ejecución con reintentos y parseo robusto
        # resultado = ejecutar_consulta_con_reintentos(client, file_ref, "PROMPT...")
        pass
    except Exception as e:
        print(f"\nError durante el procesamiento: {e}")
        sys.exit(1)
    finally:
        # 3. Limpieza garantizada de recursos
        if file_ref:
            try:
                client.files.delete(name=file_ref.name)
                print("Archivo temporal eliminado.")
            except Exception:
                pass

if __name__ == "__main__":
    main()
```

---

## 9. Diseño para la Futura Skill de Antigravity

Para convertir esta guía en una **Skill de Antigravity** reutilizable (`llm-error-handling` o `robust-llm-scripting`), la estructura recomendada en `.agents/skills/` o en la configuración global es:

```text
skills/
└── robust-llm-scripting/
    ├── SKILL.md                 # Definición de la Skill, directivas y cuándo invocarla
    ├── references/
    │   └── error_patterns.md    # Catálogo de errores (429, timeouts, malformed JSON, etc.)
    └── templates/
        └── script_template.py   # Plantilla base lista para rellenar
```

### Directivas clave a incluir en el `SKILL.md`:
1. **Obligatoriedad de `try/finally`** para cualquier recurso creado en la nube (Files API / Sockets).
2. **Backoff adaptativo:** Esperar >= 10s ante `RESOURCE_EXHAUSTED` / `429`, >= 3s ante fallos genéricos.
3. **Parseo defensivo:** Nunca asumir que `json.loads()` funcionará directamente; usar siempre pipeline de limpieza + fallback `json_repair`.
4. **Checkpointing granular:** Guardar el estado en disco tras procesar cada elemento de una lista o árbol para soportar reanudación tras fallos.
5. **Sanitización determinística complementaria:** Tratar las instrucciones negativas de los prompts como directrices y reforzarlas con filtros por código (Regex).
