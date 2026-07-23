# User Story: Centrale mise-taken, tooling en linterconfiguratie

**Als** developer **wil ik** gedeelde mise-taken, tool-definities en linterconfiguratie vanuit een centraal repository gebruiken **zodat** ik niet in elk project dezelfde scripts, tools en configuratie hoef te onderhouden.

## Achtergrondinformatie

We beheren meerdere repositories die dezelfde mise-taken (`build:image`, `start:image`, lint) en linterconfiguratie gebruiken. Dit wordt nu per repo gekopieerd, wat leidt tot duplicatie en inconsistentie.

Mise ondersteunt remote git-includes via `git::` URL-syntax in `task_config.includes`. Mise kloont het repository naar een lokale cache en maakt scripts automatisch executable. Via `ref` wordt gepind op een tag of commit.

Elke gedeelde taak vereist specifieke tooling (ruff, shellcheck, etc.). De `#MISE tools={}` header in file tasks werkt niet met de HTTP-backend, die wij nodig hebben voor onze air-gapped omgeving met zelf-gehoste binaries. **Tool stubs** bieden hier een oplossing: executable bestanden die tool + versie + download-URL bevatten en bij uitvoering automatisch installeren.

**Let op:** remote git-includes zijn momenteel nog **experimenteel** in mise.

## Oplossingsrichting

1. **Centraal repository** (`myorg/mise-shared-tasks`) met taken, tool stubs en linterconfiguratie, getagd met semver.
2. **`task_config.includes`** in elk applicatierepo met `git::` URL, gepind op een tag.
3. Lokale taakdirectory (`.mise/tasks`) als laatste entry in `includes` — lokale taken overschrijven gedeelde.
4. **Tool stubs** in `bin/` in het centrale repo definiëren tool + versie + download-URL (HTTP-backend). Taskscripts gebruiken een hybride aanpak: als de tool al op PATH staat (via app repo `[tools]`), gebruik die; anders fallback naar de shared tool stub.
5. **Linterconfiguratie** leeft in `config/` naast `tasks/`. Taskscripts resolven config via `$(dirname "$0")/../config` en `--config` flags. Heeft een project een eigen configbestand in de root, dan wint dat.

### Voorbeeld

Centraal repository:
```
myorg/mise-shared-tasks/
├── tasks/
│   ├── lint              # bash script met tool- en config-resolutie
│   ├── build/
│   │   └── image
│   └── start/
│       └── image
├── bin/
│   ├── ruff              # tool stub (HTTP backend)
│   └── shellcheck        # tool stub (HTTP backend)
└── config/
    ├── ruff.toml
    └── .shellcheckrc
```

Tool stub (`bin/ruff`):
```
#!/usr/bin/env -S mise tool-stub
version = "0.11.0"
bin = "ruff"
url = "https://internal-host/ruff-{{version}}-{{arch}}.tar.gz"
checksum = "blake3:a1b2c3..."
```

Lint-taskscript (vereenvoudigd):
```bash
#!/usr/bin/env bash
#MISE description="Lint Python code"
SHARED="$(dirname "$0")/.."

# Tool resolutie: PATH wint, anders shared stub
if command -v ruff &>/dev/null; then
    RUFF="ruff"
else
    RUFF="$SHARED/bin/ruff"
fi

# Config resolutie: lokaal wint, anders shared default
if [ -f "$MISE_PROJECT_DIR/ruff.toml" ]; then
    "$RUFF" check .
else
    "$RUFF" check --config "$SHARED/config/ruff.toml" .
fi
```

Applicatierepo `mise.toml`:
```toml
[task_config]
includes = [
    "git::ssh://git@github.com/myorg/mise-shared-tasks.git//tasks?ref=v1.0.0",
    ".mise/tasks",
]

# Optioneel: override tool versie per project
# Als dit niet opgegeven wordt, gebruikt het taskscript de shared tool stub
[tools]
ruff = "0.10.0"   # dit project gebruikt nog een oudere versie
```

## Acceptatiecriteria

- [ ] Centraal repository bevat taken, tool stubs en linterconfiguratie, getagd met semver.
- [ ] Applicatierepo's gebruiken `git::` includes met een gepinde tag.
- [ ] `mise tasks` toont zowel gedeelde als lokale taken.
- [ ] Tool stubs gebruiken HTTP-backend met interne binary mirrors.
- [ ] Taskscripts vallen terug op shared tool stubs wanneer tool niet op PATH staat.
- [ ] App repos kunnen tool versies overschrijven via `[tools]` in hun `mise.toml`.
- [ ] Lokale linterconfiguratie overschrijft gedeelde defaults.
- [ ] CI/CD pipelines kunnen gedeelde taken ophalen zonder extra configuratie.
- [ ] Versie-updates verlopen via tag-bump in `mise.toml` (commit + PR).
- [ ] `experimental = true` is ingeschakeld in mise-settings.
- [ ] Documentatie beschrijft de `includes`-gotcha (defaults worden vervangen, niet aangevuld).
