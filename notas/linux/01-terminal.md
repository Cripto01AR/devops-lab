# 01 - Terminal y permisos

## Navegación y archivos
- `pwd` - Print working directory: muestra en qué carpeta estamos parados. Ej: `/workspaces/devops-lab`
- `ls` - List: información de los archivos del directorio actual. `-l` formato largo (permisos, dueño, tamaño), `-a` incluye ocultos. Combinados: `ls -la`
- `cd` - Change directory: cambiar de directorio
- `mkdir` - Make directory: crear carpeta. `-p` crea también las carpetas padre: `mkdir -p a/b/c`
- `touch` - Crea archivos vacíos si no existen; si existen, actualiza la fecha de modificación sin alterar el contenido
- `echo` - Imprime texto en consola. Con redirección crea o modifica archivos: `>` pisa el contenido, `>>` agrega al final
- `cat` - Concatenate: muestra el contenido de uno o varios archivos, o los une: `cat a.txt b.txt > combinado.txt`
- `cp` - Copy: copiar archivos. `-a` preserva permisos, dueño y fechas (útil para backups)
- `mv` - Move: mover archivos o carpetas, y también renombrar: `mv viejo.txt nuevo.txt`
- `rm` - Remove: borrar archivos de forma permanente

## Rutas especiales
- `.` → la carpeta actual (`cd .` no cambia nada)
- `..` → la carpeta padre. Se encadena: `cd ../..` sube dos niveles
- `~` → el home del usuario (carpeta personal, ej: `/home/codespace`)
- `-` → la carpeta donde estaba justo antes (`cd -` funciona como "atrás")
- `/` → la raíz de todo el sistema de archivos

## Estructura del sistema de archivos
| Carpeta | Qué hay |
|---|---|
| `/etc` | Configuración del sistema y servicios (ej: `/etc/nginx`, `/etc/ssh`) |
| `/var/log` | Logs. Primer lugar a revisar cuando algo falla |
| `/home` | Carpetas personales de los usuarios |
| `/tmp` | Temporales, se borran al reiniciar |
| `/usr/bin`, `/bin` | Programas y comandos (`which ls` muestra dónde está) |

## Permisos
Ejemplo: `-rwxr-x---`
- Primer carácter: `-` es archivo, `d` es directorio
- Después, tres grupos de `rwx`: **dueño**, **grupo** y **otros**
- r (lectura) = 4, w (escritura) = 2, x (ejecución) = 1. Se suman por grupo
- Cada grupo son 3 bits, por eso el máximo por grupo es 7 (notación octal)
- Ej: `rwxr-x---` = 7 | 5 | 0 = **750**

| Número | Permisos | Uso típico |
|---|---|---|
| 644 | `rw-r--r--` | Archivos normales |
| 755 | `rwxr-xr-x` | Scripts y carpetas |
| 600 | `rw-------` | Claves privadas SSH, archivos sensibles |
| 777 | `rwxrwxrwx` | Todos pueden todo: señal de alarma en un servidor |

Formas de usar chmod:
- Numérica: `chmod 640 archivo` (fija todos los permisos)
- Simbólica: `chmod +x archivo` (agrega solo ejecución sin tocar el resto)

## Error real: Permission denied
Creé `saludo.sh` y al ejecutarlo con `./saludo.sh` tiró `Permission denied`.
Causa: Linux crea los archivos nuevos sin permiso de ejecución, por seguridad.
Solución: `chmod 700 saludo.sh` (o `chmod +x saludo.sh`). Verifiqué con `ls -l`.
El `./` hace falta porque la carpeta actual no está en el PATH.

## Para recordar
- Mínimo privilegio: cada archivo y usuario con los permisos justos, ni uno más
- `rm` es permanente: no hay papelera desde la terminal
- Antes de cada commit, mirar `git status` para saber qué se está subiendo


- En carpetas, la `x` significa "poder entrar", no ejecutar. Por eso carpetas en 755 y archivos en 644

