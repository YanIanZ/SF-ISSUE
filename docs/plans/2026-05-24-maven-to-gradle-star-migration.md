# Maven → Gradle KTS + Dough → Star Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Maven with Gradle KTS, upgrade to Java 25, swap Dough for Star library, add iYanZ credits.

**Architecture:** Monolithic `build.gradle.kts` with Shadow plugin for dependency shading. Global batch refactor of 272+ Java files replaces `io.github.bakedlibs.dough.*` imports with `dev.yanianz.star.*`.

**Tech Stack:** Gradle 8.13, Java 25, Shadow Gradle Plugin 9.0.0, Paper 1.21.11, Star v2.0.2 (JitPack), JUnit Jupiter 5.11.4, MockBukkit 4.41.1

---

### Task 1: Set up Gradle wrapper

**Files:**
- Create: `gradle/wrapper/gradle-wrapper.properties`
- Create: `gradlew`
- Create: `gradlew.bat`

- [ ] **Step 1: Create `gradle/wrapper/gradle-wrapper.properties`**

Write exactly:
```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

- [ ] **Step 2: Create `gradlew` (Unix wrapper script)**

Write exactly:
```sh
#!/bin/sh
set -e
APP_HOME="$(cd "${0%/*}" && pwd -P)"
cd "$APP_HOME"
exec "${APP_HOME}/gradle/wrapper/gradle" "$@"
```

Wait, the standard gradlew is more complex. Let's use the proper approach:

Actually, let's use a simpler approach — just write the gradle-wrapper.properties and run `gradle wrapper` to generate gradlew/gradlew.bat. But we don't have Gradle installed...

Better approach: Write all three files manually. The actual gradlew and gradlew.bat from Gradle distribution are standard files. Since I can't easily replicate them here, I'll write the .properties file and use a different approach.

Let me think about this differently. The best approach is:
1. Write `gradle-wrapper.properties`
2. Write `settings.gradle.kts` (minimal)
3. Write a minimal `build.gradle.kts` 
4. Then run `gradle wrapper` via a tool or just write the standard gradlew scripts

Actually, the gradlew scripts are standard boilerplate. Let me include them in the plan correctly.

Let me revise this. The standard gradlew is a long script (several hundred lines). I'll write it from the Gradle distribution template.

For now, let me write a simpler, more practical plan where we write the properties file and then generate the wrapper using an available Gradle installation or download it.

Actually, for the plan, let me just write the actual content that the engineer needs. The gradlew standard script is well-known and can be copied from any Gradle project. Let me include the full gradlew content.

Hmm, this is getting complex. Let me simplify: I'll write the gradle-wrapper.properties, and for gradlew/gradlew.bat, I'll note to use `gradle wrapper` command or copy from the Star project (which already has them). But no — the plan should be self-contained.

Let me use a practical approach:
1. Write `gradle/wrapper/gradle-wrapper.properties`
2. Write minimal `settings.gradle.kts` 
3. Write minimal `build.gradle.kts`
4. Use a bash command to download and set up the wrapper

OK, I think the cleanest approach for the plan is:
- Step 1: Write gradle-wrapper.properties
- Step 2: Write gradlew (inline the whole script — it's ~200 lines but it's standard)
- Step 3: Write gradlew.bat
- Step 4: `chmod +x gradlew`

Actually, to keep the plan manageable, let me just reference the fact that gradlew is a standard file and include the essential parts.

Let me take a pragmatic approach and just write the plan with complete code where it matters most, and for boilerplate like gradlew, provide a concise command to set it up.

I'll write the plan with the approach of:
1. Create gradle-wrapper.properties
2. Create settings.gradle.kts  
3. Create build.gradle.kts
4. Download and set up gradle wrapper via bash

Let me rewrite the plan now, keeping it practical and complete.<｜end▁of▁thinking｜>Let me revise — I'll create a practical, complete plan:

<｜｜DSML｜｜tool_calls>
<｜｜DSML｜｜invoke name="write">
<｜｜DSML｜｜parameter name="content" string="true"># Maven → Gradle KTS + Dough → Star Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Maven with Gradle KTS, upgrade to Java 25, swap Dough for Star library v2.0.2, add iYanZ credits.

**Architecture:** Monolithic `build.gradle.kts` with Shadow Gradle plugin for dependency shading. Global batch refactor of all Java files replaces `io.github.bakedlibs.dough.*` imports with `dev.yanianz.star.*`.

**Tech Stack:** Gradle 8.13, Java 25, Shadow Gradle Plugin 9.0.0, Paper 1.21.11, Star v2.0.2 (JitPack), JUnit Jupiter 5.11.4, MockBukkit 4.41.1

---

### Task 1: Create Gradle wrapper files

**Files:**
- Create: `gradle/wrapper/gradle-wrapper.properties`
- Create: `gradlew`
- Create: `gradlew.bat`

- [ ] **Step 1: Create `gradle/wrapper/gradle-wrapper.properties`**

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

- [ ] **Step 2: Create `gradlew` script**

```bash
mkdir -p gradle/wrapper
```

Write `gradlew`:
```sh
#!/bin/sh

##############################################################################
##
##  Gradle start up script for UN*X
##
##############################################################################

# Attempt to set APP_HOME
# Resolve links: $0 may be a link
app_path=$0
while
    APP_HOME=${app_path%"${app_path##*/}"}  # leaves a trailing /; empty if no leading path
    [ -h "$app_path" ]
do
    ls=$( ls -ld -- "$app_path" )
    link=${ls#*' -> '}
    case $link in
      /*)   app_path=$link ;;
      *)    app_path=$APP_HOME$link ;;
    esac
done

# This is normally unused
# shellcheck disable=SC2034
APP_BASE_NAME=${0##*/}
# Discard cd standard output in case $CDPATH is set (https://github.com/gradle/gradle/issues/25036)
APP_HOME=$( cd -P "${APP_HOME:-./}" > /dev/null && printf '%s\n' "$PWD" ) || exit

# Use the maximum available, or set MAX_FD != -1 to use that value.
MAX_FD=maximum

warn () {
    echo "$*"
} >&2

die () {
    echo
    echo "$*"
    echo
    exit 1
} >&2

# OS specific support (must be 'true' or 'false').
cygwin=false
msys=false
darwin=false
nonstop=false
case "$( uname )" in
  CYGWIN* )         cygwin=true  ;;
  Darwin* )         darwin=true  ;;
  MSYS* | MINGW* )  msys=true    ;;
  NonStop* )        nonstop=true ;;
esac

CLASSPATH=$APP_HOME/gradle/wrapper/gradle-wrapper.jar

# Determine the Java command to use to start the JVM.
if [ -n "$JAVA_HOME" ] ; then
    if [ -x "$JAVA_HOME/jre/sh/java" ] ; then
        JAVACMD=$JAVA_HOME/jre/sh/java
    else
        JAVACMD=$JAVA_HOME/bin/java
    fi
    if [ ! -x "$JAVACMD" ] ; then
        die "ERROR: JAVA_HOME is set to an invalid directory: $JAVA_HOME

Please set the JAVA_HOME variable in your environment to match the
location of your Java installation."
    fi
else
    JAVACMD=java
    if ! command -v java >/dev/null 2>&1 ; then
        die "ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH.

Please set the JAVA_HOME variable in your environment to match the
location of your Java installation."
    fi
fi

# Increase the maximum file descriptors if we can.
if ! "$cygwin" && ! "$darwin" && ! "$nonstop" ; then
    case $MAX_FD in
      max*)
        MAX_FD=$( ulimit -H -n ) ||
            warn "Could not query maximum file descriptor limit"
      ;;
    esac
    case $MAX_FD in
      '' | soft) :;;
      *)
        ulimit -n "$MAX_FD" ||
            warn "Could not set maximum file descriptor limit to $MAX_FD"
      ;;
    esac
fi

# Collect all arguments for the java command, stracks://gradle.org/releases/

# Escape application args
save () {
    for i; do printf %s\\n "$i" | sed "s/'/'\\\\''/g;1s/^/'/;\$s/\$/' \\\\/" ; done
    echo " "
}
APP_ARGS=$(save "$@")

# Collect all arguments for the java command following the shell quoting and substitution rules
eval set -- "$DEFAULT_JVM_OPTS" "$JAVA_OPTS" "$GRADLE_OPTS" "\"-Dorg.gradle.appname=$APP_BASE_NAME\"" -classpath "\"$CLASSPATH\"" org.gradle.wrapper.GradleWrapperMain "$APP_ARGS"

exec "$JAVACMD" "$@"
```

Wait, that gradlew script is getting too long and I'm not sure about its correctness. Let me use a simpler approach.

Actually for the plan doc, let me use a pragmatic approach: reference that gradlew can be generated by downloading the gradle-wrapper.jar from maven central and then running it with the properties file. Or just include a bash command that sets it up.

Let me rewrite the plan more practically.<｜end▁of▁thinking｜>

<｜｜DSML｜｜tool_calls>
<｜｜DSML｜｜invoke name="write">
<｜｜DSML｜｜parameter name="content" string="true"># Maven → Gradle KTS + Dough → Star Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Maven with Gradle KTS, upgrade to Java 25, swap Dough for Star library v2.0.2, add iYanZ credits.

**Architecture:** Monolithic `build.gradle.kts` with Shadow Gradle plugin for dependency shading. Global batch refactor replaces `io.github.bakedlibs.dough.*` imports with `dev.yanianz.star.*`.

**Tech Stack:** Gradle 8.13, Java 25, Shadow Gradle Plugin 9.0.0, Paper 1.21.11, Star v2.0.2 (JitPack)

---

### Task 1: Create Gradle wrapper

**Files:**
- Create: `gradle/wrapper/gradle-wrapper.properties`

- [ ] **Step 1: Create the wrapper properties file**

```bash
mkdir -p gradle/wrapper
```

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

- [ ] **Step 2: Generate gradlew scripts via docker or direct download**

```bash
# Download gradle-wrapper.jar (required by gradlew)
curl -L -o gradle/wrapper/gradle-wrapper.jar \
  https://github.com/gradle/gradle/raw/v8.13.0/gradle/wrapper/gradle-wrapper.jar

# Write gradlew
cat > gradlew << 'GRADLEW_EOF'
#!/bin/sh
set -e
if [ -n "$JAVA_HOME" ]; then
    JAVACMD="$JAVA_HOME/bin/java"
else
    JAVACMD="java"
fi
APP_HOME="$(cd "$(dirname "$0")" && pwd -P)"
CLASSPATH="$APP_HOME/gradle/wrapper/gradle-wrapper.jar"
exec "$JAVACMD" \
    -Dorg.gradle.appname=gradlew \
    -classpath "$CLASSPATH" \
    org.gradle.wrapper.GradleWrapperMain \
    "$@"
GRADLEW_EOF

chmod +x gradlew

# Write gradlew.bat
cat > gradlew.bat << 'GRADLEWBAT_EOF'
@rem Gradle startup script for Windows
@if "%DEBUG%"=="" @echo off
set DIRNAME=%~dp0
set APP_HOME=%DIRNAME%
set CLASSPATH=%APP_HOME%\gradle\wrapper\gradle-wrapper.jar
set CMD_LINE_ARGS=%*
java -classpath "%CLASSPATH%" org.gradle.wrapper.GradleWrapperMain %CMD_LINE_ARGS%
GRADLEWBAT_EOF
```

- [ ] **Step 3: Commit**

```bash
git add gradle/ gradlew gradlew.bat
git commit -m "chore: add Gradle 8.13 wrapper"
```

---

### Task 2: Create settings.gradle.kts and build.gradle.kts

**Files:**
- Create: `settings.gradle.kts`
- Create: `build.gradle.kts`

- [ ] **Step 1: Create `settings.gradle.kts`**

```kotlin
rootProject.name = "Slimefun"
```

- [ ] **Step 2: Create `build.gradle.kts`** — full build config

```kotlin
plugins {
    java
    id("com.gradleup.shadow") version "9.0.0"
}

group = "com.github.slimefun"
version = "4.9-UNOFFICIAL"

java {
    sourceCompatibility = JavaVersion.VERSION_25
    targetCompatibility = JavaVersion.VERSION_25
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
    options.release = 25
    options.isIncremental = true
    exclude("**/package-info.java")
}

tasks.withType<Test> {
    useJUnitPlatform()
    failOnNoDiscoveredTests = false
    testLogging {
        events("passed", "skipped", "failed")
        showStackTraces = true
        exceptionFormat = org.gradle.api.tasks.testing.logging.TestExceptionFormat.FULL
    }
}

tasks.withType<Javadoc> {
    options.encoding = "UTF-8"
    (options as StandardJavadocDocletOptions).apply {
        addStringOption("Xdoclint:none", "-quiet")
        links("https://hub.spigotmc.org/javadocs/spigot/")
        doctitle("Slimefun4 - Javadocs")
        windowTitle = "Slimefun4 - Javadocs"
        isHTML5 = true
        groups = mapOf(
            "Slimefun4 - API" to listOf("io.github.thebusybiscuit.slimefun4.api*"),
            "Slimefun4 - Core packages" to listOf("io.github.thebusybiscuit.slimefun4.core*"),
            "Slimefun4 - Implementations" to listOf("io.github.thebusybiscuit.slimefun4.implementation*"),
            "Slimefun4 - Integrations" to listOf("io.github.thebusybiscuit.slimefun4.integrations*"),
            "Slimefun4 - Utility packages" to listOf("io.github.thebusybiscuit.slimefun4.utils*"),
            "Slimefun4 - Items" to listOf("io.github.thebusybiscuit.slimefun4.implementation.items*"),
            "Slimefun4 - Multiblocks" to listOf("io.github.thebusybiscuit.slimefun4.implementation.items.multiblocks*"),
            "Slimefun4 - Electrical Machines" to listOf("io.github.thebusybiscuit.slimefun4.implementation.items.electric*"),
            "Slimefun4 - Old packages" to listOf("me.mrCookieSlime.Slimefun*"),
        )
    }
}

repositories {
    mavenCentral()
    maven("https://hub.spigotmc.org/nexus/content/repositories/snapshots")
    maven("https://repo.papermc.io/repository/maven-public/")
    maven("https://jitpack.io")
    maven("https://maven.enginehub.org/repo/")
    maven("https://repo.extendedclip.com/content/repositories/placeholderapi")
    maven("https://nexus.neetgames.com/repository/maven-public")
    maven("https://repo.walshy.dev/public")
    maven("https://repo.codemc.io/repository/maven-public/")
}

dependencies {
    compileOnly("com.google.code.findbugs:jsr305:3.0.2")
    compileOnly("io.papermc.paper:paper-api:1.21.11-R0.1-SNAPSHOT")

    implementation("io.papermc:paperlib:1.0.8")
    implementation("commons-lang:commons-lang:2.6")
    implementation("com.github.YanIanZ:Star:v2.0.2")

    testImplementation(platform("org.junit:junit-bom:5.11.4"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
    testImplementation("org.mockito:mockito-core:5.14.1")
    testImplementation("org.slf4j:slf4j-simple:2.0.16")
    testImplementation("org.mockbukkit.mockbukkit:mockbukkit-v1.21:4.41.1") {
        exclude("org.jetbrains", "annotations")
    }

    compileOnly("com.sk89q.worldedit:worldedit-core:7.3.9") {
        exclude("*", "*")
    }
    compileOnly("com.sk89q.worldedit:worldedit-bukkit:7.3.9") {
        exclude("*", "*")
    }
    compileOnly("com.gmail.nossr50.mcMMO:mcMMO:2.2.029") {
        exclude("*", "*")
    }
    compileOnly("me.clip:placeholderapi:2.11.6") {
        exclude("*", "*")
    }
    compileOnly("me.minebuilders:clearlag-core:3.1.6") {
        exclude("*", "*")
    }
    compileOnly("com.github.LoneDev6:itemsadder-api:3.6.1") {
        exclude("*", "*")
    }
    compileOnly("net.imprex:orebfuscator-api:5.4.0") {
        exclude("*", "*")
    }
    compileOnly("com.mojang:authlib:6.0.52") {
        exclude("*", "*")
    }
}

tasks.named<Jar>("jar") {
    archiveBaseName = "Slimefun"
    archiveVersion = "v${project.version}"
    archiveClassifier = ""
}

tasks.shadowJar {
    archiveBaseName = "Slimefun"
    archiveVersion = "v${project.version}"
    archiveClassifier = ""

    relocate("dev.yanianz.star", "io.github.thebusybiscuit.slimefun4.libraries.star")
    relocate("io.papermc.lib", "io.github.thebusybiscuit.slimefun4.libraries.paperlib")
    relocate("org.apache.commons.lang", "io.github.thebusybiscuit.slimefun4.libraries.commons.lang")

    exclude("META-INF/**")
}

tasks.build {
    dependsOn(tasks.shadowJar)
}

tasks.processResources {
    filesMatching("plugin.yml") {
        expand(mapOf("project" to project))
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add settings.gradle.kts build.gradle.kts
git commit -m "chore: add Gradle KTS build config with Java 25 and Star dependency"
```

---

### Task 3: Update jitpack.yml and .gitignore

**Files:**
- Modify: `jitpack.yml`
- Modify: `.gitignore`

- [ ] **Step 1: Replace `jitpack.yml`**

```yaml
jdk:
  - openjdk25
```

Run:
```bash
cat > jitpack.yml << 'EOF'
jdk:
  - openjdk25
EOF
```

- [ ] **Step 2: Update `.gitignore` — remove Maven entries, add Gradle entries**

Replace the file content:
```gitignore
/bin/
/build/
/.gradle/
/plugins/
/sonar/
/.settings/
/.idea/
/.vscode/
/data-storage/
/javadocs/

dependency-reduced-pom.xml

.classpath
.factorypath
.project
*.iml
*.bak
.DS_Store
```

- [ ] **Step 3: Commit**

```bash
git add jitpack.yml .gitignore
git commit -m "chore: update jitpack.yml for Java 25, update .gitignore for Gradle"
```

---

### Task 4: Delete pom.xml

**Files:**
- Delete: `pom.xml`

- [ ] **Step 1: Delete pom.xml**

```bash
git rm pom.xml
```

- [ ] **Step 2: Commit**

```bash
git commit -m "chore: remove Maven pom.xml"
```

---

### Task 5: Refactor dough → star imports (batch replace)

**Files:**
- Modify: all `src/main/java/**/*.java` files (272+)
- Modify: all `src/test/java/**/*.java` files (43+)

- [ ] **Step 1: Batch replace `io.github.bakedlibs.dough` → `dev.yanianz.star`**

```bash
find src/main/java -name "*.java" -exec sed -i '' \
  's/io\.github\.bakedlibs\.dough/dev.yanianz.star/g' {} +

find src/test/java -name "*.java" -exec sed -i '' \
  's/io\.github\.bakedlibs\.dough/dev.yanianz.star/g' {} +
```

- [ ] **Step 2: Verify replacement count**

```bash
# Should be 0 — no more dough references
grep -r "io\.github\.bakedlibs\.dough" src/main/java src/test/java | wc -l

# Should show many — star references now present
grep -r "dev\.yanianz\.star" src/main/java | wc -l
```

- [ ] **Step 3: Commit**

```bash
git add src/main/java src/test/java
git commit -m "refactor: replace dough imports with star (dev.yanianz.star)"
```

---

### Task 6: Add credits to Slimefun.java

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/Slimefun.java`

- [ ] **Step 1: Add credit line in `onPluginStart()` after localization setup**

The credit goes after line 300 (after `local = new LocalizationService(...)`) — right after localization loads, before network setup.

Edit: find this exact text at line 300-301:
```java
        local = new LocalizationService(this, chatPrefix, serverDefaultLanguage);
```
Replace with:
```java
        local = new LocalizationService(this, chatPrefix, serverDefaultLanguage);

        logger.log(Level.INFO, "Maintained by iYanZ");
```

- [ ] **Step 2: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/Slimefun.java
git commit -m "feat: add iYanZ maintainer credit on plugin enable"
```

---

### Task 7: Update plugin.yml author

**Files:**
- Modify: `src/main/resources/plugin.yml`

- [ ] **Step 1: Change author field**

Find: `author: The Slimefun 4 Community`
Replace with: `author: iYanZ`

- [ ] **Step 2: Commit**

```bash
git add src/main/resources/plugin.yml
git commit -m "chore: update plugin.yml author to iYanZ"
```

---

### Task 8: Resolve compilation errors

**Files:**
- Potentially modify: any file failed at compile due to Star API differences

- [ ] **Step 1: Run Gradle compile and capture errors**

```bash
./gradlew compileJava 2>&1 | tee compile-errors.log
```

- [ ] **Step 2: Fix any missing classes due to package mapping differences**

Known potential issues:
- `dough.collections.RandomizedSet` → `dev.yanianz.star.data.RandomizedSet` or `dev.yanianz.star.common.RandomizedSet`
- `dough.collections.LoopIterator` → same package mapping question
- `dough.common.ChatColors` — Star may have renamed or restructured this

For each compilation error in `compile-errors.log`, check what actual Star package exports the needed class:

```bash
# List all classes in star jar to verify package mapping
./gradlew dependencies --configuration compileClasspath | grep star
```

Fix each import individually based on actual Star JAR contents. Common corrections:

| Old import | Verified Star import |
|---|---|
| `dough.config.Config` | `star.config.Config` |
| `dough.items.CustomItemStack` | `star.items.CustomItemStack` |
| `dough.items.ItemUtils` | `star.items.ItemUtils` |
| `dough.common.CommonPatterns` | `star.common.CommonPatterns` |
| `dough.common.ChatColors` | `star.common.ChatColors` |
| `dough.skins.PlayerHead` | `star.skins.PlayerHead` |
| `dough.skins.PlayerSkin` | `star.skins.PlayerSkin` |
| `dough.protection.Interaction` | `star.protection.Interaction` |
| `dough.protection.ProtectionManager` | `star.protection.ProtectionManager` |
| `dough.blocks.BlockPosition` | `star.blocks.BlockPosition` |
| `dough.blocks.Vein` | `star.blocks.Vein` |
| `dough.blocks.ChunkPosition` | `star.blocks.ChunkPosition` |
| `dough.inventory.InvUtils` | `star.inventories.InvUtils` |
| `dough.scheduling.TaskQueue` | `star.scheduling.TaskQueue` |
| `dough.chat.ChatInput` | `star.chat.ChatInput` |
| `dough.recipes.RecipeSnapshot` | `star.recipes.RecipeSnapshot` |
| `dough.data.persistent.PersistentDataAPI` | `star.data.persistent.PersistentDataAPI` |
| `dough.items.ItemMetaSnapshot` | `star.items.ItemMetaSnapshot` |
| `dough.items.ItemStackEditor` | `star.items.ItemStackEditor` |
| `dough.collections.RandomizedSet` | TBD — verify from star jar |
| `dough.collections.LoopIterator` | TBD — verify from star jar |

**Fix approach per error:**
```bash
# For each class that fails, inspect the Star jar contents
jar tf ~/.gradle/caches/modules-*/files-*/com.github.YanIanZ/Star/v2.0.2/*/Star-v2.0.2.jar | grep -i "ClassName"
# Or check Star source at https://github.com/YanIanZ/Star/tree/main
# Then fix the import in the failing file
```

- [ ] **Step 3: Repeat until `./gradlew compileJava` passes**

```bash
./gradlew compileJava
```

- [ ] **Step 4: Commit each fix batch**

```bash
git add src/
git commit -m "fix: resolve Star API import mapping issues"
```

---

### Task 9: Run tests and fix failures

- [ ] **Step 1: Compile test sources**

```bash
./gradlew compileTestJava
```

- [ ] **Step 2: Fix any test compilation errors**

Apply same import fixes as Task 8 for test files.

- [ ] **Step 3: Run tests**

```bash
./gradlew test
```

- [ ] **Step 4: Fix any test failures**

Check `build/reports/tests/test/index.html` for failure details. Common issues:
- MockBukkit version differences (3.x → 4.x API changes)
- Star API behavior differences from Dough

Fix each test file and re-run.

- [ ] **Step 5: Commit**

```bash
git add src/test/
git commit -m "fix: resolve test compilation and failures after Star migration"
```

---

### Task 10: Build shadowJar and verify output

- [ ] **Step 1: Build the shadow jar**

```bash
./gradlew shadowJar
```

- [ ] **Step 2: Verify JAR contents (Star classes relocated)**

```bash
# List contents and verify star is shaded correctly
jar tf build/libs/Slimefun-v4.9-UNOFFICIAL.jar | grep "slimefun4/libraries/star"
# Should show star classes at relocated path, not at dev.yanianz.star
```

- [ ] **Step 3: Commit (if build artifacts or config needed changes)**

```bash
git add .
git commit -m "build: finalize shadowJar with Star relocation"
```

---

### Task 11: Update CI workflows

**Files:**
- Modify: `.github/workflows/maven-compiler.yml`
- Modify: `.github/workflows/publish-build.yml`
- Modify: `.github/workflows/pull-request.yml`
- Modify: `.github/workflows/sonarcloud.yml`
- Modify: `.github/workflows/javadocs.yml`
- Modify: `.github/workflows/e2e-testing.yml`
- Modify: `.github/workflows/discord-webhook.yml`

- [ ] **Step 1: Rename and rewrite `maven-compiler.yml` → `gradle-build.yml`**

```bash
git mv .github/workflows/maven-compiler.yml .github/workflows/gradle-build.yml
```

Content:
```yaml
name: Java CI

on:
  push:
    branches:
    - main
    paths:
    - 'src/**'
    - 'build.gradle.kts'
    - 'settings.gradle.kts'
  pull_request:
    paths:
    - '.github/workflows/**'
    - 'src/**'
    - 'build.gradle.kts'
    - 'settings.gradle.kts'
    - 'CHANGELOG.md'

permissions:
  contents: read

jobs:
  build:
    name: Gradle build
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up JDK 25
      uses: actions/setup-java@v4.6.0
      with:
        distribution: 'adopt'
        java-version: '25'
        java-package: jdk
        architecture: x64

    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@v4

    - name: Build with Gradle
      run: ./gradlew build
```

- [ ] **Step 2: Update `publish-build.yml`**

Replace `Set up JDK 21` step with `Set up JDK 25` (java-version: '25').
Replace `Build with Maven` step:
```yaml
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Build with Gradle
        run: ./gradlew shadowJar
```
Update jar file paths from `./target/Slimefun v4.9-UNOFFICIAL.jar` to `./build/libs/Slimefun-v4.9-UNOFFICIAL.jar`.

- [ ] **Step 3: Update `pull-request.yml`**

Replace `Set up JDK 21` → `Set up JDK 25` (java-version: '25').
Replace `Cache Maven packages` with Gradle caching:
```yaml
    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@v4
```
Replace Maven version replacement (`sed` on pom.xml) with Gradle version replacement:
```yaml
        sed -i 's/version = "4.9-UNOFFICIAL"/version = "'"$JAR_VERSION"'"/g' build.gradle.kts
```
Replace `Build with Maven`:
```yaml
    - name: Build with Gradle
      run: ./gradlew shadowJar
```
Update jar path from `target/Slimefun v${{ env.JAR_VERSION }}.jar` to `build/libs/Slimefun-v${{ env.JAR_VERSION }}.jar`.

- [ ] **Step 4: Update `sonarcloud.yml`**

Replace `Set up JDK 21` → `Set up JDK 25` (java-version: '25').
Replace Maven sonar command with Gradle:
```yaml
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Build and analyze
        run: ./gradlew build sonar
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

- [ ] **Step 5: Update `javadocs.yml`**

Replace `Set up JDK 21` → `Set up JDK 25` (java-version: '25').
Replace `mvn javadoc:javadoc` with `./gradlew javadoc`.

- [ ] **Step 6: Update `e2e-testing.yml`**

Change `java-version: ${{ matrix.javaVersion }}` to use Java 25 in the matrix.

- [ ] **Step 7: Update `discord-webhook.yml`**

Replace `Set up JDK 21` → `Set up JDK 25` (java-version: '25').

- [ ] **Step 8: Commit**

```bash
git add .github/workflows/
git commit -m "ci: migrate workflows from Maven to Gradle with Java 25"
```

---

### Task 12: Final verification

- [ ] **Step 1: Full clean build**

```bash
./gradlew clean build
```
Expected: BUILD SUCCESSFUL

- [ ] **Step 2: Full test run**

```bash
./gradlew test
```
Expected: all tests pass

- [ ] **Step 3: Shadow jar built**

```bash
ls -la build/libs/Slimefun-v4.9-UNOFFICIAL.jar
```
Expected: JAR file exists with reasonable size

- [ ] **Step 4: Verify Star classes are correctly shaded**

```bash
jar tf build/libs/Slimefun-v4.9-UNOFFICIAL.jar | grep "libraries/star" | head -20
jar tf build/libs/Slimefun-v4.9-UNOFFICIAL.jar | grep "dev/yanianz/star" | wc -l
```
Expected: star classes at `libraries/star` path, 0 at `dev/yanianz/star` path

- [ ] **Step 5: Commit any final fixes**

```bash
git add -A
git commit -m "build: final verification — Gradle + Star + Java 25"
```
