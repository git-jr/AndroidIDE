# Build Setup — AndroidIDE

Guia para compilar e instalar o projeto AndroidIDE localmente a partir do zero.

---

## Pré-requisitos

| Ferramenta | Versão mínima | Observação |
|---|---|---|
| JDK | 17 | JDK 11 funciona, mas 17 é preferido |
| Android SDK | compileSdk 34 | Configurado via `local.properties` |
| Android NDK | 26.1.10909125 | Necessário para compilar código nativo |
| ADB | qualquer | Para instalar no dispositivo |
| Dispositivo/Emulador | Android 8.0+ (API 26) | minSdk é 26 |

### Configurar `local.properties`

Se o arquivo não existir na raiz do projeto, crie-o:

```properties
sdk.dir=/caminho/para/seu/Android/sdk
```

No macOS o caminho padrão é `~/Library/Android/sdk`.

---

## Como buildar e instalar

```bash
# compilar APK debug
./gradlew :core:app:assembleDebug

# compilar e instalar direto no dispositivo conectado
./gradlew :core:app:installDebug
```

O APK gerado é `core/app/build/outputs/apk/debug/app-arm64-v8a-debug.apk`.

---

## Problemas conhecidos e correções aplicadas

O repositório original contém três problemas que impedem o build. Todas as correções já estão commitadas — **não é necessário nenhum passo manual**.

---

### 1. `jakarta.servlet-api` — versão dinâmica resolve para snapshot quebrado

**Arquivo:** `composite-builds/build-deps/logback-core/build.gradle.kts`

**Problema:** A versão `+` (dinâmica) resolvia para `6.2.0-M1` (pré-release), que declara `org.eclipse.ee4j:project:2.0.0-SNAPSHOT` como parent POM. Esse artefato não existe em nenhum repositório público:

```
> Could not find org.eclipse.ee4j:project:2.0.0-SNAPSHOT.
```

**Correção:**

```kotlin
// antes
compileOnly("jakarta.servlet:jakarta.servlet-api:+")

// depois
compileOnly("jakarta.servlet:jakarta.servlet-api:6.1.0")
```

---

### 2. `sora-editor` — versão SNAPSHOT customizada não publicada

**Arquivo:** `gradle/libs.versions.toml`

**Problema:** A versão `0.23.4-ce8de8e-SNAPSHOT` é um build customizado baseado em um commit específico que nunca foi publicado nos repositórios configurados:

```
> Could not find io.github.Rosemoe.sora-editor:editor:0.23.4-ce8de8e-SNAPSHOT.
```

**Versões testadas:**

| Versão | Resultado |
|---|---|
| `0.23.4` | Falha de compilação — APIs adicionadas no SNAPSHOT ausentes |
| `0.23.5` | ✅ Compatível |
| `0.23.6` | Falha — puxa `kotlin-stdlib:2.1.x`, incompatível com Kotlin 1.9.24 do projeto |

**Correção:**

```toml
# antes
editor = "0.23.4-ce8de8e-SNAPSHOT"

# depois
editor = "0.23.5"
```

---

### 3. Chave GPG expirada no repositório de pacotes do terminal

**Arquivo:** `composite-builds/build-logic/plugins/src/main/java/com/itsaky/androidide/plugins/TerminalBootstrapPackagesPlugin.kt`

**Contexto:** O app embute bootstrap packages do Termux (arquivos `.zip`) que configuram o ambiente de terminal interno. Esses ZIPs contêm um `etc/apt/sources.list` apontando para `packages.androidide.com`. Quando o usuário abre o app e instala as ferramentas de desenvolvimento (`idesetup.sh`), o `pkg update` falha porque a chave GPG de assinatura do repositório expirou:

```
E: EXPKEYSIG 521D9A6E171FFD55 Akash Yadav <itsaky01@gmail.com>
E: The repository '...' is not signed.
N: Metadata integrity can't be verified, repository is disabled.
```

**Observação importante:** O servidor `packages.androidide.com` **ainda está no ar** — o problema é exclusivamente a chave GPG expirada, não a indisponibilidade do servidor.

**Correção:** O `TerminalBootstrapPackagesPlugin.kt` foi modificado para, após baixar cada bootstrap ZIP do GitHub (verificando o SHA256 original), aplicar automaticamente um patch que adiciona `[trusted=yes]` ao `sources.list`:

```
# antes
deb https://packages.androidide.com/apt/termux-main/ stable main

# depois
deb [trusted=yes] https://packages.androidide.com/apt/termux-main/ stable main
```

Isso instrui o `apt` a aceitar o repositório mesmo sem assinatura GPG válida.

O processo acontece automaticamente a cada build:
1. Gradle baixa `bootstrap-{arch}-original.zip` do GitHub e verifica o SHA256
2. Aplica o patch no `sources.list` e salva como `bootstrap-{arch}.zip`
3. O ZIP patched é embutido no APK via assembly (`.incbin`)

Nenhum passo manual é necessário — commitar as alterações no `TerminalBootstrapPackagesPlugin.kt` é suficiente para que qualquer clone do repositório reproduza o processo.

---

## Observações gerais

- As mensagens `Signing key not found. Debug signing will be used.` são esperadas em builds locais sem keystore configurada.
- O aviso `Deprecated Gradle features` é informativo e não impede o build.
- O projeto está marcado como **não mantido ativamente** pelo time original. Ajustes de dependências podem ser necessários conforme os repositórios Maven evoluem.
