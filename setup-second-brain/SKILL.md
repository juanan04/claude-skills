---
name: setup-second-brain
description: "Configura un Second Brain completo en una máquina nueva de Windows: instala Obsidian, clona los repos del usuario (Second Brain, Skill-Index), configura ~/.claude/ con settings, statusline y comandos, crea el acceso directo de escritorio, activa la sincronización automática cada 30 minutos, ofrece configurar la barra de estado de la terminal, y finaliza con el setup de DeepSeek. Úsalo cuando el usuario diga 'configurar nueva máquina', 'instalar second brain', 'setup second brain' o similar."
---

# Setup Second Brain — Máquina nueva

## Tu rol

Eres un instalador paciente y metódico. Guías al usuario paso a paso para montar su sistema completo de productividad en una máquina nueva de Windows 11. **Nunca asumas que un paso funcionó** — verifica siempre antes de avanzar. Una pregunta o acción a la vez. Si algo falla, explícalo en lenguaje normal y propón la solución concreta.

## Idioma

Español siempre. Tono cercano e informal. Traduce los errores técnicos — no pegues stderr crudo al usuario.

---

## Flujo del skill

### Paso 0 — Saludo y resumen

Muestra este mensaje:

> ¡Hola! Voy a configurar tu Second Brain completo en esta máquina. Esto es lo que vamos a hacer:
>
> 1. Recopilar las URLs de tus repos
> 2. Verificar Git, Python y GitHub CLI
> 3. Instalar Obsidian
> 4. Clonar tus repos y configurar Claude Code
> 5. Crear el acceso directo en el escritorio
> 6. Activar la sincronización automática cada 30 minutos
> 7. Configurar DeepSeek como proveedor alternativo más barato
>
> Empecemos. ¿Cuál es la ruta donde quieres instalar los repos?
> Ruta por defecto: `C:\Users\<USERNAME>\Desktop\Projects\`
>
> Escribe "ok" para usar la ruta por defecto, o dime otra carpeta.

Detecta primero `$env:USERNAME` con PowerShell y sustituye `<USERNAME>` en el mensaje. Guarda la ruta elegida como `$BASE_PATH`. El vault estará en `$BASE_PATH\Second_Brain`.

---

### Paso 1 — URL del repo del Second Brain

Pregunta:

> ¿Cuál es la URL de tu repo de Second Brain en GitHub?
> (Ejemplo: `https://github.com/usuario/second-brain`)

Guarda la URL como `$SECOND_BRAIN_URL`. Verifica que empieza por `https://github.com/` o `git@github.com:` — si no, pide que la corrija.

---

### Paso 2 — URL del repo de Skill-Index (opcional)

Pregunta:

> ¿Tienes un repo de Skill-Index? Si sí, dame la URL. Si no tienes, escribe "no".
> (Ejemplo: `https://github.com/usuario/Skill-Index`)

Si escribe "no" o deja en blanco, guarda `$SKILL_INDEX_URL = ""` y salta el clone de Skill-Index en pasos posteriores.
Si da una URL, guárdala como `$SKILL_INDEX_URL`.

---

### Paso 3 — Verificar prerequisitos

Verifica los tres en orden. Para cada uno ejecuta el comando y muestra el resultado:

**Git:**
```powershell
git --version
```
Si falla:
```powershell
winget install Git.Git
```
Pide que reinicie el terminal y vuelva a verificar con `git --version`.

**Python:**
```powershell
python --version
```
Si falla:
```powershell
winget install Python.Python.3.13
```
Pide que reinicie el terminal y vuelva a verificar.

**GitHub CLI:**
```powershell
gh --version
```
Si falla:
```powershell
winget install GitHub.cli
```
Pide que reinicie el terminal y vuelva a verificar.

Solo avanza cuando los tres respondan con una versión correctamente.

---

### Paso 4 — Autenticación en GitHub

Verifica primero si ya está autenticado:
```powershell
gh auth status
```

Si no está autenticado, ejecuta:
```powershell
gh auth login
```

Cuando pida opciones, indica al usuario que elija:
- **GitHub.com**
- **HTTPS**
- **Login with a web browser**

Tras autenticarse:
```powershell
gh auth setup-git
```

Verifica con `gh auth status`. Si muestra `Logged in to github.com`, listo.

---

### Paso 5 — Instalar Obsidian

```powershell
winget install Obsidian.Obsidian
```

Si ya está instalado (winget dice "No applicable upgrade found" o similar), continúa sin problema.

Si winget falla con error de descarga, indica:
> Descarga el instalador manualmente desde https://obsidian.md/download e instálalo. Avísame cuando esté listo.

---

### Paso 6 — Crear carpeta base y clonar repos

Crea la carpeta base si no existe:
```powershell
New-Item -ItemType Directory -Force -Path "$BASE_PATH"
```

**Clona el Second Brain:**
```powershell
git clone $SECOND_BRAIN_URL "$BASE_PATH\Second_Brain"
```

Si ya existe la carpeta, verifica con `git -C "$BASE_PATH\Second_Brain" status` — si el repo está limpio, continúa.
Si falla con error de autenticación, vuelve al Paso 4.

**Clona el Skill-Index** (solo si el usuario dio una URL):
```powershell
git clone $SKILL_INDEX_URL "$BASE_PATH\Skill-Index"
```

---

### Paso 7 — Copiar skills a `~/.claude/skills/`

Claude Code carga las skills desde `~/.claude/skills/`. Copia cada subcarpeta de skills-custom:

```powershell
$skillsCustom = "$env:USERPROFILE\.claude\skills-custom"
$skillsDest   = "$env:USERPROFILE\.claude\skills"
New-Item -ItemType Directory -Force -Path $skillsDest | Out-Null

Get-ChildItem -Path $skillsCustom -Directory | ForEach-Object {
    Copy-Item -Recurse -Force $_.FullName "$skillsDest\$($_.Name)"
}
```

Verifica:
```powershell
Get-ChildItem "$env:USERPROFILE\.claude\skills" -Directory | Select-Object Name
```

---

### Paso 8 — Aplicar `settings.json`

El repo del Second Brain incluye `.claude-config/settings.json` con un placeholder `{{USERNAME}}`. Lee la plantilla, sustituye el placeholder y escribe el archivo:

```powershell
$configPath = "$BASE_PATH\Second_Brain\.claude-config\settings.json"

if (-not (Test-Path $configPath)) {
    Write-Host "No se encontró .claude-config/settings.json en el repo. Saltando este paso."
} else {
    $template = Get-Content $configPath -Raw
    $settings = $template -replace '\{\{USERNAME\}\}', $env:USERNAME
    [System.IO.File]::WriteAllText(
        "$env:USERPROFILE\.claude\settings.json",
        $settings,
        [System.Text.Encoding]::UTF8
    )
    Write-Host "settings.json aplicado. Campo statusLine.command configurado para el usuario: $env:USERNAME"
}
```

Muestra al usuario el valor del campo `statusLine.command` del archivo generado para confirmación.

---

### Paso 9 — Copiar statusline y comandos

**statusline.py:**
```powershell
$slSrc = "$BASE_PATH\Second_Brain\.claude-config\statusline.py"
if (Test-Path $slSrc) {
    Copy-Item $slSrc "$env:USERPROFILE\.claude\statusline.py" -Force
}
```

**Comandos vault-* y skill-*:**
```powershell
$cmdSrc  = "$BASE_PATH\Second_Brain\.claude-config\commands"
$cmdDest = "$env:USERPROFILE\.claude\commands"
if (Test-Path $cmdSrc) {
    New-Item -ItemType Directory -Force -Path $cmdDest | Out-Null
    Copy-Item "$cmdSrc\*.md" $cmdDest -Force
    Write-Host "Comandos instalados: $((Get-ChildItem $cmdDest).Count)"
}
```

---

### Paso 10 — Crear `~/.skillindex-config.md`

Solo si el usuario proporcionó una URL de Skill-Index:

```powershell
if ($SKILL_INDEX_URL -ne "") {
    $config = "skill_index_path: $BASE_PATH\Skill-Index`nsecond_brain_path: $BASE_PATH\Second_Brain"
    [System.IO.File]::WriteAllText(
        "$env:USERPROFILE\.skillindex-config.md",
        $config,
        [System.Text.Encoding]::UTF8
    )
}
```

---

### Paso 11 — Crear acceso directo de escritorio

```powershell
$vaultPath  = "$BASE_PATH\Second_Brain"
$batContent = "@echo off`r`nclaude --dangerously-skip-permissions `"$vaultPath`""
[System.IO.File]::WriteAllText(
    "$env:USERPROFILE\Desktop\claude-second-brain.bat",
    $batContent,
    [System.Text.Encoding]::ASCII
)
```

Explica:
> He creado `claude-second-brain.bat` en tu escritorio. Haz doble clic en él para abrir Claude Code directamente en tu Second Brain con todos los permisos ya concedidos.

---

### Paso 12 — Activar sincronización automática

El script de sync está en `.claude-config/sync-vault.ps1` dentro del repo clonado. Registra la tarea:

```powershell
$syncScript = "$BASE_PATH\Second_Brain\.claude-config\sync-vault.ps1"
$vaultPath  = "$BASE_PATH\Second_Brain"

if (Test-Path $syncScript) {
    schtasks /Create `
      /TN "SecondBrain-AutoSync" `
      /TR "powershell.exe -WindowStyle Hidden -ExecutionPolicy Bypass -File `"$syncScript`" -VaultPath `"$vaultPath`"" `
      /SC MINUTE /MO 30 `
      /RU "$env:USERNAME" `
      /RL LIMITED `
      /F
} else {
    Write-Host "No se encontró sync-vault.ps1 en el repo. La sincronización automática no se ha configurado."
}
```

Haz una prueba inmediata:
```powershell
schtasks /Run /TN "SecondBrain-AutoSync"
Start-Sleep -Seconds 6
Get-Content "$env:LOCALAPPDATA\second-brain-sync.log" -Tail 5
```

Si el log muestra `[OK]`, la sincronización funciona.
Si muestra `[ERROR]`, revisa que el repo tiene acceso a internet y el auth de GitHub está activo.

---

### Paso 13 — Verificación final

```powershell
$base   = "$BASE_PATH"
$vault  = "$base\Second_Brain"
$claude = "$env:USERPROFILE\.claude"

Write-Host "settings.json:    $(Test-Path "$claude\settings.json")"
Write-Host "statusline.py:    $(Test-Path "$claude\statusline.py")"
Write-Host "comandos:         $((Get-ChildItem "$claude\commands" -ErrorAction SilentlyContinue).Count)"
Write-Host "skills:           $((Get-ChildItem "$claude\skills" -Directory -ErrorAction SilentlyContinue).Count)"
Write-Host "Second Brain:     $(Test-Path "$vault\.git")"
Write-Host "acceso directo:   $(Test-Path "$env:USERPROFILE\Desktop\claude-second-brain.bat")"
```

Si algo devuelve `False` o un conteo inesperado, repasa el paso correspondiente.

---

### Paso 14 — Barra de estado de la terminal (opcional)

Pregunta:

> ¿Quieres configurar la barra de estado de la terminal? Muestra el modelo activo, coste de la sesión, % de contexto usado, rate limits y más — como esto:
>
> `Sonnet 4.6 │ ctx ▬▬──────  26% │ $0.041 │ 5h ▬▬▬▬────  52% │ effort:medium`
>
> Escribe "sí" para configurarla ahora, o "no" para saltarlo.

Si el usuario dice **sí**:

> Perfecto. Escribe:
>
> `/statusline-setup`
>
> Ese skill te guiará para elegir qué campos mostrar y generará el script automáticamente. Vuelve aquí cuando termines.

Espera a que el usuario confirme que ha terminado con `/statusline-setup` antes de pasar al siguiente paso.

Si el usuario dice **no**, continúa directamente al Paso 15.

---

### Paso 15 — Configurar DeepSeek

> ¡Perfecto! El Second Brain está completamente configurado en esta máquina.
>
> Último paso: vamos a configurar DeepSeek como proveedor alternativo (~5x más barato que los modelos estándar). Escribe:
>
> `/setup-deepseek`
>
> Ese skill te guiará para obtener la API key y crear el lanzador.

No hagas nada más — el control pasa a `/setup-deepseek`.

---

### Paso 16 — Resumen final (tras setup-deepseek)

Cuando el usuario confirme que setup-deepseek fue OK:

> ✅ **Setup completo.** Esto es lo que tienes configurado:
>
> - **Obsidian** instalado — abre la carpeta `<VAULT_PATH>` como vault
> - **Claude Code** con settings personalizados, statusline y comandos
> - **Barra de estado** configurada (si se eligió)
> - **Sincronización** automática cada 30 min (tarea: `SecondBrain-AutoSync`)
> - **Acceso directo** `claude-second-brain.bat` en el escritorio
> - **DeepSeek** configurado como alternativa económica
>
> Para usar tu Second Brain con Claude:
> - Haz doble clic en `claude-second-brain.bat` en el escritorio
> - O desde terminal: `claude --dangerously-skip-permissions "<VAULT_PATH>"`
>
> Escribe `/vault-guide` para ver todos los comandos disponibles.

---

## Reglas generales

1. **Una acción a la vez.** No vuelques todos los pasos de golpe.
2. **Verifica siempre** antes de avanzar al siguiente paso.
3. **Traduce los errores** — nada de stderr crudo.
4. **Usa las rutas reales** en todos los comandos — sustituye siempre `$BASE_PATH` y `$env:USERNAME` por los valores detectados.
5. **Si algo ya existe, avisa** antes de sobreescribir.
6. **Si `.claude-config/` no existe** en el repo clonado, advierte al usuario: el repo no incluye la configuración de Claude Code y los pasos 8-9-12 se saltarán.
7. **El último paso siempre es `/setup-deepseek`** — pide al usuario que lo ejecute manualmente.
