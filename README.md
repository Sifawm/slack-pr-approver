# slack-pr-approver

Script que cada minuto lee un canal de Slack y aprueba **con tu usuario de GitHub** las PRs que tu
equipo pide aprobar.

Alguien del equipo escribe en el canal:

> :aprobaditto: https://github.com/mi-org/mi-repo/pull/1234

y en menos de un minuto la PR queda aprobada a tu nombre y el mensaje recibe tu reacción
:aprobaditto:. El emoji que lanza la aprobación y el de la reacción se configuran.

## Qué hace exactamente

Un mensaje cuenta solo si cumple todo esto:

- Es un mensaje de primer nivel del canal (las respuestas en hilo se ignoran).
- Lo ha escrito alguien de `TEAM`.
- Contiene el emoji `TRIGGER` y al menos un enlace a `github.com/<REPO>/pull/<n>`.

Cada PR enlazada se aprueba si está abierta, no es draft, no es tuya y su autor está en `TEAM`.
Después el mensaje recibe la reacción `REACTION` si se aprobó alguna (o ya estaba aprobada), y
`WARN_REACTION` si alguna no se pudo aprobar. El motivo queda en el log.

Nunca mira más de 24h atrás. Si un mensaje antiguo se edita para añadirle el emoji, no se vuelve a
procesar. Si una llamada a Slack falla, el estado no avanza y la siguiente ejecución lo reintenta.

Como usa un token de usuario de Slack, lee el canal y reacciona como tú: no hay que invitar a ningún
bot, pero tienes que ser miembro del canal.

## Ficheros

| Fichero | Qué es |
|---|---|
| `slack-pr-approver` | El script |
| `config.example` | Plantilla de la configuración |
| `slack-app-manifest.yml` | Manifest para crear la Slack app |
| `INSTALL_PROMPT.md` | Instrucciones para que un agente te lo instale paso a paso |

## Instalación con un agente

Clona el repo y pídele a tu agente (Claude Code, Codex, Cursor…) que siga `INSTALL_PROMPT.md`:

```bash
git clone https://github.com/Sifawm/slack-pr-approver.git
cd slack-pr-approver
claude "Sigue INSTALL_PROMPT.md para instalarme slack-pr-approver"
```

El agente comprueba las dependencias, te pregunta el canal, el repo, el equipo y los emojis, te
guía para crear la Slack app, escribe la configuración, hace una prueba en seco y programa el cron.
El token de Slack lo guardas tú en el keyring: el agente no lo ve en ningún momento.

Si prefieres hacerlo a mano, sigue los pasos de abajo.

## Instalación manual

Probado en Linux con GNOME Keyring o KWallet. Para macOS, ver el apartado del final.

### Requisitos

`bash` ≥ 4, `curl`, `jq`, `gh` (GitHub CLI), `secret-tool` y `cron`.

```bash
# Arch/Manjaro
sudo pacman -S --needed jq github-cli libsecret cronie && sudo systemctl enable --now cronie
# Debian/Ubuntu
sudo apt install jq gh libsecret-tools cron
```

### 1. GitHub CLI

El script aprueba con la sesión de `gh`, así que las aprobaciones salen a tu nombre.

```bash
gh auth login             # GitHub.com, HTTPS, con scope "repo"
gh api user --jq .login   # debe imprimir tu login
```

Si el repo es de una organización con SSO, autoriza la sesión de `gh` para esa organización.

### 2. Crear la Slack app

Cada persona crea la suya.

1. Entra en <https://api.slack.com/apps> → **Create New App** → **From a manifest**.
2. Elige el workspace.
3. Pega el contenido de [`slack-app-manifest.yml`](slack-app-manifest.yml). Si quieres, cambia el
   nombre (`PR approver (tu nombre)`), y pulsa **Create**.
   `token_rotation_enabled` tiene que quedarse en `false`: con rotación el token caduca a las 12h.
4. **Install App** → **Install to Workspace** → **Allow**. Si el workspace exige aprobación de un
   admin, saldrá **Request to Install**: pídela y espera a que la aprueben.
5. En **OAuth & Permissions**, copia el **User OAuth Token** (empieza por `xoxp-`).

Scopes que pide, todos de usuario: `channels:history` y `groups:history` para leer el canal (público
o privado) y `reactions:write` para reaccionar.

### 3. Guardar el token en el keyring

```bash
secret-tool store --label='Slack PR approver' service slack-pr-approver
# pega el token xoxp-... y pulsa Enter
```

Comprobación (debe devolver `"ok": true` con tu usuario):

```bash
curl -s -H "Authorization: Bearer $(secret-tool lookup service slack-pr-approver)" \
  https://slack.com/api/auth.test | jq '{ok, user, team, error}'
```

Si quieres guardarlo de otra forma (un gestor de contraseñas, otro atributo…), cambia `TOKEN_CMD`
en la configuración por cualquier comando que imprima el token.

### 4. Configurar

```bash
mkdir -p ~/.config/slack-pr-approver
cp config.example ~/.config/slack-pr-approver/config
$EDITOR ~/.config/slack-pr-approver/config
```

| Variable | Qué es |
|---|---|
| `CHANNEL` | ID del canal (empieza por `C`). Abre el canal → su nombre arriba → **Información**, abajo del todo |
| `REPO` | Repo de GitHub, como `owner/nombre` |
| `TRIGGER` | Emoji que pide la aprobación, sin los dos puntos (`aprobaditto`) |
| `REACTION` | Reacción cuando se aprueba, sin los dos puntos |
| `WARN_REACTION` | Reacción cuando alguna PR no se pudo aprobar. Por defecto `warning` |
| `TOKEN_CMD` | Comando que imprime el token de Slack. Por defecto lo lee del keyring |
| `TEAM` | `[ID de Slack]=login de GitHub` de cada persona. ID de Slack: su perfil → `⋮` → **Copiar ID de miembro** |

No te incluyas en `TEAM`: GitHub no deja aprobar tus propias PRs. Los emojis tienen que existir en
el workspace; si `REACTION` no existe, la aprobación se hace igual pero la reacción falla y queda en
el log.

### 5. Instalar el script

```bash
mkdir -p ~/.local/bin && install -m 755 slack-pr-approver ~/.local/bin/
mkdir -p ~/.local/state/slack-pr-approver
```

### 6. Probar en seco

`DRY_RUN=1` mira las últimas 24h y solo escribe lo que haría: no aprueba, no reacciona y no guarda
estado.

```bash
DRY_RUN=1 ~/.local/bin/slack-pr-approver
```

Verás líneas como `#1234 (asked by alice-gh): WOULD APPROVE` o `skipped, state is MERGED`. Si nadie
ha pedido nada en las últimas 24h no sale nada, y eso también es correcto.

### 7. Programar el cron

```bash
crontab -e
```

y añade:

```cron
* * * * * $HOME/.local/bin/slack-pr-approver >> $HOME/.local/state/slack-pr-approver/log 2>&1
```

- El `PATH` de cron es mínimo (`/usr/bin:/bin`). Si `gh` o `jq` están en otra ruta (`which gh jq`),
  añade encima de la línea algo como `PATH=/usr/local/bin:/usr/bin:/bin`.
- El script lee el token del keyring de tu sesión, así que solo funciona mientras tienes la sesión
  iniciada y el keyring desbloqueado. Con el equipo apagado no hace nada, y al volver recupera como
  mucho las últimas 24h.
- La **primera ejecución** solo guarda la marca de tiempo (`first run: only messages from now on
  will count`): las peticiones anteriores no se procesan.

### Logs y problemas típicos

```bash
tail -f ~/.local/state/slack-pr-approver/log
```

| Mensaje en el log | Causa |
|---|---|
| `ERROR no config at ...` / `ERROR X is not set` | Falta la configuración del paso 4 |
| `ERROR TOKEN_CMD returned no Slack token` | Falta el token (paso 3) o el keyring está bloqueado |
| `ERROR slack conversations.history: not_in_channel` / `channel_not_found` | No eres miembro del canal o el ID está mal |
| `ERROR slack ...: missing_scope` | Falta un scope en la app: añádelo y reinstálala |
| `ERROR slack ...: token_expired` / `invalid_auth` | Rotación de token activada o token revocado: genera uno nuevo y repite el paso 3 |
| `ERROR slack reactions.add: invalid_name` | El emoji de `REACTION` o `WARN_REACTION` no existe en el workspace |
| `gh pr view failed` / `gh pr review failed` | `gh` sin sesión o fuera del `PATH` de cron |
| `skipped, author X is not in the team` | El autor de la PR no está en `TEAM` |

### Desinstalar

Borra la línea del `crontab -e` y luego:

```bash
rm -r ~/.local/bin/slack-pr-approver ~/.config/slack-pr-approver ~/.local/state/slack-pr-approver
secret-tool clear service slack-pr-approver
```

Por último, elimina la app en <https://api.slack.com/apps>.

## macOS (sin probar)

- La bash del sistema es la 3.2 y no sirve: `brew install bash jq gh` y pon
  `PATH=/opt/homebrew/bin:/usr/bin:/bin` encima de la línea del crontab.
- `secret-tool` no existe. Guarda el token con
  `security add-generic-password -s slack-pr-approver -a "$USER" -w` y pon en la configuración
  `TOKEN_CMD='security find-generic-password -s slack-pr-approver -w'`.
- Puede que cron necesite permiso de acceso al llavero o a disco: si el log dice que no hay token,
  revísalo en **Ajustes del sistema → Privacidad y seguridad**.
