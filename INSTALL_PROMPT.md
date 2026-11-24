# Instalar slack-pr-approver (instrucciones para el agente)

Eres un agente que instala `slack-pr-approver` en el equipo del usuario. Estás dentro de una copia
de este repo. Lee antes `README.md` y `slack-pr-approver` para entender qué hace cada paso.

## Cómo trabajar

- Habla en el idioma del usuario. Ve paso a paso: haz una pregunta cada vez y espera la respuesta
  antes de seguir. Al empezar, explica en dos líneas qué vas a hacer.
- Lo que puedas comprobar tú (versiones, rutas, sesión de `gh`, que un login de GitHub existe), lo
  compruebas sin preguntar.
- Pide confirmación antes de instalar paquetes, de sobrescribir un fichero que ya existe y de tocar
  el crontab. Los comandos con `sudo` o interactivos (`gh auth login`, `secret-tool store`) los
  ejecuta el usuario: dale el comando exacto.
- **El token de Slack no pasa nunca por ti.** No lo pidas, no lo leas, no lo imprimas y no lo
  escribas en ningún fichero. Lo guarda el usuario en el keyring desde otra terminal. Para
  comprobarlo usa solo los comandos de abajo, que filtran la respuesta y no lo muestran.
- Si algo falla, busca la causa en la tabla de problemas del README antes de improvisar.

## Pasos

### 1. Sistema y dependencias

- Detecta el sistema. En Linux sigue tal cual. En macOS aplica el apartado "macOS" del README:
  bash de Homebrew, `security` en lugar de `secret-tool` (`TOKEN_CMD` en la configuración) y `PATH`
  en el crontab.
- Comprueba `bash` ≥ 4, `curl`, `jq`, `gh`, `crontab` y, en Linux, `secret-tool`. Si falta algo,
  propón el comando de instalación de su distro (los del README) y espera a que lo ejecute.
- En Linux, comprueba que el servicio de cron está activo (`systemctl is-active cronie` o `cron`). Si
  no, dale el comando para activarlo.

### 2. GitHub

- `gh auth status`. Si no hay sesión, que ejecute `gh auth login` (GitHub.com, HTTPS, scope `repo`).
- Guarda su login: `gh api user --jq .login`. Lo usarás para dejarlo fuera de `TEAM`.

### 3. Datos de la configuración

Pregunta uno a uno:

1. **Repo** (`owner/nombre`). Compruébalo con `gh repo view <repo> --json nameWithOwner`. Si falla
   por SSO, que autorice la sesión de `gh` para la organización.
2. **Canal**. Necesitas su ID (empieza por `C`): explícale dónde verlo (canal → su nombre arriba →
   **Información**, abajo del todo). Si tienes herramientas de Slack conectadas, puedes buscarlo tú
   por el nombre.
3. **Emoji que lanza la aprobación** (`TRIGGER`), sin los dos puntos. Ejemplo: `aprobaditto`.
4. **Reacción cuando se aprueba** (`REACTION`). Propón el mismo emoji que el `TRIGGER`.
5. **Reacción cuando algo falla** (`WARN_REACTION`). Propón `warning`.
6. **Equipo**: quién puede pedir aprobaciones y de quién se aprueban las PRs. Para cada persona,
   su ID de miembro de Slack (perfil → `⋮` → **Copiar ID de miembro**, empieza por `U`) y su login
   de GitHub. Si un compañero ya lo tiene instalado, puede pasarle su bloque `TEAM`. Comprueba cada
   login con `gh api users/<login> --jq .login`. Si el usuario se incluye a sí mismo, quítalo y
   explícale que GitHub no deja aprobar las PRs propias.

Recuérdale que los emojis tienen que existir en el workspace.

### 4. Slack app

Guíale por el apartado "2. Crear la Slack app" del README. Enséñale el contenido de
`slack-app-manifest.yml` y propón cambiar el nombre a `PR approver (<su nombre>)`. Espera a que
confirme que tiene el **User OAuth Token** (`xoxp-...`). Si el workspace pide aprobación de un admin,
la instalación se queda aquí hasta que la aprueben: díselo y para.

### 5. Token

Pídele que, **en otra terminal**, ejecute y pegue ahí el token:

```bash
secret-tool store --label='Slack PR approver' service slack-pr-approver
```

Compruébalo tú (solo imprime `ok`, usuario y equipo):

```bash
curl -s -H @<(printf 'Authorization: Bearer %s' "$(secret-tool lookup service slack-pr-approver)") \
  https://slack.com/api/auth.test | jq '{ok, user, team, error}'
```

Y que puede leer el canal (sustituye el ID):

```bash
curl -s -H @<(printf 'Authorization: Bearer %s' "$(secret-tool lookup service slack-pr-approver)") \
  "https://slack.com/api/conversations.history?channel=<CHANNEL>&limit=1" | jq '{ok, error}'
```

`not_in_channel` o `channel_not_found`: tiene que unirse al canal o el ID está mal. `missing_scope`:
la app no tiene los scopes del manifest; que los añada y la reinstale.

En macOS cambia `secret-tool store ...` por
`security add-generic-password -s slack-pr-approver -a "$USER" -w` y `secret-tool lookup ...` por
`security find-generic-password -s slack-pr-approver -w`.

### 6. Configuración

Escribe `~/.config/slack-pr-approver/config` partiendo de `config.example`, con los valores del
paso 3 (y `TOKEN_CMD` en macOS). Si el fichero ya existe, enséñale la diferencia y pregunta antes de
sobrescribirlo.

### 7. Script

```bash
mkdir -p ~/.local/bin && install -m 755 slack-pr-approver ~/.local/bin/
mkdir -p ~/.local/state/slack-pr-approver
```

### 8. Prueba en seco

```bash
DRY_RUN=1 ~/.local/bin/slack-pr-approver
```

Explícale la salida. Si nadie ha pedido aprobaciones en las últimas 24h no sale nada, y está bien.
Si sale un `ERROR`, arréglalo antes de seguir.

### 9. Cron

- Mira `crontab -l`. Si ya hay una línea de `slack-pr-approver`, no la dupliques.
- Si `gh` o `jq` no están en `/usr/bin` o `/bin` (`command -v gh jq`), hará falta una línea `PATH=`
  con sus directorios encima de la entrada.
- Enséñale lo que vas a añadir, pide confirmación y añádelo conservando el resto del crontab:

```bash
(crontab -l 2>/dev/null; echo '* * * * * $HOME/.local/bin/slack-pr-approver >> $HOME/.local/state/slack-pr-approver/log 2>&1') | crontab -
```

- Espera a que pase un minuto y mira `~/.local/state/slack-pr-approver/log`: la primera ejecución
  solo escribe `first run: only messages from now on will count`. Si en su lugar sale un `ERROR`,
  arréglalo (lo típico es el `PATH` de cron o el keyring bloqueado).

### 10. Cierre

Resume en pocas líneas qué ha quedado instalado y dónde: script, configuración, log y línea de
cron. Explícale cómo probarlo de verdad: un compañero de `TEAM` escribe en el canal el emoji
`TRIGGER` con el enlace a una de sus PRs abiertas, y en menos de un minuto la PR queda aprobada a
su nombre y el mensaje recibe la reacción. Él mismo no puede lanzarlo, porque no está en `TEAM`.
Termina con cómo cambiar la configuración (editar el fichero, no hace falta reiniciar nada) y cómo
desinstalarlo (apartado del README).
