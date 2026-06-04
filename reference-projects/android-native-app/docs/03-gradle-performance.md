# 03 — Gradle Performance (MacBook Pro M4 Pro 48GB)

El template de Android Studio trae `gradle.properties` mínimo (`-Xmx2048m`, parallel
comentado). En M4 Pro 48GB se desaprovecha. Configuración recomendada.

## gradle.properties recomendado

```properties
# JVM — M4 Pro tiene 48GB; 6GB al daemon es seguro y acelera builds grandes
org.gradle.jvmargs=-Xmx6g -XX:+UseParallelGC -Dfile.encoding=UTF-8

# Paralelismo — M4 Pro = 14 cores
org.gradle.parallel=true

# Caches
org.gradle.caching=true
org.gradle.configuration-cache=true

# Kotlin incremental
kotlin.incremental=true
kotlin.code.style=official

# AndroidX
android.useAndroidX=true

# Build features off por defecto (encender solo lo que se usa)
android.defaults.buildfeatures.buildconfig=false
android.defaults.buildfeatures.resvalues=false
```

> **configuration-cache** es la mayor ganancia en incremental builds. Si algún plugin no
> es compatible, Gradle avisa; en ese caso usar `--configuration-cache-problems=warn`
> temporalmente o desactivarlo hasta que el plugin se actualice.

## SSD externo — consideración de I/O

El proyecto + SDK viven en your external SSD. El build cache de Gradle también queda en el SSD.
Implicaciones:

- ✅ Mantener el SSD conectado por un puerto rápido (TB/USB-C 10Gbps+)
- ⚠️ Si el SSD es más lento que el disco interno, el build cache compartido sufre. Medir:
  ```bash
  ./gradlew assembleDebug --profile   # genera report HTML con tiempos
  ```
- Opción: build cache en disco interno aunque el proyecto esté en SSD:
  ```properties
  # gradle.properties
  org.gradle.caching=true
  ```
  ```kotlin
  // settings.gradle.kts — buildCache local en disco interno
  buildCache {
      local { directory = file("${System.getProperty("user.home")}/.gradle-build-cache") }
  }
  ```

## Modularización (cuando el build time lo justifique)

Single-module es más rápido al inicio (menos overhead de configuración). Modularizar cuando:
- Build incremental > 30s consistentemente
- Equipo > 2-3 devs con conflictos de merge en `:app`
- Se quiere paralelizar builds por feature

Beneficio de módulos: Gradle paraleliza y cachea por módulo. Un cambio en `feature-feed`
no recompila `feature-chat`.

## Medición

```bash
# Profile de un build (genera report HTML)
./gradlew assembleDebug --profile
# → build/reports/profile/profile-<timestamp>.html

# Build scan (detallado, sube a gradle.com)
./gradlew assembleDebug --scan

# Ver qué tasks corren
./gradlew assembleDebug --dry-run
```

## Targets de referencia (M4 Pro, single-module)

| Build | Objetivo |
|---|---|
| Clean build | < 60s |
| Incremental (1 archivo Kotlin) | < 10s |
| Incremental Compose (1 Composable) | < 8s con Live Edit |
| Unit test suite | < 15s |
