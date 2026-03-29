# Guia: Adicionar Suporte a Novas Versões de Android

Este documento descreve o processo completo para adicionar suporte a um novo Android API level no AndroidIDE. Foi usado pela primeira vez para adicionar Android 15 (API 35) e Android 16 (API 36).

---

## Visão Geral da Arquitetura

O suporte a uma nova versão do Android envolve dois repositórios:

```
git-jr/androidide-tools          AndroidIDE (este repo)
────────────────────────         ────────────────────────────────────
manifest.json                    Sdk.kt          → opções na UI de novo projeto
scripts/repackage.sh             ideSetupConfig.kt → versões disponíveis no setup
scripts/idesetup                 idesetup.sh       → URL do manifest + default SDK
```

- **`git-jr/androidide-tools`**: hospeda os binários reempacotados e o manifest com as URLs de download.
- **AndroidIDE**: consome o manifest durante o onboarding e expõe as opções de API level na criação de projetos.

---

## Parte 1 — Binários (repo `git-jr/androidide-tools`)

### Fonte dos binários

O repositório [lzhiyong/android-sdk-tools](https://github.com/lzhiyong/android-sdk-tools/releases) publica builds cross-compilados das Android SDK build tools para ARM. Verifique lá se a versão desejada já está disponível.

### Passo 1 — Reempacotar os binários

```bash
cd /Users/junior/Dev/AndroidStudio/Androidide-tools/scripts

# Substitua X.Y.Z pela versão desejada (ex: 36.0.0)
./repackage.sh X.Y.Z aarch64
./repackage.sh X.Y.Z arm
./repackage.sh X.Y.Z x86_64
```

O script baixa o `.zip` do Lzhiyong, separa `build-tools` e `platform-tools` e gera 6 arquivos `.tar.xz`.

### Passo 2 — Criar release no GitHub

```bash
cd /Users/junior/Dev/AndroidStudio/Androidide-tools/scripts

gh release create vX.Y.Z \
  build-tools-X.Y.Z-aarch64.tar.xz \
  build-tools-X.Y.Z-arm.tar.xz \
  build-tools-X.Y.Z-x86_64.tar.xz \
  platform-tools-X.Y.Z-aarch64.tar.xz \
  platform-tools-X.Y.Z-arm.tar.xz \
  platform-tools-X.Y.Z-x86_64.tar.xz \
  --repo git-jr/androidide-tools \
  --title "vX.Y.Z" \
  --notes "Android SDK build tools and platform tools X.Y.Z (aarch64, arm, x86_64)."
```

### Passo 3 — Atualizar `manifest.json`

Adicionar entradas `_X_Y_Z` para cada arquitetura em `build_tools` e `platform_tools`:

```json
"build_tools": {
    "x86_64": {
        "_X_Y_Z": "https://github.com/git-jr/androidide-tools/releases/download/vX.Y.Z/build-tools-X.Y.Z-x86_64.tar.xz",
        ...
    },
    "aarch64": {
        "_X_Y_Z": "https://github.com/git-jr/androidide-tools/releases/download/vX.Y.Z/build-tools-X.Y.Z-aarch64.tar.xz",
        ...
    },
    "arm": {
        "_X_Y_Z": "https://github.com/git-jr/androidide-tools/releases/download/vX.Y.Z/build-tools-X.Y.Z-arm.tar.xz",
        ...
    }
},
"platform_tools": {
    ... (mesma estrutura)
}
```

> **Nota sobre `ARM_ONLY` vs `ALL`:** versões mais antigas (33.x, 34.0.0–34.0.3) só têm aarch64 e arm. Versões a partir de 34.0.4 têm também x86_64. Reflita isso na chave `availability` em `ideSetupConfig.kt`.

### Passo 4 — Atualizar `scripts/idesetup` do fork

```bash
sdkver_org=X.Y.Z   # nova versão padrão
```

### Passo 5 — Commit e push

```bash
cd /Users/junior/Dev/AndroidStudio/Androidide-tools
git add manifest.json scripts/idesetup
git commit -m "add: support for build-tools X.Y.Z"
git push
```

---

## Parte 2 — AndroidIDE (este repo)

### Arquivo 1 — `Sdk.kt`
**Caminho:** `utilities/templates-api/src/main/java/com/itsaky/androidide/templates/Sdk.kt`

Adicionar nova entrada no enum com o codinome, número da versão e API level:

```kotlin
  NomeDoAndroid("CodinomeAndroid", "XX", API_LEVEL),
```

Codinomes oficiais:
| Versão | Codinome | API |
|---|---|---|
| Android 13 | Tiramisu | 33 |
| Android 14 | UpsideDownCake | 34 |
| Android 15 | VanillaIceCream | 35 |
| Android 16 | Baklava | 36 |

### Arquivo 2 — `ideSetupConfig.kt`
**Caminho:** `core/app/src/main/java/com/itsaky/androidide/fragments/onboarding/ideSetupConfig.kt`

Adicionar a versão das build tools correspondente:

```kotlin
SDK_X_Y_Z("X.Y.Z", ALL),  // ou ARM_ONLY se x86_64 não estiver disponível
```

### Arquivo 3 — `idesetup.sh`
**Caminho:** `termux/application/src/main/assets/data/common/idesetup.sh`

Atualizar o default da versão do SDK:

```bash
sdkver_org=X.Y.Z
```

### Commit no AndroidIDE

```bash
git add \
  utilities/templates-api/src/main/java/com/itsaky/androidide/templates/Sdk.kt \
  core/app/src/main/java/com/itsaky/androidide/fragments/onboarding/ideSetupConfig.kt \
  termux/application/src/main/assets/data/common/idesetup.sh
git commit -m "feat: add support for Android XX (API YY)"
```

---

## Status atual

| Versão Android | API | Codinome | Build tools | Status |
|---|---|---|---|---|
| Android 13 | 33 | Tiramisu | 33.0.1, 33.0.3 | Suportado |
| Android 14 | 34 | UpsideDownCake | 34.0.0–34.0.4 | Suportado |
| Android 15 | 35 | VanillaIceCream | 35.0.2 | **Adicionado** |
| Android 16 | 36 | Baklava | — | Enum adicionado, aguardando binários |

---

## Referências

| Recurso | URL |
|---|---|
| Fork androidide-tools | https://github.com/git-jr/androidide-tools |
| Binários cross-compilados | https://github.com/lzhiyong/android-sdk-tools/releases |
| Manifest do fork | https://raw.githubusercontent.com/git-jr/androidide-tools/main/manifest.json |
