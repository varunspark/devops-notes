# Maven Notes

---

## Maven Day 2 — Advanced Maven & DevOps Integration

---

### What Day 1 Covered (Quick Recap)

Before Day 2 — make sure these are solid:

```
pom.xml            →  project definition, GAV coordinates
Lifecycle          →  validate → compile → test → package → install → deploy
Dependencies       →  scopes: compile, test, provided, runtime
Plugins            →  compiler, surefire, jar, shade
Profiles           →  -P dev / -P prod
Multi-module       →  parent pom + child modules
mvn clean install  →  your bread-and-butter command
```

If any of these feel fuzzy — re-read Day 1 before continuing.

---

### What You Will Learn Today

- BOM (Bill of Materials) — version management at scale
- Maven Wrapper (mvnw) — no Maven installation needed
- Nexus & Artifactory — private artifact repositories
- Publishing artifacts — `mvn deploy` to a private repo
- Maven in Jenkins — real CI/CD pipeline
- Maven in GitHub Actions — caching, publishing, releasing
- Dockerfile multi-stage builds — production-ready images
- Troubleshooting builds — reading errors, resolving conflicts
- Release management — versions, tags, the release plugin

---

### BOM — Bill of Materials

**The problem:** You have 10 Spring dependencies. Each has its own version. Getting them to work together without conflicts is painful.

```xml
<!-- Without BOM — version on every single dependency -->
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-core</artifactId>
  <version>6.1.2</version>         <!-- must match -->
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-web</artifactId>
  <version>6.1.2</version>         <!-- must match -->
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>6.1.2</version>         <!-- must match — 10 more of these -->
</dependency>
```

**With BOM — import once, versions managed automatically:**

```xml
<dependencyManagement>
  <dependencies>

    <!-- Import the Spring Boot BOM -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>

  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- No version needed — BOM manages it -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
</dependencies>
```

**What a BOM does:**

| Without BOM | With BOM |
|---|---|
| Version on every dependency | Version declared once |
| Risk of mismatched versions | Guaranteed compatible versions |
| Update = change 10 lines | Update = change 1 line |
| Transitive conflicts common | Conflicts resolved by BOM author |

**BOM is the #1 tool for managing dependency versions in enterprise projects.**

---

### Maven Wrapper (mvnw)

**The problem:** Your CI server has Maven 3.6. Your teammate has Maven 3.9. Different versions produce different results.

**The solution:** Maven Wrapper. A small script checked into your repo that downloads the exact right Maven version automatically — no installation required.

**Generate the wrapper:**

```bash
mvn wrapper:wrapper
```

**This creates:**

```
my-project/
├── mvnw              ← Linux/Mac script
├── mvnw.cmd          ← Windows script
└── .mvn/
    └── wrapper/
        └── maven-wrapper.properties   ← pinned Maven version
```

**maven-wrapper.properties:**

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
```

**Now use `./mvnw` instead of `mvn` — everywhere:**

```bash
./mvnw clean install          # Linux/Mac
mvnw.cmd clean install        # Windows
```

**In CI/CD (GitHub Actions, Jenkins) — always use the wrapper:**

```bash
./mvnw clean verify           # uses exact pinned version, no Maven install needed
```

**Golden rules:**
- Always commit `mvnw`, `mvnw.cmd`, and `.mvn/` to Git
- Never commit `~/.m2/` — that's your local cache
- CI pipelines should always use `./mvnw` — never assume Maven is installed

---

### Nexus & Artifactory — Private Artifact Repositories

**What they are:** A private server that stores your built JARs, WARs, and other artifacts — just like Maven Central, but for your company's internal code.

```
Maven Central              →  public JARs (Spring, JUnit, Google etc.)
Nexus / Artifactory        →  your company's private JARs
```

**Why you need a private repo:**

```
Reason                    Why it matters
─────────────────────     ─────────────────────────────────────────
Internal libraries        Code that can't go on Maven Central
Reproducible builds       Maven Central can delete old versions
Offline builds            Cache external deps internally
Access control            Only authorised teams can publish
Speed                     Internal network is faster than internet
```

**Setting up Nexus locally with Docker (fastest way to learn):**

```bash
docker run -d \
  --name nexus \
  -p 8081:8081 \
  -v nexus-data:/nexus-data \
  sonatype/nexus3:latest
```

Access at `http://localhost:8081` — default login: `admin` / check `/nexus-data/admin.password`

**Three repository types in Nexus/Artifactory:**

```
hosted     →  YOUR artifacts go here (release + snapshot repos)
proxy      →  caches Maven Central downloads locally
group      →  combines hosted + proxy into one URL for devs
```

---

### Publishing Artifacts — mvn deploy

`mvn deploy` pushes your built artifact to a remote repository so your whole team can use it.

**Step 1 — Add the distribution repository to pom.xml:**

```xml
<distributionManagement>

  <!-- Where to publish RELEASE versions (e.g. 1.0.0) -->
  <repository>
    <id>nexus-releases</id>
    <url>http://localhost:8081/repository/maven-releases/</url>
  </repository>

  <!-- Where to publish SNAPSHOT versions (e.g. 1.0.0-SNAPSHOT) -->
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>http://localhost:8081/repository/maven-snapshots/</url>
  </snapshotRepository>

</distributionManagement>
```

**Step 2 — Add credentials to `~/.m2/settings.xml` (never in pom.xml):**

```xml
<settings>
  <servers>

    <server>
      <id>nexus-releases</id>
      <username>admin</username>
      <password>yourpassword</password>
    </server>

    <server>
      <id>nexus-snapshots</id>
      <username>admin</username>
      <password>yourpassword</password>
    </server>

  </servers>
</settings>
```

**Step 3 — Deploy:**

```bash
mvn clean deploy                  # build + publish to Nexus
mvn clean deploy -DskipTests      # skip tests, still deploy
```

**What happens:**

```
mvn clean deploy
  ↓
compile → test → package → install → deploy
                                        ↓
                             pushes JAR to Nexus
                             pushes POM to Nexus
                             team can now add it as dependency
```

**Another team uses your artifact:**

```xml
<dependency>
  <groupId>com.mycompany</groupId>
  <artifactId>my-shared-lib</artifactId>
  <version>1.2.0</version>
</dependency>
```

---

### Maven in Jenkins — CI/CD Pipeline

Jenkins is the most common CI/CD server in enterprise DevOps. Maven + Jenkins is a very standard combination.

**Freestyle Jenkins job (basic — via UI):**

```
New Item → Freestyle Project
→ Source Code Management: Git repo URL
→ Build Triggers: Poll SCM or GitHub webhook
→ Build Steps: Invoke top-level Maven targets
  → Goals: clean install
→ Post-build: Archive artifacts: target/*.jar
```

**Jenkinsfile (pipeline as code — the right way):**

```groovy
// Jenkinsfile
pipeline {
    agent any

    tools {
        maven 'Maven-3.9'           // name configured in Jenkins → Global Tools
        jdk 'Java-17'
    }

    environment {
        NEXUS_CREDENTIALS = credentials('nexus-creds')   // Jenkins credentials store
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh './mvnw clean compile'
            }
        }

        stage('Test') {
            steps {
                sh './mvnw test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'    // publish test results
                }
            }
        }

        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }

        stage('Deploy to Nexus') {
            when {
                branch 'main'      // only deploy from main branch
            }
            steps {
                sh './mvnw deploy -DskipTests \
                    -Dnexus.username=${NEXUS_CREDENTIALS_USR} \
                    -Dnexus.password=${NEXUS_CREDENTIALS_PSW}'
            }
        }

    }

    post {
        success {
            echo 'Build succeeded!'
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
        failure {
            echo 'Build failed!'
            mail to: 'team@mycompany.com', subject: 'Build Failed', body: 'Check Jenkins'
        }
    }
}
```

**Why pipeline as code (Jenkinsfile) wins:**
- Lives in your Git repo — versioned with your code
- Code review the pipeline like any other code
- Rebuild any historical commit with the exact same pipeline

---

### Maven in GitHub Actions — Full Pipeline

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:

  # ─── Job 1: Build and Test ───────────────────────────────
  build:
    name: Build & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven local repository
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Build and run tests
        run: ./mvnw clean verify

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Maven Tests
          path: target/surefire-reports/*.xml
          reporter: java-junit

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

  # ─── Job 2: Deploy to Nexus (main branch only) ───────────
  deploy:
    name: Publish Artifact
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          server-id: nexus-releases
          server-username: NEXUS_USERNAME
          server-password: NEXUS_PASSWORD

      - name: Cache Maven
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

      - name: Publish to Nexus
        run: ./mvnw deploy -DskipTests
        env:
          NEXUS_USERNAME: ${{ secrets.NEXUS_USERNAME }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
```

**Cache is critical in CI:**

```
Without cache:  Maven downloads 200MB of deps every run    →  5–8 minutes
With cache:     Maven reuses cached deps                   →  30–60 seconds
```

---

### Dockerfile — Multi-Stage Build (Production Grade)

A multi-stage Dockerfile uses Maven to build in one stage, then creates a lean runtime image.

```dockerfile
# ─── Stage 1: Dependency cache (Docker layer cache trick) ───
FROM maven:3.9-eclipse-temurin-17 AS deps
WORKDIR /build

# Copy pom.xml first — if pom.xml hasn't changed, deps layer is cached
COPY pom.xml .
RUN mvn dependency:go-offline -B     # download all deps, then cache this layer

# ─── Stage 2: Build ─────────────────────────────────────────
FROM deps AS build
COPY src ./src
RUN mvn clean package -DskipTests -B

# ─── Stage 3: Runtime — tiny image, no Maven, no JDK ────────
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Create non-root user — security best practice
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /build/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Why multi-stage matters:**

```
Single-stage image (with Maven + JDK):    ~600 MB
Multi-stage image (JRE only):             ~90 MB

Smaller image = faster pulls, smaller attack surface, cheaper storage
```

**Build and run:**

```bash
docker build -t my-app:1.0.0 .
docker run -p 8080:8080 my-app:1.0.0

# Tag for a registry
docker tag my-app:1.0.0 myregistry.azurecr.io/my-app:1.0.0
docker push myregistry.azurecr.io/my-app:1.0.0
```

---

### Release Management — The Maven Release Plugin

**The problem:** Releasing a new version manually means:
1. Change `1.0.0-SNAPSHOT` to `1.0.0` in pom.xml
2. Commit and tag in Git
3. Build and deploy the release
4. Bump version to `1.1.0-SNAPSHOT`
5. Commit again

That's error-prone. The Maven Release Plugin automates all of it.

**Add the plugin:**

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-release-plugin</artifactId>
  <version>3.0.1</version>
  <configuration>
    <tagNameFormat>v@{project.version}</tagNameFormat>
    <autoVersionSubmodules>true</autoVersionSubmodules>
  </configuration>
</plugin>
```

**Release in two commands:**

```bash
# Step 1: Prepare — updates version, runs tests, creates Git tag
mvn release:prepare

# Step 2: Perform — checks out the tag, builds, deploys to Nexus
mvn release:perform
```

**What `mvn release:prepare` does automatically:**

```
1. Checks no uncommitted changes exist
2. Runs all tests
3. Changes 1.0.0-SNAPSHOT → 1.0.0 in all pom.xml files
4. Commits the change
5. Creates Git tag: v1.0.0
6. Changes version to 1.1.0-SNAPSHOT for next development cycle
7. Commits again
```

**What `mvn release:perform` does:**

```
1. Checks out the v1.0.0 tag
2. Builds the release
3. Deploys to Nexus release repository
```

---

### Dependency Conflict Resolution

When two dependencies need different versions of the same library — Maven picks one. You need to know which and why.

```bash
mvn dependency:tree -Dverbose      # show ALL versions including overridden ones
```

**Reading the output:**

```
[INFO] com.myapp:my-app:jar:1.0.0
[INFO] ├── com.google.guava:guava:jar:32.0.0:compile
[INFO] └── org.springframework:spring-core:jar:6.1.2:compile
[INFO]     └── (com.google.guava:guava:jar:30.0.0:compile - omitted for conflict)
```

`omitted for conflict` means Maven chose `32.0.0` over `30.0.0` (nearest-wins rule).

**Force a specific version — use `<dependencyManagement>`:**

```xml
<dependencyManagement>
  <dependencies>
    <!-- Force this version across all transitive dependencies -->
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>32.1.3-jre</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Exclude a transitive dependency:**

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-core</artifactId>
  <version>6.1.2</version>
  <exclusions>
    <exclusion>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>    <!-- exclude this transitive dep -->
    </exclusion>
  </exclusions>
</dependency>
```

---

### Troubleshooting Common Build Failures

**1. Dependency not found:**

```
[ERROR] Could not resolve dependencies for project ...
        Artifact 'com.example:my-lib:jar:1.0.0' not found
```

Fix:
```bash
mvn clean install -U          # -U forces update of SNAPSHOT dependencies
# Check your repo URL in pom.xml and credentials in settings.xml
```

**2. Compilation error:**

```
[ERROR] COMPILATION ERROR
[ERROR] /src/main/java/App.java:[15,8] cannot find symbol
```

Fix: Read the exact file and line. Check for missing imports, wrong Java version.
```bash
mvn compile -X 2>&1 | grep "ERROR"     # verbose output, filter errors
```

**3. Test failure:**

```
[ERROR] Tests run: 5, Failures: 1, Errors: 0
[ERROR] AppTest.testSomething — expected <200> but was <404>
```

Fix:
```bash
mvn test -Dtest=AppTest#testSomething     # run just the failing test
mvn test -DfailIfNoTests=false            # skip test failures temporarily
cat target/surefire-reports/AppTest.txt   # full test report
```

**4. Out of memory during build:**

```
[ERROR] GC overhead limit exceeded
```

Fix:
```bash
export MAVEN_OPTS="-Xmx1024m -XX:MaxMetaspaceSize=512m"
mvn clean install
```

**5. Version conflict:**

```
[ERROR] NoSuchMethodError: com.google.guava.collect.ImmutableList.of(...)
```

Fix:
```bash
mvn dependency:tree -Dverbose | grep guava    # find which version is being used
# Then force the correct version in <dependencyManagement>
```

---

### settings.xml — Full Reference

`~/.m2/settings.xml` is the global Maven config on your machine. Never commit this — it contains credentials.

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">

  <!-- Local repository location (default: ~/.m2/repository) -->
  <localRepository>/home/user/.m2/repository</localRepository>

  <!-- Credentials for private repositories -->
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>admin</username>
      <password>password</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>admin</username>
      <password>password</password>
    </server>
  </servers>

  <!-- Mirror — redirect Maven Central through your Nexus proxy -->
  <mirrors>
    <mirror>
      <id>nexus-mirror</id>
      <mirrorOf>central</mirrorOf>
      <url>http://localhost:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>

  <!-- Default active profile -->
  <activeProfiles>
    <activeProfile>dev</activeProfile>
  </activeProfiles>

</settings>
```

**When to use settings.xml vs pom.xml:**

```
pom.xml          →  project config — what to build, what deps, what plugins
settings.xml     →  machine config — credentials, mirrors, local repo path
```

---

### Full Advanced Commands Cheat Sheet

```bash
# Force re-download of all SNAPSHOT dependencies
mvn clean install -U

# Skip tests (build only)
mvn clean package -DskipTests

# Run a single test class
mvn test -Dtest=MyServiceTest

# Run a single test method
mvn test -Dtest=MyServiceTest#shouldReturnUser

# Show effective POM (after inheritance + profiles are resolved)
mvn help:effective-pom

# Show effective settings
mvn help:effective-settings

# Check for newer plugin versions
mvn versions:display-plugin-updates

# Check for newer dependency versions
mvn versions:display-dependency-updates

# Automatically update all dependencies to latest
mvn versions:use-latest-releases

# Run only specific module in multi-module build
mvn install -pl my-app-api

# Run module + everything it depends on
mvn install -pl my-app-api -am

# Run module + everything that depends on it
mvn install -pl my-app-core -amd

# Verbose debug output
mvn clean install -X 2>&1 | tee build.log

# Profile activation
mvn package -P prod
mvn package -P dev,metrics        # multiple profiles

# Set property at command line
mvn package -Dspring.profiles.active=staging

# Offline mode (use only cached deps, no internet)
mvn clean install -o

# Thread parallel builds (faster on multi-core)
mvn clean install -T 4            # 4 threads
mvn clean install -T 1C           # 1 thread per CPU core
```

---

### Interview Questions — Maven Day 2

**Q: What is a BOM and why is it useful?**
A Bill of Materials is a special POM imported via `<dependencyManagement>` that centrally manages the versions of a set of related dependencies. It eliminates version conflicts and means you don't need to specify a version on each individual dependency — the BOM guarantees compatible versions.

**Q: What is the Maven Wrapper and why should you use it?**
The Maven Wrapper (`mvnw`) is a script checked into the project repository that downloads and uses a specific pinned Maven version. It ensures every developer and every CI server uses the exact same Maven version, eliminating "works on my machine" build differences.

**Q: What is the difference between `mvn install` and `mvn deploy`?**
`mvn install` puts the artifact in the local `~/.m2/repository` on your machine only. `mvn deploy` pushes it to a remote repository (Nexus/Artifactory) so the whole team can access it. Deploy is used in CI/CD pipelines.

**Q: What is Nexus / Artifactory?**
A private artifact repository server. It stores your team's internal JARs (hosted repo), caches Maven Central downloads to speed up builds (proxy repo), and exposes both through a single URL (group repo). Nexus is the open-source option; Artifactory is the enterprise option.

**Q: What is multi-stage Docker build and why use it with Maven?**
A multi-stage Dockerfile uses separate stages — one with Maven + JDK to build the JAR, and a second with only the JRE to run it. The final image doesn't contain Maven, the JDK, or source code — so it's 6–7x smaller, faster to pull, and has a smaller security attack surface.

**Q: How do you resolve a dependency version conflict in Maven?**
First run `mvn dependency:tree -Dverbose` to see which versions are present and which are being excluded. Then use `<dependencyManagement>` to force a specific version across all transitive dependencies, or use `<exclusions>` on a specific dependency to remove the conflicting transitive dependency.

**Q: What does the Maven Release Plugin do?**
It automates the release process: removes the SNAPSHOT suffix from the version, commits the change, creates a Git tag, builds and deploys the release to Nexus, then bumps the version to the next SNAPSHOT. Two commands replace a multi-step manual process.

**Q: Where do you store Nexus credentials for Maven — and where not?**
In `~/.m2/settings.xml` on the local machine, or as environment variables/secrets in CI/CD. Never in `pom.xml` — that file is committed to Git and would expose credentials publicly.

---

### End of Day 2 Checklist

- [ ] BOM imported in a project — no versions on individual Spring dependencies
- [ ] Maven Wrapper generated — `./mvnw` works on your project
- [ ] Nexus running locally via Docker — accessible at `http://localhost:8081`
- [ ] `mvn deploy` successfully publishes a JAR to Nexus
- [ ] Jenkinsfile written with build, test, and deploy stages
- [ ] GitHub Actions workflow set up with Maven cache
- [ ] Multi-stage Dockerfile built and runs successfully
- [ ] `mvn dependency:tree` used to investigate a conflict
- [ ] `settings.xml` configured with server credentials and mirror
- [ ] Release plugin used to cut a `1.0.0` release from a SNAPSHOT
- [ ] All 8 interview questions answered in your own words

---

*Next — Maven Day 3: Spring Boot with Maven (spring-boot-maven-plugin), code quality with Checkstyle + SpotBugs + JaCoCo coverage, SonarQube integration, signing artifacts for Maven Central, monorepo strategies*
