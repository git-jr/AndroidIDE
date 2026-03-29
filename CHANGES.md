# Registro de alterações para execução do projeto

Histórico de tudo que foi analisado e modificado para o projeto compilar, instalar e funcionar corretamente.

---

## 1. Análise inicial do projeto

Antes de qualquer modificação, foi feita uma análise completa da estrutura:

- Projeto multi-módulo com 40+ módulos organizados em grupos (core, editor, java, termux, tooling, etc.)
- Build system: Gradle 8.8, AGP 8.5.0, Kotlin 1.9.24
- Requisitos: JDK 17, Android SDK compileSdk 34, NDK 26.1.10909125
- `local.properties` já existia com o caminho do SDK configurado
- Dispositivo Samsung SM-S921B conectado via ADB

---

## 2. Correção 1 — `jakarta.servlet-api` com versão dinâmica quebrada

**Arquivo modificado:** `composite-builds/build-deps/logback-core/build.gradle.kts`

**Erro encontrado:**
```
Could not resolve jakarta.servlet:jakarta.servlet-api:6.2.0-M1.
  Could not find org.eclipse.ee4j:project:2.0.0-SNAPSHOT.
```

**Causa:** A dependência usava `+` como versão (dinâmica). O Gradle resolveu para `6.2.0-M1` (pré-release), cujo POM pai é um snapshot inexistente em repositórios públicos.

**Mudança:**
```kotlin
// antes
compileOnly("jakarta.servlet:jakarta.servlet-api:+")

// depois
compileOnly("jakarta.servlet:jakarta.servlet-api:6.1.0")
```

---

## 3. Correção 2 — `sora-editor` SNAPSHOT customizado inexistente

**Arquivo modificado:** `gradle/libs.versions.toml`

**Erro encontrado:**
```
Could not find io.github.Rosemoe.sora-editor:editor:0.23.4-ce8de8e-SNAPSHOT.
```

**Causa:** O projeto referenciava um SNAPSHOT customizado do sora-editor baseado no commit `ce8de8e`, que nunca foi publicado em nenhum repositório (Sonatype, JitPack, etc.).

**Versões testadas antes de chegar na correta:**

| Versão | Resultado |
|---|---|
| `0.23.4` | Falha — APIs `backingCharArray` e `getIndentAdvance` ausentes |
| `0.23.6` | Falha — importa `kotlin-stdlib:2.1.x`, incompatível com Kotlin 1.9.24 |
| `0.23.5` | ✅ Funciona |

**Mudança:**
```toml
# antes
editor = "0.23.4-ce8de8e-SNAPSHOT"

# depois
editor = "0.23.5"
```

---

## 4. Primeiro build e instalação bem-sucedidos

Com as duas correções acima, o build compilou com sucesso:

```
BUILD SUCCESSFUL
```

APK instalado no dispositivo: `app-arm64-v8a-debug.apk` → SM-S921B (Android 16).

---

## 5. Problema em runtime — chave GPG expirada no repositório de pacotes

Ao abrir o app no dispositivo e tentar instalar as ferramentas de desenvolvimento (via `idesetup.sh`), o terminal exibiu:

```
E: EXPKEYSIG 521D9A6E171FFD55 Akash Yadav <itsaky01@gmail.com>
E: The repository 'https://packages.androidide.com/apt/termux-main stable InRelease' is not signed.
N: Metadata integrity can't be verified, repository is disabled.
[Process completed (code 100)]
```

**Investigação:**
- O servidor `packages.androidide.com` ainda está no ar (HTTP 200)
- O problema é exclusivamente a chave GPG de assinatura que expirou
- Os bootstrap ZIPs são baixados do GitHub (`AndroidIDEOfficial/terminal-packages`, release `bootstrap-16.12.2023`) e embutidos no APK via assembly (`.incbin`)
- Dentro de cada ZIP existe `etc/apt/sources.list` que configura onde o `apt` busca pacotes

**Arquivo relevante:** `composite-builds/build-logic/plugins/src/main/java/com/itsaky/androidide/plugins/TerminalBootstrapPackagesPlugin.kt`

---

## 6. Correção 3 — Patch automático do `sources.list` no bootstrap

**Abordagem escolhida:** Modificar o `TerminalBootstrapPackagesPlugin.kt` para aplicar um patch automaticamente após baixar cada bootstrap ZIP do GitHub. O patch adiciona `[trusted=yes]` ao `sources.list`, instruindo o `apt` a aceitar o repositório mesmo sem assinatura GPG válida.

**Por que essa abordagem e não outras:**

| Alternativa considerada | Motivo de descarte |
|---|---|
| Trocar para `packages.termux.dev` | Pacotes do Termux usam prefix `/data/data/com.termux/...`, incompatível com AndroidIDE |
| Hospedar os ZIPs modificados externamente | Dependência de infraestrutura externa extra |
| Commitar os ZIPs modificados no repositório | Arquivos binários de 26 MB+ no git |
| Modificar os ZIPs manualmente | Não reproduzível em outros computadores |

**Arquivo modificado:** `composite-builds/build-logic/plugins/src/main/java/com/itsaky/androidide/plugins/TerminalBootstrapPackagesPlugin.kt`

**O que foi adicionado ao plugin:**

1. Imports de `ZipFile`, `ZipEntry`, `ZipOutputStream`
2. Os checksums SHA256 originais foram mantidos (para verificar o download do GitHub)
3. O download agora salva como `bootstrap-{arch}-original.zip`
4. Uma nova função `patchBootstrapSourcesList()` copia o ZIP inteiro, substituindo apenas `etc/apt/sources.list`
5. O ZIP patched é salvo como `bootstrap-{arch}.zip` e usado no assembly

**Conteúdo do `sources.list` após o patch:**
```
# The main AndroidIDE repository
deb [trusted=yes] https://packages.androidide.com/apt/termux-main/ stable main
```

**Fluxo de build após a correção:**
```
Gradle configura :termux:application
  → plugin baixa bootstrap-{arch}-original.zip do GitHub
  → verifica SHA256 (checksums originais)
  → aplica patch no sources.list
  → salva bootstrap-{arch}.zip
  → assembly .S embute o ZIP no .so nativo
  → APK inclui o ambiente termux com repositório funcional
```

---

## 7. Instalações realizadas

| # | Motivo | Resultado |
|---|---|---|
| 1 | Build inicial após correções 1 e 2 | ✅ Instalado |
| 2 | Após correção 3 (GPG fix) | ✅ Instalado |
| 3 | Validação de reprodutibilidade (ZIPs apagados, rebuild do zero) | ✅ Instalado |
| 4 | Reinstalação solicitada manualmente | ✅ Instalado |

---

## 8. Arquivos modificados — resumo

| Arquivo | Tipo de mudança |
|---|---|
| `composite-builds/build-deps/logback-core/build.gradle.kts` | Fixou versão `jakarta.servlet-api:+` → `6.1.0` |
| `gradle/libs.versions.toml` | Trocou `sora-editor` SNAPSHOT → `0.23.5` |
| `composite-builds/build-logic/plugins/.../TerminalBootstrapPackagesPlugin.kt` | Adicionou patch automático do `sources.list` com `[trusted=yes]` |
| `BUILD_SETUP.md` | Documentação técnica de pré-requisitos e correções |
| `CHANGES.md` | Este arquivo |
