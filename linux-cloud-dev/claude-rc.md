# Sesiones remotas de Claude Code con `claude-rc`

Guía para montar en un servidor Linux sesiones de Claude Code que siguen funcionando sin una conexión SSH abierta y que
se controlan desde [claude.ai/code](https://claude.ai/code) o la app móvil.

> **Solo para servidores de desarrollo y pruebas.** Esta configuración da a Claude permisos totales, sin pedir
> confirmación. No la uses en producción ni en equipos con datos o credenciales que no puedas perder.

## Cómo funciona

```text
claude.ai/code o app móvil
        │  Remote Control (enlace https://claude.ai/code/session_...)
        ▼
   claude --remote-control <nombre>
        ▲
        │  corre dentro de
   sesión tmux <nombre>  ◄── la crea, lista y cierra claude-rc
```

| Pieza | Función |
|---|---|
| tmux | Mantiene Claude corriendo aunque se cierre la conexión SSH |
| CLI de Claude Code | Trabaja en el servidor y expone la sesión con `--remote-control` |
| `~/.local/bin/claude-rc` | Crea, lista y cierra sesiones. La sesión de tmux y la de Remote Control llevan el mismo nombre |
| `~/.claude/settings.json` | Activa los permisos totales por defecto |
| `~/.claude/CLAUDE.md` | Le explica a Claude el comando en cada sesión, para poder pedirle "abre una nueva sesión remota" |

## Requisitos

- Linux con `bash`, `curl`, `python3` y acceso a internet.
- tmux 3.2 o superior (`claude-rc` usa `new-session -e`).
- Una cuenta de claude.ai con acceso a Claude Code. El montaje se probó con un plan Pro; no se probó con una clave de API.

Probado en Ubuntu 26.04, tmux 3.6 y Claude Code 2.1.314 a 2.1.318, **siempre como `root`**. Con un usuario normal los pasos
son los mismos sin `IS_SANDBOX`, pero ese caso no se ha probado.

## Instalación

Ejecuta los pasos en una terminal SSH, con el usuario que vaya a usar Claude. Si le pides a Claude que los haga, el modo
automático puede bloquearle los cambios en `~/.bashrc` y en los permisos de `settings.json`: esos pasos hay que hacerlos a
mano.

### 1. Instalar tmux

En Debian o Ubuntu:

```bash
sudo apt-get update && sudo apt-get install -y tmux
tmux -V
```

### 2. Instalar el CLI de Claude Code

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Queda en `~/.local/bin/claude` y es por usuario. No lo ejecutes con `sudo`: el instalador lo rechaza, porque se instalaría en
el directorio de `root`. Tampoco uses el binario interno de la app de escritorio (`~/.claude/remote/ccd-cli/...`).

### 3. Configurar `~/.bashrc`

```bash
cp ~/.bashrc ~/.bashrc.bak-$(date +%Y%m%d%H%M%S)
grep -q 'HOME/.local/bin' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

**Solo si el usuario es `root`.** Claude Code no permite el modo de permisos totales como `root` sin esta variable:

```bash
grep -q 'IS_SANDBOX' ~/.bashrc || echo 'export IS_SANDBOX=1' >> ~/.bashrc
```

Después carga los cambios y compruébalos:

```bash
source ~/.bashrc
command -v claude && claude --version
```

### 4. Configurar los permisos

Este script conserva lo que ya haya en `settings.json` y hace una copia de seguridad antes:

```bash
[ -f ~/.claude/settings.json ] && cp -a ~/.claude/settings.json ~/.claude/settings.json.bak-$(date +%Y%m%d%H%M%S)
python3 -I - <<'PY'
import json, os
p = os.path.expanduser("~/.claude/settings.json")
os.makedirs(os.path.dirname(p), exist_ok=True)
s = json.load(open(p)) if os.path.exists(p) else {}
s.setdefault("permissions", {})["defaultMode"] = "bypassPermissions"
s["skipDangerousModePermissionPrompt"] = True
if os.geteuid() == 0:
    s.setdefault("env", {})["IS_SANDBOX"] = "1"
with open(p, "w") as f:
    json.dump(s, f, indent=2)
    f.write("\n")
print(open(p).read())
PY
```

### 5. Crear el comando `claude-rc`

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/claude-rc <<'CLAUDE_RC_EOF'
#!/usr/bin/env bash
# claude-rc: gestiona sesiones de Claude Code con Remote Control dentro de tmux.
# La sesión de tmux y la de Remote Control comparten siempre el mismo nombre.
#
#   claude-rc [nombre] [directorio]   crea una sesión (por defecto: <hostname>-MMDD-HHMM en $PWD)
#   claude-rc ls                      lista las sesiones
#   claude-rc stop <nombre>           cierra la sesión (Claude + tmux)

set -euo pipefail

CLAUDE_BIN="${CLAUDE_BIN:-$HOME/.local/bin/claude}"
LINK_TIMEOUT="${CLAUDE_RC_TIMEOUT:-60}"
LINK_RE='https://claude\.ai/code/session_[A-Za-z0-9_-]+'

die() { echo "claude-rc: $*" >&2; exit 1; }

usage() {
  cat <<EOF
Uso:
  claude-rc [nombre] [directorio]   Crea una sesión remota (nombre por defecto: $(hostname -s)-MMDD-HHMM)
  claude-rc ls                      Lista las sesiones
  claude-rc stop <nombre>           Cierra la sesión (Claude + tmux)
EOF
}

valid_name() { [[ "$1" =~ ^[A-Za-z0-9_-]+$ ]]; }
exists() { tmux has-session -t "=$1" 2>/dev/null; }
# "=nombre:" fuerza coincidencia exacta del nombre de sesión en capture-pane
screen() { tmux capture-pane -p -J -S - -t "=$1:" 2>/dev/null; }
link_of() { screen "$1" | grep -oE "$LINK_RE" | tail -n1 || true; }

cmd_ls() {
  if ! tmux ls >/dev/null 2>&1; then
    echo "No hay sesiones."
    return 0
  fi
  printf '%-28s %-40s %s\n' NOMBRE DIRECTORIO ENLACE
  tmux ls -F '#{session_name}	#{session_path}' | while IFS=$'\t' read -r name path; do
    printf '%-28s %-40s %s\n' "$name" "$path" "$(link_of "$name")"
  done
}

cmd_stop() {
  local name="${1:-}"
  [[ -n "$name" ]] || die "indica el nombre: claude-rc stop <nombre>"
  valid_name "$name" || die "nombre no válido: $name"
  exists "$name" || die "no existe la sesión '$name'"
  tmux kill-session -t "=$name"
  echo "Sesión '$name' cerrada."
}

cmd_start() {
  local name="${1:-$(hostname -s)-$(date +%m%d-%H%M)}"
  local dir="${2:-$PWD}"

  valid_name "$name" || die "nombre no válido '$name' (solo letras, números, - y _)"
  [[ -x "$CLAUDE_BIN" ]] || die "no se encuentra el CLI de Claude en $CLAUDE_BIN"
  [[ -d "$dir" ]] || die "el directorio no existe: $dir"
  dir="$(cd "$dir" && pwd -P)"
  exists "$name" && die "ya existe una sesión llamada '$name'"

  local env_args=()
  [[ -n "${IS_SANDBOX:-}" ]] && env_args=(-e "IS_SANDBOX=$IS_SANDBOX")

  tmux new-session -d -s "$name" -c "$dir" "${env_args[@]}" \
    "$(printf '%q --remote-control %q' "$CLAUDE_BIN" "$name")"

  local link="" i
  for ((i = 0; i < LINK_TIMEOUT; i++)); do
    if ! exists "$name"; then
      die "la sesión '$name' terminó antes de mostrar el enlace (¿CLI sin login?). Prueba a ejecutar 'claude' a mano en $dir"
    fi
    link="$(link_of "$name")"
    [[ -n "$link" ]] && break
    sleep 1
  done

  if [[ -z "$link" ]]; then
    echo "claude-rc: no apareció el enlace en ${LINK_TIMEOUT}s. Últimas líneas de la pantalla:" >&2
    screen "$name" | grep -v '^\s*$' | tail -n 15 >&2
    echo >&2
    echo "Si está en la configuración inicial/login o pide confiar en la carpeta, entra con: tmux attach -t '=$name'" >&2
    exit 1
  fi

  cat <<EOF
Sesión remota '$name' lista
  Directorio: $dir
  Enlace:     $link

  Entrar:     tmux attach -t '=$name'     (salir sin cerrar: Ctrl-b d)
  Cerrar:     claude-rc stop $name
EOF
}

case "${1:-}" in
  -h|--help|help) usage ;;
  ls|list) cmd_ls ;;
  stop) shift; cmd_stop "${1:-}" ;;
  *) cmd_start "$@" ;;
esac
CLAUDE_RC_EOF
chmod +x ~/.local/bin/claude-rc
claude-rc --help
```

### 6. Documentarlo en `CLAUDE.md`

Claude lee este archivo en cada sesión, así que entenderá peticiones como "abre una nueva sesión remota" o "cierra la sesión
X" desde cualquier carpeta. Añade esto a `~/.claude/CLAUDE.md`:

```markdown
## Sesiones remotas (`claude-rc`)
Sesiones de Claude Code en tmux con Remote Control, controlables desde claude.ai/code o la app móvil. La sesión de tmux y la de Remote Control tienen el mismo nombre.
- `claude-rc [nombre] [directorio]`: crea la sesión y muestra el enlace. Por defecto: `<hostname>-MMDD-HHMM` y el directorio actual. Nombre: solo letras, números, `-` y `_`.
- `claude-rc ls`: lista las sesiones. `claude-rc stop <nombre>`: cierra Claude y tmux. Los subcomandos van en minúsculas: `claude-rc LS` crea una sesión llamada `LS`.
- Entrar: `tmux attach -t '=<nombre>'` (salir sin cerrar: `Ctrl-b d`). En comandos de tmux, target `=<nombre>:` para coincidencia exacta.
- La carpeta debe ser de confianza (ejecutar `claude` en ella y aceptar) o la sesión se queda esperando esa confirmación.
- Antes de abrir una sesión, comprobar si la carpeta está en git (`git -C <dir> rev-parse --is-inside-work-tree`). Si no lo está, avisar: con permisos totales no se recuperan archivos borrados por error.
- Corren con permisos totales (`defaultMode: bypassPermissions` en `~/.claude/settings.json`). Si una sesión abierta lo devuelve a `auto`, cerrarla y corregir el archivo.
```

## Pasos manuales de la primera vez

Estos pasos no se pueden automatizar. Sin ellos, la sesión de tmux se queda esperando una respuesta y `claude-rc` termina
sin mostrar el enlace.

1. **Iniciar sesión.** Abre una terminal SSH, ejecuta `claude`, elige el tema, inicia sesión y sal con `/exit`.
2. **Confiar en cada carpeta de proyecto.** Ejecuta `claude` dentro de la carpeta y acepta la confirmación. En la prueba, una
   subcarpeta funcionó sin confirmar porque su carpeta superior ya era de confianza.
3. **Responder los avisos de una sola vez.** El CLI puede mostrar avisos propios, por ejemplo "Try the new fullscreen
   renderer?". El enlace aparece igual, pero conviene entrar con `tmux attach` y responderlos.
4. **Comprobar el modo.** Dentro de la sesión, la línea inferior debe decir `bypass permissions on`.

## Uso

| Acción | Comando |
|---|---|
| Crear una sesión | `claude-rc [nombre] [directorio]` |
| Listar las sesiones y sus enlaces | `claude-rc ls` |
| Cerrar una sesión (Claude + tmux) | `claude-rc stop <nombre>` |
| Entrar desde SSH | `tmux attach -t '=<nombre>'` |
| Salir sin cerrar la sesión | `Ctrl-b` y luego `d` |

Valores por defecto y variables:

- **Nombre:** `<hostname>-MMDD-HHMM`. Solo admite letras, números, `-` y `_`, y no se puede repetir.
- **Directorio:** el directorio actual.
- **`CLAUDE_RC_TIMEOUT`:** segundos de espera del enlace (60 por defecto).
- **`CLAUDE_BIN`:** ruta del CLI (`~/.local/bin/claude` por defecto).

Ejemplo:

```bash
claude-rc mi-proyecto /ruta/al/proyecto
```

```text
Sesión remota 'mi-proyecto' lista
  Directorio: /ruta/al/proyecto
  Enlace:     https://claude.ai/code/session_...

  Entrar:     tmux attach -t '=mi-proyecto'     (salir sin cerrar: Ctrl-b d)
  Cerrar:     claude-rc stop mi-proyecto
```

Abre el enlace en el navegador o en la app móvil. También puedes pedirle a Claude "abre una nueva sesión remota", "lista
las sesiones" o "cierra la sesión X".

## Detalles a tener en cuenta

- **Target de tmux.** `capture-pane` necesita el formato `=nombre:` para una coincidencia exacta. Sin el `=`, tmux también
  acepta nombres parciales y puede actuar sobre otra sesión.
- **Subcomandos en minúsculas.** Solo se reconocen `ls`, `list`, `stop` y `help`. `claude-rc LS` no lista nada: crea una sesión
  llamada `LS`. Por lo mismo, esos nombres no se pueden usar para una sesión.
- **Reinicios.** Las sesiones de tmux no sobreviven a un reinicio del servidor; hay que volver a crearlas.
- **Fin de Claude.** Si Claude termina, también termina su sesión de tmux.
- **`IS_SANDBOX` en la sesión.** `claude-rc` solo la pasa a tmux si está definida en el entorno desde el que se ejecuta.
- **SSH sin terminal.** En `ssh servidor comando`, el `~/.bashrc` por defecto de Ubuntu termina antes de llegar a las líneas
  añadidas al final, así que no carga ni el `PATH` ni `IS_SANDBOX`. Usa una terminal interactiva o defínelas en el propio comando.
- **Actualizaciones.** El CLI se actualizó solo de 2.1.314 a 2.1.318 durante la prueba. Los avisos nuevos de una versión pueden
  volver a aparecer.

## Seguridad

- **Sin confirmaciones.** Claude puede borrar archivos o ejecutar cualquier comando sin preguntar, con los permisos del
  usuario. Como `root`, eso alcanza al sistema entero.
- **Usuario sin privilegios.** Si el equipo lo permite, usa un usuario normal en lugar de `root`.
- **Proyectos en git.** Con permisos totales no hay forma de recuperar un archivo borrado por error si la carpeta no está en
  git o sin copia de seguridad. Comprueba antes de abrir la sesión:

  ```bash
  git -C /ruta/al/proyecto rev-parse --is-inside-work-tree
  ```

- **Credenciales.** No dejes en el servidor credenciales de producción. Claude puede leer cualquier archivo al que el
  usuario tenga acceso.
- **Enlaces.** Trata el enlace de la sesión como información privada y no lo publiques.

## Solución de problemas

| Síntoma | Causa | Solución |
|---|---|---|
| `claude: command not found` | `~/.local/bin` no está en el `PATH` | Repite el paso 3 y ejecuta `source ~/.bashrc` |
| `no apareció el enlace en 60s`, con la pantalla de tema o login | El CLI no tiene sesión iniciada | `claude-rc stop <nombre>`, ejecuta `claude` a mano e inicia sesión |
| `no apareció el enlace en 60s`, pidiendo confiar en la carpeta | La carpeta no es de confianza | `tmux attach -t '=<nombre>'` y acepta, o ejecuta `claude` en la carpeta |
| `la sesión terminó antes de mostrar el enlace` | Claude se cerró al arrancar | Ejecuta `claude --remote-control prueba` a mano para ver el error |
| Claude rechaza los permisos totales como `root` | Falta `IS_SANDBOX=1` | Repite el paso 3 y abre una terminal nueva |
| `defaultMode` vuelve a `"auto"` | Una sesión de Claude ya abierta reescribió `settings.json` | Cierra las sesiones abiertas y repite el paso 4 |
| Queda un proceso de Claude tras cerrar tmux | Se vio una vez tras `tmux kill-server` | Búscalo con el comando de abajo y ciérralo con `kill <pid>` |

```bash
ps -eo pid,args | grep '[c]laude --remote-control'
```

## Desinstalar

Primero cierra las sesiones con `claude-rc stop <nombre>`. Después:

```bash
rm -f ~/.local/bin/claude-rc
```

Para quitar los permisos totales, restaura la copia `~/.claude/settings.json.bak-*` o elimina `permissions.defaultMode`,
`skipDangerousModePermissionPrompt` y `env.IS_SANDBOX`. Quita también `export IS_SANDBOX=1` de `~/.bashrc` y la sección
"Sesiones remotas" de `~/.claude/CLAUDE.md`.
