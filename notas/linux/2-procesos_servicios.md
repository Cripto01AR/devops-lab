# 02 - Procesos y servicios

## Procesos
- Un **proceso** es un programa en ejecución
- Cada proceso tiene un **PID** (identificador único) y un proceso padre
- El **PID 1** es el primer proceso que arranca y el padre de todos los demás

| Comando | Para qué |
|---|---|
| `ps aux` | Lista todos los procesos del sistema |
| `ps -p <PID>` | Muestra un proceso puntual |
| `top` | Monitor en vivo. `Shift+P` ordena por CPU, `Shift+M` por memoria, `q` sale |
| `comando &` | Corre el comando en segundo plano y libera la terminal |
| `Ctrl+C` | Corta el proceso que está en primer plano |

Al lanzar `sleep 300 &` la terminal devuelve `[1] 5064`:
- `[1]` → número de job (tarea en segundo plano de esta terminal)
- `5064` → PID (identificador en todo el sistema)

Columna **TTY** de `ps`:
- `pts/0` → proceso asociado a una terminal (lo lancé yo)
- `?` → sin terminal: daemons y servicios en segundo plano

## Pipe `|`
Pasa la salida de un comando como entrada de otro.
- `ps aux | grep sleep` → lista los procesos y filtra los que contienen "sleep"
- Ojo: `grep` se encuentra a sí mismo en el resultado, porque su comando también contiene "sleep"

## Señales: terminar procesos
| Comando | Señal | Resultado | Qué hace |
|---|---|---|---|
| `kill <PID>` | SIGTERM (15) | `Terminated` | Le pide al proceso que termine; puede cerrar prolijo |
| `kill -9 <PID>` | SIGKILL (9) | `Killed` | El kernel lo mata en el acto, sin limpieza |

**Regla:** primero `kill` a secas. `-9` solo si no responde (riesgo de datos corruptos).
`kill -l` lista todas las señales.

## Servicios y systemd
- Un **servicio** es un proceso que administra **systemd**: lo arranca al encender, lo reinicia si se cae y guarda sus logs
- Para systemd cada servicio es una **unidad** (ej: `nginx.service`)

| Comando | Qué hace | Cuándo aplica |
|---|---|---|
| `systemctl status nginx` | Estado del servicio | — |
| `systemctl start nginx` | Lo arranca | Ahora |
| `systemctl stop nginx` | Lo para | Ahora |
| `systemctl restart nginx` | Lo para y lo arranca | Ahora |
| `systemctl enable nginx` | Arranca solo al encender el equipo | Próximo inicio |
| `systemctl disable nginx` | No arranca solo al encender | Próximo inicio |
| `systemctl enable --now nginx` | Habilita y arranca de una | Ambos |

**start/stop = ahora. enable/disable = al reiniciar. Son independientes.**
Probado: con `disable` el servicio quedó `disabled` pero seguía `active (running)`.

Cómo leer `systemctl status`:
- `Loaded:` → dónde está definido el servicio y si está `enabled` o `disabled`
- `Active:` → si está corriendo ahora y desde cuándo

Por dentro, `enable` crea un symlink en `/etc/systemd/system/multi-user.target.wants/` y `disable` lo borra.

## Logs
- `journalctl -u nginx` → historial del servicio (arranques, paradas, errores)
- Primer lugar a revisar cuando algo falla

## Errores reales
- `Unit ngix.service could not be found` → typo en el nombre. Usar Tab para autocompletar
- `curl: (7) Failed to connect to localhost port 80` → no hay nada escuchando en ese puerto. Primera pregunta: ¿el servicio está levantado?
- Los comandos que salen bien no imprimen nada (ej: `systemctl stop`). Se verifica con otro comando

## Contenedor vs VM
| | Codespaces (contenedor) | Killercoda (VM) |
|---|---|---|
| PID 1 | `docker-init` | `systemd` |
| systemctl | No funciona | Funciona |

Un contenedor está pensado para correr una aplicación, no un sistema operativo completo, por eso no trae systemd. (Esto se profundiza en Fase 2 con Docker.)

## Diagnóstico: "la web no carga"
Primero corro systemctl status nginx y miro la linea para ver si esta activo -> Active. Si esta parado reviso con journalctl -u nginx y lo levanto con systemctl start nginx
Si esta corriendo pero igual no carga, reviso con curl localhost. Si el servicio responde y funciona bien, el problema esta entre el usuario y el servidor (comunicacion)
