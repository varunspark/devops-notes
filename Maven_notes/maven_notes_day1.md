# Maven Notes

---

## Maven Day 1 — Introduction & Core Concepts

---

### Why Maven is Non-Negotiable for DevOps

Here is the most common problem in Java development:

**"I manually compiled 10 Java files, downloaded 30 JARs, and now my teammate's machine won't build the project."**

You write Java code — it compiles on your machine. Your teammate clones the repo, spends 2 hours downloading the right JAR versions, fixing classpath errors, and still gets a different result. Every environment is slightly different. Chaos.

Maven solves this completely. You define exactly what your project needs in one file — and Maven downloads the right dependencies, compiles, tests, and packages identically on every machine with one command.

**Maven is in every Java/DevOps job description. No exceptions.**

---

### What You Will Learn Today

- What Maven is and why it exists
- Maven vs other build tools
- Core concepts — POM, lifecycle, phases, goals, plugins, dependencies
- Installing Maven
- Your first Maven project
- Dependency management
- Maven plugins
- Maven profiles
- Multi-module projects
- Best practices

---

### What is Maven?

Maven is a **build automation tool** that compiles code, runs tests, manages dependencies, and packages your application — all from one command, on any machine.

```
Normal way (painful):               Maven way (smart):

Download JAR A manually             Write pom.xml with dependencies
Download JAR B manually             Run: mvn clean install
Put them in /lib folder             Done. All dependencies downloaded,
Fix classpath manually              code compiled, tests run, JAR built.
Compile each file manually
...repeat on every machine
```

**Three things that make Maven special:**

| Feature | What it means |
|---|---|
| Convention over configuration | Maven knows where your code lives — `src/main/java`. No setup needed. |
| Dependency management | Declare what you need in pom.xml — Maven downloads it automatically |
| Standardised lifecycle | Same commands work on every Maven project everywhere |

---

### Maven vs Other Build Tools

This is one of the most asked interview questions:

```
Tool          What it does                        When to use
----------    --------------------------------    -------------------------
Maven         Build Java projects, manage deps    Standard Java/enterprise
Gradle        Build Java/Android, faster builds   Android, large monorepos
Ant           Old XML-based build tool            Legacy Java projects
npm           Build JavaScript projects           Node.js / frontend apps
Make          Build C/C++ projects                System-level programs
```

**Key point:** Maven and Gradle do the same job. Maven is more opinionated (less config), Gradle is more flexible (more powerful). Learn Maven first — Gradle will feel familiar.

---

### Core Concepts

Five concepts. Understand these and you understand Maven.

| Concept | What it is | Real world analogy |
|---|---|---|
| POM | Project definition file (pom.xml) | Blueprint of a building |
| Lifecycle | The sequence of build steps | Assembly line in a factory |
| Phase | One step in the lifecycle | One station on the assembly line |
| Goal | The actual action executed | The worker doing the job |
| Plugin | Tool that provides goals | The machine at each station |

**How they connect:**

```
POM        →  WHAT to build   (project definition + dependencies)
Lifecycle  →  THE PLAN        (ordered steps: compile → test → package)
Phase      →  one step        (e.g. compile, test, package)
Goal       →  actual action   (compiler:compile, surefire:test)
Plugin     →  the tool        (maven-compiler-plugin, maven-jar-plugin)
```

---

### Installing Maven

**On Ubuntu / WSL:**

```bash
sudo apt update
sudo apt install maven -y

# Verify installation:
mvn --version
```

**Expected output:**
```
Apache Maven 3.x.x
Java version: 17.x.x
```

**On macOS (with Homebrew):**

```bash
brew install maven
mvn --version
```

**Test it works — create and build your first project:**

```bash
mvn archetype:generate \
  -DgroupId=com.myapp \
  -DartifactId=hello-world \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false

cd hello-world
mvn package
```

If you see **BUILD SUCCESS** — you're set up correctly. That's your first Maven win.

---

### The Build Lifecycle

Maven has **3 built-in lifecycles**. The default lifecycle is what you use 95% of the time.

**Default Lifecycle — most important phases in order:**

```
validate   →  check project structure is correct
compile    →  compile your source code (src/main/java)
test       →  run unit tests (src/test/java)
package    →  bundle into JAR / WAR file
verify     →  run integration tests
install    →  put the JAR in your local ~/.m2 repository
deploy     →  push the JAR to a remote repository (Nexus/Artifactory)
```

**Critical rule:** Running a phase runs ALL phases before it.

```bash
mvn package     # runs: validate → compile → test → package
mvn install     # runs: validate → compile → test → package → verify → install
mvn compile     # runs: validate → compile only
```

---

### The POM File — Heart of Maven

`pom.xml` is the Project Object Model — the single file that defines everything about your project.

```xml
<!-- pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <!-- Project identity — called GAV coordinates -->
  <groupId>com.mycompany</groupId>       <!-- your organisation -->
  <artifactId>my-app</artifactId>        <!-- project name -->
  <version>1.0.0</version>              <!-- version -->
  <packaging>jar</packaging>             <!-- output type: jar, war, pom -->

  <properties>
    <java.version>17</java.version>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
  </properties>

  <dependencies>
    <!-- add your dependencies here -->
  </dependencies>

</project>
```

**GAV Coordinates explained:**

| Coordinate | What it is | Example |
|---|---|---|
| groupId | Your organisation / package root | `com.google`, `org.springframework` |
| artifactId | The project name | `my-app`, `spring-core` |
| version | The version number | `1.0.0`, `2.1.5-SNAPSHOT` |

These three together uniquely identify any artifact in the world.

---

### Dependency Management

Instead of downloading JARs manually, declare them in `pom.xml`. Maven downloads them automatically.

```xml
<dependencies>

  <!-- Spring Boot -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.2.0</version>
  </dependency>

  <!-- JUnit — only for testing, not shipped in final JAR -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
  </dependency>

</dependencies>
```

**Dependency scopes explained:**

| Scope | When available | Included in JAR? | Use case |
|---|---|---|---|
| compile | Always (default) | Yes | Your main libraries |
| test | Test phase only | No | JUnit, Mockito |
| provided | Compile only | No | Servlet API (server provides it) |
| runtime | Runtime only | Yes | JDBC drivers |

**Where Maven downloads JARs from:**

```
1. Local cache     →  ~/.m2/repository       (checked first, instant)
2. Central repo    →  repo.maven.apache.org   (internet, one-time download)
3. Private repo    →  your Nexus/Artifactory  (company internal artifacts)
```

**Check what's in your dependency tree:**

```bash
mvn dependency:tree
```

Essential command for debugging version conflicts.

---

### Standard Project Structure

Maven enforces a standard folder structure. No configuration needed — it just works.

```
my-app/
├── pom.xml                          ← project definition
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/myapp/
    │   │       └── App.java         ← your application code
    │   └── resources/
    │       └── application.yml      ← config files
    └── test/
        ├── java/
        │   └── com/myapp/
        │       └── AppTest.java     ← your test code
        └── resources/
            └── test.yml             ← test config files
```

**Convention over configuration:**
- Source code → `src/main/java`
- Test code → `src/test/java`
- Resources → `src/main/resources`
- Output → `target/` folder (auto-created, never commit to Git)

```
# .gitignore — always add this
target/
*.class
```

---

### Plugins

Plugins are what actually do the work. Every phase is powered by a plugin.

**Most common plugins:**

| Plugin | What it does | Example use |
|---|---|---|
| maven-compiler-plugin | Compiles Java code | Set Java version |
| maven-surefire-plugin | Runs unit tests | Configure test runner |
| maven-jar-plugin | Packages into a JAR | Set main class |
| maven-war-plugin | Packages into a WAR | Web app deployment |
| maven-shade-plugin | Creates a fat JAR (all deps inside) | Standalone runnable JAR |
| spring-boot-maven-plugin | Spring Boot specific packaging | Boot app deployment |

**Configuring a plugin in pom.xml:**

```xml
<build>
  <plugins>

    <!-- Set Java version -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.11.0</version>
      <configuration>
        <source>17</source>
        <target>17</target>
      </configuration>
    </plugin>

    <!-- Build a fat JAR with all dependencies included -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-shade-plugin</artifactId>
      <version>3.5.0</version>
      <executions>
        <execution>
          <phase>package</phase>
          <goals>
            <goal>shade</goal>
          </goals>
          <configuration>
            <transformers>
              <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                <mainClass>com.myapp.App</mainClass>
              </transformer>
            </transformers>
          </configuration>
        </execution>
      </executions>
    </plugin>

  </plugins>
</build>
```

---

### SNAPSHOT vs Release Versions

```
1.0.0-SNAPSHOT   →  Work in progress. Changes every build. Use during development.
1.0.0            →  Released. Immutable. Never changes. Use in production.
```

**The rules:**

```
Development  →  use SNAPSHOT    →  1.0.0-SNAPSHOT
Finished     →  release it      →  1.0.0
Next cycle   →  bump version    →  1.1.0-SNAPSHOT
```

Maven re-downloads SNAPSHOTs frequently. Release versions are cached forever once downloaded.

---

### Maven Profiles

Profiles let you customise the build for different environments — dev, staging, production.

```xml
<profiles>

  <!-- Development profile -->
  <profile>
    <id>dev</id>
    <activation>
      <activeByDefault>true</activeByDefault>   <!-- used by default -->
    </activation>
    <properties>
      <db.url>jdbc:mysql://localhost:3306/devdb</db.url>
      <log.level>DEBUG</log.level>
    </properties>
  </profile>

  <!-- Production profile -->
  <profile>
    <id>prod</id>
    <properties>
      <db.url>jdbc:mysql://prod-server:3306/proddb</db.url>
      <log.level>ERROR</log.level>
    </properties>
  </profile>

</profiles>
```

**Activate a profile at build time:**

```bash
mvn package -P prod        # build with production profile
mvn package -P dev         # build with dev profile (default)
```

---

### Multi-Module Projects

When your app grows, split it into modules. Each module is its own Maven project with its own `pom.xml`.

**Structure:**

```
my-enterprise-app/              ← parent project
├── pom.xml                     ← parent pom (packaging: pom)
├── my-app-core/                ← module 1 (shared logic)
│   ├── pom.xml
│   └── src/...
├── my-app-api/                 ← module 2 (REST API)
│   ├── pom.xml
│   └── src/...
└── my-app-web/                 ← module 3 (web UI)
    ├── pom.xml
    └── src/...
```

**Parent pom.xml:**

```xml
<project>
  <groupId>com.mycompany</groupId>
  <artifactId>my-enterprise-app</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>    <!-- IMPORTANT: must be pom, not jar -->

  <modules>
    <module>my-app-core</module>
    <module>my-app-api</module>
    <module>my-app-web</module>
  </modules>

  <!-- Shared dependency versions for all modules -->
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>3.2.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

**Child module pom.xml (inherits from parent):**

```xml
<project>
  <parent>
    <groupId>com.mycompany</groupId>
    <artifactId>my-enterprise-app</artifactId>
    <version>1.0.0</version>
  </parent>

  <artifactId>my-app-api</artifactId>   <!-- only override what's different -->

  <dependencies>
    <dependency>
      <groupId>com.mycompany</groupId>
      <artifactId>my-app-core</artifactId>   <!-- depend on another module -->
      <version>${project.version}</version>
    </dependency>
  </dependencies>
</project>
```

**Build all modules in one command:**

```bash
mvn clean install           # builds all modules in correct order
mvn clean install -pl my-app-api    # build only one module
```

---

### Repositories — Local, Central, Private

```
~/.m2/repository/                   ← local cache on your machine
    com/
      google/
        guava/
          32.0/
            guava-32.0.jar          ← downloaded JARs live here
```

**Configure a private repository in pom.xml:**

```xml
<repositories>
  <repository>
    <id>company-nexus</id>
    <url>https://nexus.mycompany.com/repository/maven-public/</url>
  </repository>
</repositories>
```

**Configure credentials in `~/.m2/settings.xml` (never in pom.xml!):**

```xml
<!-- ~/.m2/settings.xml -->
<settings>
  <servers>
    <server>
      <id>company-nexus</id>
      <username>myusername</username>
      <password>mypassword</password>
    </server>
  </servers>
</settings>
```

**Golden rules:**
- Credentials go in `settings.xml` — never in `pom.xml`
- `pom.xml` goes to Git — `settings.xml` stays local
- Never commit `~/.m2/` to Git — it's a cache, not source code

---

### Maven in a DevOps CI/CD Pipeline

In a real DevOps pipeline, Maven runs inside CI/CD (GitHub Actions, Jenkins, GitLab CI).

**GitHub Actions workflow for a Maven project:**

```yaml
# .github/workflows/build.yml
name: Build and Test

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Java 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven dependencies       # don't re-download every run
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

      - name: Build and test
        run: mvn clean verify

      - name: Upload JAR artifact
        uses: actions/upload-artifact@v3
        with:
          name: app-jar
          path: target/*.jar
```

**Dockerfile — multi-stage build with Maven:**

```dockerfile
# Stage 1: Build the JAR using Maven
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline          # cache deps before copying source
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run — lightweight image, no Maven needed
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build it:

```bash
docker build -t my-app:1.0.0 .
docker run -p 8080:8080 my-app:1.0.0
```

---

### Common Maven Commands Cheat Sheet

```bash
# Create a new project from template
mvn archetype:generate -DgroupId=com.myapp -DartifactId=my-app \
  -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false

# Full clean build
mvn clean install

# Compile only
mvn compile

# Run tests only
mvn test

# Package into JAR/WAR (skipping tests)
mvn package -DskipTests

# Dry run — see what would happen
mvn install --dry-run

# Show full dependency tree
mvn dependency:tree

# Check for newer dependency versions
mvn versions:display-dependency-updates

# Run with a specific profile
mvn package -P prod

# Run with extra system properties
mvn package -Denv=staging

# Run only a specific plugin goal
mvn compiler:compile
mvn surefire:test

# Delete target/ folder
mvn clean

# Show effective POM (after all inheritance is resolved)
mvn help:effective-pom

# Debug a build with verbose output
mvn clean install -X
```

---

### Common Modules / Built-in Plugins Reference

These are the plugins you'll see 90% of the time:

| Plugin | What it does | Key goal |
|---|---|---|
| maven-compiler-plugin | Compiles Java source | `compiler:compile` |
| maven-surefire-plugin | Runs unit tests | `surefire:test` |
| maven-jar-plugin | Creates the JAR file | `jar:jar` |
| maven-war-plugin | Creates the WAR file | `war:war` |
| maven-shade-plugin | Fat JAR with all deps | `shade:shade` |
| maven-resources-plugin | Copies resources | `resources:resources` |
| maven-clean-plugin | Deletes target/ | `clean:clean` |
| maven-install-plugin | Installs to ~/.m2 | `install:install` |
| maven-deploy-plugin | Pushes to remote repo | `deploy:deploy` |
| versions-maven-plugin | Check/update versions | `versions:display-dependency-updates` |
| spring-boot-maven-plugin | Spring Boot packaging | `spring-boot:run` |

---

### Interview Questions — Maven

**Q: What is Maven and why is it used?**
Maven is a build automation tool for Java projects. It manages dependencies, compiles code, runs tests, and packages the application using a standard lifecycle. It ensures builds are reproducible across all machines.

**Q: What is a POM file?**
POM stands for Project Object Model. It's the `pom.xml` file at the root of every Maven project. It defines the project's identity (GAV coordinates), dependencies, plugins, and build configuration.

**Q: What are GAV coordinates?**
GroupId, ArtifactId, and Version — three values that together uniquely identify any Maven artifact in the world. Like a unique address for a library.

**Q: What is the difference between `mvn install` and `mvn deploy`?**
`mvn install` puts the artifact in your local `~/.m2` repository. `mvn deploy` pushes it to a remote shared repository like Nexus or Artifactory so your whole team can use it.

**Q: What is dependency scope in Maven?**
Scope controls when a dependency is available. `compile` (default) is always available. `test` is only during testing. `provided` is available at compile time but not packaged (the server provides it). `runtime` is not needed to compile but needed to run.

**Q: What is the difference between SNAPSHOT and release versions?**
A SNAPSHOT version (e.g. `1.0.0-SNAPSHOT`) is a work in progress — it changes frequently. A release version (e.g. `1.0.0`) is immutable and stable. Maven handles them differently — SNAPSHOTs are re-downloaded more frequently.

**Q: What is a Maven profile?**
A profile lets you customise the build for different environments. You can define different database URLs, log levels, or plugin configs per profile, then activate the right one at build time with `-P profileName`.

**Q: What is a multi-module Maven project?**
A project with a parent POM that contains multiple child modules, each with its own `pom.xml`. The parent manages shared dependencies and plugin versions. Useful for splitting a large application into separate components like `core`, `api`, `web`.

**Q: Where should repository credentials be stored?**
In `~/.m2/settings.xml` on the local machine or CI/CD environment — never in `pom.xml` which gets committed to Git.

**Q: What does `mvn dependency:tree` do?**
It prints the complete dependency graph including transitive dependencies (dependencies of your dependencies). Essential for debugging version conflicts.

---

### Project Folder — Full Real-World Setup

```
my-project/
├── pom.xml                     ← project definition
├── .gitignore                  ← always ignore target/
├── .github/
│   └── workflows/
│       └── build.yml           ← CI/CD pipeline
├── Dockerfile                  ← container build
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/myapp/
│   │   │       ├── App.java
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       └── repository/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       └── application-prod.yml
│   └── test/
│       └── java/
│           └── com/myapp/
│               └── AppTest.java
└── target/                     ← auto-generated, never commit
    └── my-app-1.0.0.jar
```

**`.gitignore` — always add this:**

```
target/
*.class
*.jar
.mvn/
!.mvn/wrapper/
```

---

### End of Day 1 Checklist

- [ ] Maven installed — `mvn --version` works
- [ ] First project generated with `archetype:generate`
- [ ] `mvn clean install` runs successfully
- [ ] `pom.xml` understood — groupId, artifactId, version
- [ ] Dependencies added with correct scope
- [ ] Build lifecycle phases understood and practiced
- [ ] A plugin configured in pom.xml
- [ ] `mvn dependency:tree` run on a project
- [ ] Multi-module concept understood
- [ ] settings.xml created with repository config
- [ ] 10 interview questions answered in your own words

---

*Next — Maven Day 2: BOM (Bill of Materials), Nexus/Artifactory setup, publishing artifacts, Maven Wrapper (mvnw), integrating Maven with Jenkins and GitHub Actions, Dockerfile multi-stage builds*
