# 04 - SSH con claves

## Contexto
Configuré mi notebook (Linux Mint) como servidor SSH en mi red local y me conecto desde mi PC con Windows. Es la base del home lab y del proyecto de la Fase 0.

## Conceptos
- **SSH:** terminal remota cifrada. Puerto **22**
- **Cliente:** mi Windows (`ssh`, viene incluido desde Windows 10)
- **Servidor:** `sshd`, corre como servicio en la notebook

Autenticación:
| Método | Riesgo |
|---|---|
| Contraseña | Vulnerable a fuerza bruta |
| Par de claves | La privada nunca viaja por la red |

## Dos pares de claves distintos
| Clave | Dónde vive | Para qué |
|---|---|---|
| Mi clave (`id_ed25519` / `.pub`) | Windows: `C:\Users\<yo>\.ssh\` | Probar que **yo** soy yo |
| Host key del servidor | Notebook: `/etc/ssh/ssh_host_*` | Probar que **el servidor** es quien dice ser |

- Analogía: la **pública** es un candado (cualquiera puede verlo); la **privada** es la llave (nunca se comparte)
- Si roban la pública: no pasa nada
- Si roban la privada: entran como yo a todos los servidores que tengan mi pública → revocar (borrarla de `authorized_keys`) y generar un par nuevo
- **Passphrase:** protege la privada si roban el archivo. Se agrega sin regenerar: `ssh-keygen -p -f <ruta>`

## Fingerprint y known_hosts
- La primera conexión muestra el **fingerprint** de la host key → *confiar en el primer uso*
- Se guarda en `known_hosts` del cliente. Si cambia, SSH advierte: posible ataque o servidor reinstalado
- Verificar en el servidor: `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`
- En HTTPS una CA garantiza la identidad; en SSH no hay CA, confío yo

## Pasos realizados
1. Red: `ip -br a` → interfaz `enx50547be42e25` (el nombre incluye la MAC), IP privada `192.168.1.28/24` por DHCP
2. Instalar servidor: `sudo apt install -y openssh-server`
3. Generar claves en Windows: `ssh-keygen -t ed25519`
4. Copiar la pública desde Windows:
   `type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh usuario@IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"`
   (lo que va entre comillas se ejecuta en el servidor)
5. Permisos: `chmod 700 ~/.ssh` y `chmod 600 ~/.ssh/authorized_keys`
6. Endurecer `/etc/ssh/sshd_config`:
   - `PasswordAuthentication no`
   - `PermitRootLogin no`
7. Validar y aplicar: `sudo sshd -t` → `sudo systemctl restart ssh`
8. Comprobar: `ssh -o PubkeyAuthentication=no usuario@IP` → `Permission denied (publickey)`

## Activación por socket
- `ssh.service` figuraba `inactive (dead)` y `disabled`, pero el 22 escuchaba igual
- En Ubuntu/Mint recientes, **`ssh.socket`** (systemd, PID 1) escucha el puerto y levanta `ssh.service` cuando llega una conexión
- `ss -tlnp` mostraba el 22 en manos de `systemd` y `sshd` a la vez

## Precauciones al tocar sshd_config
- Backup antes: `sudo cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`
- Las líneas con `#` son comentarios: no tienen efecto
- Si una opción aparece dos veces, vale la **primera**. Los archivos en `/etc/ssh/sshd_config.d/` pueden pisar la config
- Verificar lo realmente aplicado: `sudo sshd -T | grep -i <opcion>`
- **Nunca cerrar la sesión actual**: probar en una ventana nueva

## Errores reales
- La edición no se guardó (quedó `#PasswordAuthentication yes`) → verificar con `grep -E "^Opcion" archivo` en vez de confiar
- Fingerprint pedido de nuevo → `known_hosts` borrado o cliente distinto

## Logs
- `sudo journalctl -u ssh` → cada login queda registrado con usuario, IP de origen y método (`password` o `publickey`)

## Preguntas de entrevista

### 1. ¿Qué diferencia hay entre la clave pública y la privada? ¿Dónde va cada una y qué pasa si te roban cada una?
La **pública** va en el servidor (en `~/.ssh/authorized_keys`) y la **privada** queda en el cliente. Una analogía: la pública es un candado, que cualquiera puede ver; la privada es la llave que lo abre.

- **Si roban la pública:** no pasa nada. Es pública por diseño y, sin la privada, nadie puede autenticarse en mi nombre.
- **Si roban la privada:** pueden entrar como yo a todos los servidores que tengan mi pública. Hay que **revocarla** (borrar mi pública del `authorized_keys` de cada servidor) y generar un par nuevo. Si la privada tiene **passphrase**, el archivo robado no sirve sin ella, y eso da tiempo para revocar.

### 2. Copiaste tu clave pública al servidor, pero SSH te sigue pidiendo contraseña o te rechaza. ¿Qué revisás?
Primero, los **logs**, para no adivinar:
- En el servidor: `sudo journalctl -u ssh` → muestra qué pasó y por qué rechazó la conexión.
- En el cliente: `ssh -v usuario@ip` → muestra paso a paso qué claves prueba y dónde falla.

Después, las causas más comunes:
1. **Permisos en el servidor:** verificar con `ls -ld ~/.ssh` y `ls -la ~/.ssh`. La carpeta debe estar en `700` y `authorized_keys` en `600`; se corrigen con `chmod`.
2. **Dueño de los archivos:** si `~/.ssh` o `authorized_keys` se crearon con `sudo`, el dueño es `root` y SSH los ignora aunque los permisos estén bien. Se ve con `ls -la` y se corrige con `chown`.
3. **Contenido de `authorized_keys`:** revisar con `cat ~/.ssh/authorized_keys` que la clave esté completa, en una sola línea, y que coincida con la `.pub` del cliente.
4. **Permisos de la clave privada en el cliente:** si son demasiado abiertos (por ejemplo, que otros usuarios puedan leerla), SSH se niega a usarla.

### 3. ¿Por qué desactivarías `PasswordAuthentication` y `PermitRootLogin`? ¿Qué precaución tomás al modificar `sshd_config` en un servidor remoto?
Son dos pasos fundamentales para blindar un servidor Linux.

- **`PasswordAuthentication no`:** por defecto, SSH permite autenticarse con contraseña, que se puede adivinar por fuerza bruta. Si solo se permiten claves, la fuerza bruta se vuelve inútil.
- **`PermitRootLogin no`:** prohíbe el acceso del superusuario `root` por SSH. Así como los números de puerto son conocidos, el nombre `root` existe en todos los servidores Linux: eso les da a los atacantes la mitad del trabajo. Con esto deshabilitado, ni siquiera pueden intentarlo con `root`. Se entra con un usuario normal y se usa `sudo` cuando hace falta, lo que además deja registrado quién hizo qué.

**Precauciones al modificar `sshd_config`:**
1. Hacer un **backup** antes: `sudo cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`
2. **Validar** la sintaxis antes de aplicar: `sudo sshd -t`
3. **No cerrar la sesión actual:** reiniciar el servicio y probar la conexión en una ventana nueva. Si algo falla, la sesión abierta permite revertir el cambio. En un servidor remoto, sin ella, te quedás afuera.
