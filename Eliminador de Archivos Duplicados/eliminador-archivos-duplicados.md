# Guía Rápida: Eliminador de Archivos Duplicados en Python

¿Cansado de tener el disco lleno con 5 copias de la misma foto o documentos repetidos por todos lados? Esta herramienta en Python se encarga de hacer la limpieza pesada por ti, de forma rápida, inteligente y sin poner en riesgo tus archivos.

---

## ¿Qué problema te resuelve?

Todos terminamos con carpetas desordenadas tras respaldos acumulados, descargas repetidas o archivos copiados sin querer. Borrar todo a mano es una pesadilla y toma horas. 

Este script soluciona eso en minutos:
* **Recupera espacio en tu disco** eliminando réplicas idénticas.
* **Cero errores humanos**: no se confía en el nombre del archivo (que puede cambiar), sino en su contenido exacto.
* **Guarda un respaldo seguro**: la copia que decide conservar la mueve a una carpeta especial de **Refugio**, para que nunca pierdas nada por accidente.

---

## ¿Cómo funciona?

Para procesar cientos de gigabytes sin congelar tu computadora, el programa trabaja en **dos pasos estratégicos**:

1. **Filtro ultrarrápido por tamaño**: 
   Primero revisa el peso exacto en bytes de cada archivo. Si un archivo pesa algo que nadie más pesa, se descarta al instante sin perder tiempo leyéndolo.
2. **Firma digital (SHA-256)**: 
   A los archivos que coinciden en tamaño les calcula una "huella dactilar" única (Hash SHA-256) leyéndolos por pedacitos (bloques de 64 KB) para no saturar tu memoria RAM. Mientras lo hace, te muestra en consola una **barra de progreso** con el tiempo transcurrido y una estimación de cuánto falta.

---

## Instrucciones de Uso: Paso a Paso

### 1. Prepara la configuración
Abre el script y ajusta la sección de arriba con tus carpetas:

```python
# Directorios donde vas a buscar duplicados:
escanear_directorios = [
    r"C:\Users\TuUsuario\Fotos",
    r"D:\Respaldos\Libros",
]

# Carpeta segura donde se moverá la copia única conservada:
refugio_de_archivos = r"C:\Users\TuUsuario\Desktop\Refugio_de_Archivos"

# Banderilla de seguridad:
simulacion = True  # ¡Déjalo en True para la primera prueba!
```

> **Tip:** Usa la letra `r` antes de las comillas (ej. `r"C:\..."`) para que Windows no confunda las barras invertidas `\`.

### 2. Haz una prueba en Modo Simulación (`simulacion = True`)
Ejecuta el script. En este modo **no se moverá ni borrará nada**. El programa solo escaneará tu disco y te dirá exactamente qué duplicados encontró y qué planea hacer.

### 3. Revisa los resultados
Verás en tu consola la barra de progreso en tiempo real:
```text
 => Procesando: [|||||||||||||||··········]  60.0% (204/340) | Transcurrido: 1m 20s | Restante est.: 0m 45s
```
Y al finalizar, un desglose claro de cada grupo de duplicados encontrado.

### 4. Ejecuta la limpieza real (`simulacion = False`)
Cuando estés conforme con lo que viste en la simulación:
1. Cambia la variable a `simulacion = False`.
2. Vuelve a ejecutar el script.
3. ¡Listo! La copia única quedará guardada en tu carpeta de Refugio y las réplicas sobrantes se borrarán automáticamente.

---

## Código Fuente Completo

Puedes copiar este código directo a tu editor de Python:

```python
import hashlib
from pathlib import Path
import shutil
import sys
import time

# =========================================================================
#            C O N F I G U R A C I Ó N   D E   U S U A R I O
# =========================================================================

# Rutas de las carpetas a revisar:
escanear_directorios = [
    r"C:\Ruta\A\Tu\Carpeta1",
]

# Carpeta destino para la copia única conservada:
refugio_de_archivos = r"C:\Ruta\A\Tu\Refugio_de_Archivos"

# Modo seguro:
#   True  = Simulación (Modo prueba: solo muestra en pantalla)
#   False = Limpieza real (Mueve y elimina archivos)
simulacion = True

# =========================================================================


def formatear_tiempo(segundos: float) -> str:
  """Convierte segundos a minutos y segundos legibles."""
  minutos, segs = divmod(int(segundos), 60)
  horas, mins = divmod(minutos, 60)
  if horas > 0:
    return f"{horas} h ~ {minutos:02d} m ~ {segs:02d} s"
  return f"{minutos} m ~ {segs:02d} s"


def mostrar_progreso(actual: int, total: int, tiempo_inicio: float, prefijo: str = "Procesando") -> None:
  """Dibuja la barra de progreso en vivo en la consola."""
  if total == 0:
    return

  porcentaje = (actual / total) * 100
  longitud_barra = 25
  bloques = int(longitud_barra * actual // total)
  barra = "█" * bloques + "░" * (longitud_barra - bloques)

  tiempo_transcurrido = time.time() - tiempo_inicio
  velocidad = actual / tiempo_transcurrido if tiempo_transcurrido > 0 else 0
  tiempo_restante = ((total - actual) / velocidad if velocidad > 0 else 0)

  texto_transcurrido = formatear_tiempo(tiempo_transcurrido)
  texto_restante = formatear_tiempo(tiempo_restante)

  sys.stdout.write(
      f"\r{prefijo}: [{barra}] {porcentaje:5.1f}% ({actual}/{total}) | "
      f"Transcurrido: {texto_transcurrido} | Restante est.: {texto_restante}  "
  )
  sys.stdout.flush()


def calcular_hash(ruta_archivo: str, tamanio_bloque: int = 65536) -> str:
  """Calcula la firma SHA-256 leyendo el archivo en bloques de 64 KB."""
  hasher = hashlib.sha256()
  try:
    with open(ruta_archivo, "rb") as firma_digital:
      while bloque := firma_digital.read(tamanio_bloque):
        hasher.update(bloque)
      return hasher.hexdigest()
  except Exception as error:
    print(f"\n ### ERROR AL LEER ~ {ruta_archivo} ~ : {error}")
    return ""


def buscar_duplicados(directorios: list[str]) -> dict[str, list[str]]:
  """Encuentra archivos idénticos usando tamaño y firma Hash."""
  archivos_por_tamanio = {}
  print("\n ===== Paso 1: Escaneando carpetas y agrupando por tamaño... =====")

  for directorio in directorios:
    ruta_base = Path(directorio)
    if not ruta_base.exists():
      print(f"El directorio >> {directorio} << no existe. Omitiendo...")
      continue

    for ruta in ruta_base.rglob("*"):
      if ruta.is_file():
        try:
          tamanio = ruta.stat().st_size
          archivos_por_tamanio.setdefault(tamanio, []).append(str(ruta))
        except Exception as error:
          print(f"No se pudo leer >> {ruta} << | {error}")

  candidatos = {
      tamanio: rutas
      for tamanio, rutas in archivos_por_tamanio.items()
      if len(rutas) > 1
  }

  total_candidatos = sum(len(rutas) for rutas in candidatos.values())

  if total_candidatos == 0:
    print(" ======= No se encontraron archivos con pesos coincidentes =====")
    return {}

  print(
      f"\n ===== Paso 2: Verificando contenido exacto ({total_candidatos} archivos a revisar)... ====="
  )
  duplicados_por_hash = {}
  procesados = 0
  tiempo_inicio_hash = time.time()

  for tamanio, rutas in candidatos.items():
    for ruta in rutas:
      nombre_hash = calcular_hash(ruta)
      if nombre_hash:
        duplicados_por_hash.setdefault(nombre_hash, []).append(ruta)

      procesados += 1
      mostrar_progreso(procesados, total_candidatos, tiempo_inicio_hash, prefijo=" => Procesando ")

  print("\n")

  return {nombre_hash: rutas for nombre_hash, rutas in duplicados_por_hash.items() if len(rutas) > 1}


def procesar_duplicados(duplicados: dict[str, list[str]], refugio_de_archivos: str, simular: bool,) -> None:
  """Mueve la copia principal al Refugio y borra las repetidas."""
  if not duplicados:
    print("\n ===== ¡Todo limpio! No hay duplicados =====\n")
    return

  total_duplicados = sum(len(rutas) - 1 for rutas in duplicados.values())
  print(
      f"\n Se encontraron {len(duplicados)} grupos de duplicados"
      f" ({total_duplicados} archivos repetidos)."
  )

  if simular:
    print(
        "\n ========================= MODO SIMULACIÓN ACTIVADO"
        " ====================\n"
        "\n No se ha movido ni borrado nada."
        "\n Para aplicar los cambios reales, cambia 'simulacion = False' en el"
        " script.\n"
        "\n ==========================================================================="
    )

  path_refugio = Path(refugio_de_archivos)

  for indice, (nombre_hash, rutas) in enumerate(duplicados.items(), start=1):
    original = rutas[0]
    copias_restantes = rutas[1:]
    nombre_base = Path(original).name

    print(f"\nGrupo duplicado #{indice}:")
    print(f" [Mover a Refugio] -> {original}")
    for copia in copias_restantes:
      print(f" [Eliminar]        -> {copia}")

    if not simular:
      try:
        path_refugio.mkdir(parents=True, exist_ok=True)

        destino_refugio = path_refugio / nombre_base
        contador = 1
        while destino_refugio.exists():
          nombre_nuevo = (
              f"{Path(nombre_base).stem}_{contador}{Path(nombre_base).suffix}"
          )
          destino_refugio = path_refugio / nombre_nuevo
          contador += 1

        shutil.move(original, destino_refugio)
        print(f" Guardado en Refugio: {destino_refugio.name}")

        for copia in copias_restantes:
          try:
            Path(copia).unlink()
            print(f" Eliminado con éxito: {copia}")
          except Exception as error:
            print(f" ERROR AL ELIMINAR > {copia} < : {error}")

      except Exception as error:
        print(f" ERROR AL MOVER AL REFUGIO > {original} < : {error}")

    print("-" * 60)


if __name__ == "__main__":
  duplicados = buscar_duplicados(escanear_directorios)
  procesar_duplicados(duplicados, refugio_de_archivos, simular=simulacion)
```