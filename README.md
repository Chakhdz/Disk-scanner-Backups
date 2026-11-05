# Disk-scanner-Backups

Respaldo público del skill **`disk-scanner-carlos`** para Claude: organiza y audita discos (SSD/HDD/SD) **moviendo, nunca borrando**.

## Qué hace
- **Escanea** discos y carpetas (resumen rápido o árbol detallado).
- **Organiza** archivos por patrón o por fecha EXIF en **bloques fijos de 2 años** (`Prefijo_2023-2024`).
- **Verifica** cada movimiento de forma proporcional al riesgo (existe el destino, ya no existe el origen, tamaño coincide).
- **Registra** cada operación de forma incremental en un solo log por día (`SISTEMA/`).
- **Detecta duplicados** y los mueve a una carpeta de revisión; nunca los elimina.

## Reglas de oro
1. Vista previa y un "sí" explícito antes de ejecutar nada.
2. **Mover, no copiar** (excepto SSD → HDD, donde el HDD es el respaldo).
3. **Nunca borrar nada**: lo sobrante va a `PARA_BORRAR/` para que lo borres tú.
4. Entre discos distintos se arma una carpeta de staging `MOVER_A_<DISCO>/` en el mismo disco de origen; tú la llevas físicamente al destino.

## Uso
Copia `SKILL.md` a una carpeta `disk-scanner-carlos/` dentro de tus skills de Claude (o súbelo desde Ajustes → Skills) y pídele, por ejemplo:
- "Escanea el disco DISK_A y dame el resumen."
- "Organiza VIDEOS por años en bloques de 2."
- "Manda la carpeta PERSONAL al HDD."

## Archivos
| Archivo | Contenido |
|---|---|
| `SKILL.md` | Instrucciones completas del skill (en inglés) |

## Versión
Actualizado el 2026-11-05 con las secciones de fecha (EXIF), nombres y agrupación, trabajos largos, análisis de rostros y escaneo NSFW. Los nombres de discos del ejemplo son genéricos.

## Autor
Carlos Hernández (CHAK).
