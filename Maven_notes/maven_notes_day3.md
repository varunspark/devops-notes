# Maven Notes

---

## Maven Day 3 — Code Quality, Spring Boot & Real-World Patterns

---

### What Day 2 Covered (Quick Recap)

Before Day 3 — make sure these are solid:

```
BOM                →  import once, versions managed automatically
Maven Wrapper      →  ./mvnw — pinned Maven version in every environment
mvn deploy         →  publish to Nexus/Artifactory
Jenkinsfile        →  pipeline as code for CI/CD
GitHub Actions     →  caching Maven deps, publishing artifacts
Multi-stage Docker →  build in JDK stage, run in lean JRE stage
Release Plugin     →  mvn release:prepare + release:perform
Conflict resolution→  dependency:tree + dependencyManagement
```

If any of these feel fuzzy — re-read Day 2 before continuing.

---

### What You Will Learn Today

- Spring Boot with Maven — spring-boot-maven-plugin deep dive
- Code quality — Checkstyle, SpotBugs, PMD
- Test coverage — JaCoCo
- SonarQube — full code analysis integration
- Enforcing quality gates in CI/CD
- Signing artifacts — GPG for Maven Central publishing
- Monorepo strategies — large multi-module projects
- Maven archetypes — project templates
- Real-world pom.xml — production-grade complete file

---

### Spring Boot with Maven

Spring Boot's Maven plugin is one of the most important plugins in the ecosystem. It goes far beyond just packaging.

**Add the plugin:**

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <configuration>
        <excludes>
          <!-- Don't include Lombok in the final JAR -->
          <exclude>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
          </exclude>
        </excludes>
      </configuration>
    </plugin>
  </plugins>
</build>
```

**Key goals:**

| Goal | What it does | When to use |
|---|---|---|
| `spring-boot:run` | Run the app locally from Maven | Development |
| `spring-boot:build-image` | Build OCI/Docker image with Buildpacks | No Dockerfile needed |
| `spring-boot:repackage` | Repackage JAR into executable fat JAR | Default, runs at `package` phase |
| `spring-boot:build-info` | Generate build metadata | For `/actuator/info` endpoint |
| `spring-boot:help` | List all available goals | Learning |

**Run the app directly:**

```bash
./mvnw spring-boot:run

# With Spring profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# With JVM arguments
./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-Xmx512m"
```

**Build a Docker image — without writing a Dockerfile:**

```bash
# Buildpacks — zero Dockerfile, production-grade image
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:1.0.0
```

**What the repackaged JAR looks like:**

```
target/
├── my-app-1.0.0.jar           ← fat JAR (executable, has all deps inside)
└── my-app-1.0.0.jar.original  ← original thin JAR (kept for reference)
```

**Run the fat JAR directly:**

```bash
java -jar target/my-app-1.0.0.jar
java -jar target/my-app-1.0.0.jar --spring.profiles.active=prod
java -jar target/my-app-1.0.0.jar --server.port=9090
```

---

### Code Quality — The Big Picture

Shipping working code is not enough in professional DevOps. You also need:

```
Checkstyle   →  code style rules (formatting, naming conventions)
PMD          →  code smell detection (unused variables, empty catch blocks)
SpotBugs     →  bug pattern detection (null dereferences, resource leaks)
JaCoCo       →  test coverage measurement (what % of code is tested)
SonarQube    →  combines all of the above + security vulnerability scanning
```

**How they fit in the pipeline:**

```
Code pushed
    ↓
mvn verify
    ↓
compile → test → [checkstyle] → [pmd] → [spotbugs] → [jacoco] → package
    ↓
Build fails if quality gates are not met
    ↓
SonarQube dashboard updated with full report
```

---

### Checkstyle — Code Style Rules

Checkstyle enforces a consistent coding style across the whole team. If your code doesn't match the rules — the build fails.

**Add the plugin:**

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <version>3.3.1</version>
  <configuration>
    <configLocation>google_checks.xml</configLocation>   <!-- Google style rules -->
    <failsOnError>true</failsOnError>
    <consoleOutput>true</consoleOutput>
  </configuration>
  <executions>
    <execution>
      <id>checkstyle</id>
      <phase>verify</phase>             <!-- runs during mvn verify -->
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

**Custom rules file — `checkstyle.xml`:**

```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
  "https://checkstyle.org/dtds/configuration_1_3.dtd">

<module name="Checker">
  <module name="TreeWalker">

    <!-- Method names must start with lowercase -->
    <module name="MethodName"/>

    <!-- No wildcard imports (import java.util.*) -->
    <module name="AvoidStarImport"/>

    <!-- Max line length -->
    <module name="LineLength">
      <property name="max" value="120"/>
    </module>

    <!-- Every public class/method must have Javadoc -->
    <module name="JavadocMethod">
      <property name="scope" value="public"/>
    </module>

  </module>
</module>
```

**Run checkstyle alone:**

```bash
mvn checkstyle:check        # fail if violations
mvn checkstyle:checkstyle   # generate HTML report only
```

---

### SpotBugs — Bug Pattern Detection

SpotBugs analyses your compiled bytecode and finds real bug patterns before they hit production.

```
What SpotBugs catches:
  - Null pointer dereferences
  - Resource leaks (unclosed streams)
  - Infinite loops
  - Incorrect equals/hashCode implementations
  - Ignored return values
  - SQL injection patterns
```

**Add the plugin:**

```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.8.3.1</version>
  <configuration>
    <effort>Max</effort>           <!-- analysis depth: Min, Default, Max -->
    <threshold>Medium</threshold>  <!-- severity to fail on: Low, Medium, High -->
    <failOnError>true</failOnError>
  </configuration>
  <executions>
    <execution>
      <goals>
        <goal>check</goal>
      </goals>
      <phase>verify</phase>
    </execution>
  </executions>
</plugin>
```

**Suppress a false positive with an annotation:**

```java
import edu.umd.cs.findbugs.annotations.SuppressFBWarnings;

@SuppressFBWarnings(value = "NP_NULL_ON_SOME_PATH", justification = "Null check is above")
public void processUser(User user) {
    // SpotBugs warns here but the null check is guaranteed upstream
}
```

**Run SpotBugs alone:**

```bash
mvn spotbugs:check      # fail if bugs found
mvn spotbugs:gui        # open the SpotBugs GUI (visual report)
mvn spotbugs:spotbugs   # generate XML report only
```

---

### JaCoCo — Test Coverage

JaCoCo measures how much of your source code is actually executed by your tests. A coverage report tells you what's tested and what isn't.

**Add the plugin:**

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <executions>

    <!-- Step 1: Set up the JaCoCo agent before tests run -->
    <execution>
      <id>prepare-agent</id>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>

    <!-- Step 2: Generate report after tests -->
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals>
        <goal>report</goal>
      </goals>
    </execution>

    <!-- Step 3: Enforce minimum coverage threshold -->
    <execution>
      <id>check</id>
      <goals>
        <goal>check</goal>
      </goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
            <limits>
              <!-- Build fails if overall coverage drops below 80% -->
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>

  </executions>
</plugin>
```

**Run and view the report:**

```bash
mvn clean verify                        # runs tests + generates coverage
open target/site/jacoco/index.html      # open the HTML coverage report
```

**Coverage report shows:**

```
Class                  Missed Instructions   Coverage
──────────────────     ───────────────────   ────────
UserService            12 of 150             92%    ✅
PaymentService         45 of 60              25%    ❌  ← needs more tests
OrderController        0 of 88               100%   ✅
```

**In CI — coverage report goes to SonarQube or uploaded as an artifact:**

```bash
mvn clean verify
# jacoco.xml is at: target/site/jacoco/jacoco.xml
# HTML report at: target/site/jacoco/index.html
```

---

### SonarQube — Full Code Analysis

SonarQube is the industry-standard platform that combines code quality, coverage, security scanning, and technical debt tracking into one dashboard.

```
What SonarQube analyses:
  Bugs          →  code that will probably fail at runtime
  Vulnerabilities →  security issues (SQL injection, XSS etc.)
  Code Smells   →  maintainability problems (long methods, duplications)
  Coverage      →  test coverage from JaCoCo
  Duplications  →  copy-paste code
  Security Hotspots → code that needs manual security review
```

**Run SonarQube locally with Docker:**

```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  sonarqube:lts-community
```

Access at `http://localhost:9000` — default login: `admin` / `admin`

**Add the Sonar plugin to pom.xml:**

```xml
<plugin>
  <groupId>org.sonarsource.scanner.maven</groupId>
  <artifactId>sonar-maven-plugin</artifactId>
  <version>3.10.0.2594</version>
</plugin>
```

**Configure Sonar properties:**

```xml
<properties>
  <sonar.host.url>http://localhost:9000</sonar.host.url>
  <sonar.projectKey>my-app</sonar.projectKey>
  <sonar.projectName>My Application</sonar.projectName>

  <!-- Tell SonarQube where JaCoCo report is -->
  <sonar.coverage.jacoco.xmlReportPaths>
    target/site/jacoco/jacoco.xml
  </sonar.coverage.jacoco.xmlReportPaths>

  <!-- Exclude generated code from analysis -->
  <sonar.exclusions>
    **/generated/**,**/target/**
  </sonar.exclusions>
</properties>
```

**Run analysis:**

```bash
# Run tests + coverage + push to SonarQube
mvn clean verify sonar:sonar -Dsonar.token=your-token-here
```

**SonarQube in GitHub Actions:**

```yaml
- name: Run tests and analysis
  run: ./mvnw clean verify sonar:sonar
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

- name: SonarQube Quality Gate check
  uses: sonarsource/sonarqube-quality-gate-action@master
  timeout-minutes: 5
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

**Quality Gate — the key concept:**

```
A Quality Gate is a pass/fail threshold on SonarQube metrics.
Default gate fails if any of these are true:

  New bugs:                > 0
  New vulnerabilities:     > 0
  New code smells:         > 5
  New coverage:            < 80%
  New duplications:        > 3%

Pipeline fails → PR is blocked → bad code never merges.
This is enforcement, not just reporting.
```

---

### GPG Signing — Publishing to Maven Central

If you want to publish an open-source library to Maven Central (the public repo), you must sign your artifacts with GPG. This proves the JAR actually came from you.

**Why signing matters:**

```
Without signing:  Anyone can publish a JAR claiming to be 'com.google.guava'
With signing:     Maven verifies the JAR was signed by the real owner's GPG key
```

**Step 1 — Generate a GPG key:**

```bash
gpg --gen-key
# Follow prompts: name, email, passphrase

# List your keys
gpg --list-keys

# Publish your public key to a keyserver
gpg --keyserver keyserver.ubuntu.com --send-keys YOUR_KEY_ID
```

**Step 2 — Add the GPG plugin:**

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-gpg-plugin</artifactId>
  <version>3.1.0</version>
  <executions>
    <execution>
      <id>sign-artifacts</id>
      <phase>verify</phase>
      <goals>
        <goal>sign</goal>
      </goals>
      <configuration>
        <gpgArguments>
          <arg>--pinentry-mode</arg>
          <arg>loopback</arg>
        </gpgArguments>
      </configuration>
    </execution>
  </executions>
</plugin>
```

**Step 3 — Add Sonatype OSSRH distribution (Maven Central gateway):**

```xml
<distributionManagement>
  <snapshotRepository>
    <id>ossrh</id>
    <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
  </snapshotRepository>
  <repository>
    <id>ossrh</id>
    <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
  </repository>
</distributionManagement>
```

**Step 4 — Deploy signed artifact:**

```bash
mvn clean deploy -P release \
  -Dgpg.passphrase=yourpassphrase
```

**In CI/CD — store GPG key as secret:**

```bash
# Export key for CI
gpg --export-secret-keys YOUR_KEY_ID | base64 > private.key
# Store the base64 output in GitHub Secrets as GPG_PRIVATE_KEY
```

```yaml
- name: Import GPG key
  run: echo "${{ secrets.GPG_PRIVATE_KEY }}" | base64 --decode | gpg --import

- name: Deploy signed artifact
  run: ./mvnw clean deploy -DskipTests
  env:
    GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
```

---

### Maven Archetypes — Project Templates

An archetype is a project template. Instead of starting from scratch, generate a fully structured project in seconds.

**Built-in archetypes:**

```bash
# Standard Java app
mvn archetype:generate \
  -DarchetypeGroupId=org.apache.maven.archetypes \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DgroupId=com.myapp -DartifactId=my-app -Dversion=1.0.0-SNAPSHOT

# Web app (WAR)
mvn archetype:generate \
  -DarchetypeArtifactId=maven-archetype-webapp \
  -DgroupId=com.myapp -DartifactId=my-webapp

# Spring Boot (via Spring Initializr — the best option)
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.2.0 \
  -d groupId=com.myapp \
  -d artifactId=my-spring-app \
  -d dependencies=web,data-jpa,actuator \
  -o my-spring-app.zip
unzip my-spring-app.zip
```

**Create your own archetype from an existing project:**

```bash
# Inside an existing project
mvn archetype:create-from-project

# This generates a reusable archetype in target/generated-sources/archetype
# Install it:
cd target/generated-sources/archetype
mvn install

# Now use it to generate new projects:
mvn archetype:generate -DarchetypeGroupId=com.mycompany \
  -DarchetypeArtifactId=my-company-archetype
```

**Company archetypes are powerful:** Create one archetype with pre-configured Checkstyle, JaCoCo, SonarQube, company Nexus repo, Docker setup — and every new project in your company starts with all of it built in.

---

### Monorepo Strategies for Large Multi-Module Projects

When a company has 20+ teams all working in one Maven multi-module repo, you need strategies to keep builds fast and manageable.

**The problem with naive multi-module builds:**

```bash
mvn clean install    # rebuilds ALL 50 modules even if you changed 1 file
                     # takes 20 minutes on a large codebase
```

**Strategy 1 — Build only changed modules:**

```bash
# With git diff to find what changed
CHANGED=$(git diff --name-only HEAD~1 HEAD | \
  grep src | cut -d/ -f1 | sort -u | tr '\n' ',')

mvn clean install -pl $CHANGED -am    # -am = also build upstream dependencies
```

**Strategy 2 — Incremental builds (Gradle's strength, but Maven has it too):**

```xml
<!-- In parent pom.xml — skip unchanged modules -->
<plugin>
  <groupId>io.github.gitflowincrementalbuilder</groupId>
  <artifactId>gitflow-incremental-builder</artifactId>
  <version>4.5.0</version>
  <extensions>true</extensions>
  <configuration>
    <skipIfPathMatches>.*\.md$</skipIfPathMatches>
    <buildUpstream>changed</buildUpstream>
    <buildDownstream>always</buildDownstream>
  </configuration>
</plugin>
```

**Strategy 3 — Parallel builds:**

```bash
mvn clean install -T 1C    # 1 thread per CPU core — 2–4x faster on multi-core
mvn clean install -T 8     # explicit 8 threads
```

**Strategy 4 — Module dependency graph (understand before you build):**

```bash
mvn dependency:resolve      # resolve all deps
mvn help:evaluate -Dexpression=project.modules   # list all modules
```

**Recommended monorepo folder layout for 20+ modules:**

```
enterprise-platform/
├── pom.xml                         ← root parent
├── bom/
│   └── pom.xml                     ← BOM for version management
├── libs/
│   ├── common-utils/               ← shared utilities
│   ├── security-lib/               ← shared security
│   └── persistence-lib/            ← shared DB layer
├── services/
│   ├── user-service/               ← microservice 1
│   ├── order-service/              ← microservice 2
│   └── payment-service/            ← microservice 3
├── apps/
│   ├── web-frontend/
│   └── admin-portal/
└── build-tools/
    ├── checkstyle.xml
    ├── spotbugs-exclude.xml
    └── jacoco.properties
```

**Root parent pom.xml module order matters — Maven resolves in the order listed:**

```xml
<modules>
  <module>bom</module>              <!-- first — other modules import this -->
  <module>libs/common-utils</module>
  <module>libs/security-lib</module>
  <module>libs/persistence-lib</module>
  <module>services/user-service</module>
  <module>services/order-service</module>
  <module>services/payment-service</module>
  <module>apps/web-frontend</module>
</modules>
```

---

### Production-Grade pom.xml — Complete Reference

Here is what a real, production-quality pom.xml looks like — with every important section included.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <!-- ─── Identity ──────────────────────────────── -->
  <groupId>com.mycompany</groupId>
  <artifactId>my-service</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>jar</packaging>
  <name>My Service</name>
  <description>Core service for the platform</description>

  <!-- ─── Parent (Spring Boot BOM) ─────────────── -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
  </parent>

  <!-- ─── Properties ───────────────────────────── -->
  <properties>
    <java.version>17</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

    <!-- SonarQube -->
    <sonar.projectKey>my-service</sonar.projectKey>
    <sonar.coverage.jacoco.xmlReportPaths>
      target/site/jacoco/jacoco.xml
    </sonar.coverage.jacoco.xmlReportPaths>

    <!-- Dependency versions not managed by Spring BOM -->
    <mapstruct.version>1.5.5.Final</mapstruct.version>
  </properties>

  <!-- ─── Dependencies ─────────────────────────── -->
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <!-- ─── Build ────────────────────────────────── -->
  <build>
    <plugins>

      <!-- Spring Boot packaging -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <configuration>
          <excludes>
            <exclude>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
            </exclude>
          </excludes>
        </configuration>
      </plugin>

      <!-- JaCoCo coverage -->
      <plugin>
        <groupId>org.jacoco</groupId>
        <artifactId>jacoco-maven-plugin</artifactId>
        <version>0.8.11</version>
        <executions>
          <execution>
            <goals><goal>prepare-agent</goal></goals>
          </execution>
          <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
          </execution>
          <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
              <rules>
                <rule>
                  <limits>
                    <limit>
                      <counter>LINE</counter>
                      <value>COVEREDRATIO</value>
                      <minimum>0.80</minimum>
                    </limit>
                  </limits>
                </rule>
              </rules>
            </configuration>
          </execution>
        </executions>
      </plugin>

      <!-- Checkstyle -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-checkstyle-plugin</artifactId>
        <version>3.3.1</version>
        <configuration>
          <configLocation>checkstyle.xml</configLocation>
          <failsOnError>true</failsOnError>
        </configuration>
        <executions>
          <execution>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
          </execution>
        </executions>
      </plugin>

      <!-- SpotBugs -->
      <plugin>
        <groupId>com.github.spotbugs</groupId>
        <artifactId>spotbugs-maven-plugin</artifactId>
        <version>4.8.3.1</version>
        <configuration>
          <effort>Max</effort>
          <threshold>Medium</threshold>
        </configuration>
        <executions>
          <execution>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
          </execution>
        </executions>
      </plugin>

    </plugins>
  </build>

  <!-- ─── Distribution ─────────────────────────── -->
  <distributionManagement>
    <repository>
      <id>nexus-releases</id>
      <url>http://nexus.mycompany.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
      <id>nexus-snapshots</id>
      <url>http://nexus.mycompany.com/repository/maven-snapshots/</url>
    </snapshotRepository>
  </distributionManagement>

  <!-- ─── Profiles ─────────────────────────────── -->
  <profiles>
    <profile>
      <id>dev</id>
      <activation><activeByDefault>true</activeByDefault></activation>
      <properties>
        <spring.profiles.active>dev</spring.profiles.active>
      </properties>
    </profile>
    <profile>
      <id>prod</id>
      <properties>
        <spring.profiles.active>prod</spring.profiles.active>
      </properties>
    </profile>
  </profiles>

</project>
```

---

### Enforcer Plugin — Build Rules

The Maven Enforcer plugin lets you set rules that must pass for the build to succeed. Block builds that use the wrong Java version, banned dependencies, or Maven version.

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-enforcer-plugin</artifactId>
  <version>3.4.1</version>
  <executions>
    <execution>
      <id>enforce</id>
      <goals>
        <goal>enforce</goal>
      </goals>
      <configuration>
        <rules>

          <!-- Require Java 17+ -->
          <requireJavaVersion>
            <version>[17,)</version>
            <message>Java 17 or higher is required.</message>
          </requireJavaVersion>

          <!-- Require Maven 3.8+ -->
          <requireMavenVersion>
            <version>[3.8,)</version>
          </requireMavenVersion>

          <!-- No duplicate dependencies -->
          <banDuplicatePomDependencyVersions/>

          <!-- Ban a specific library (e.g. banned by security policy) -->
          <bannedDependencies>
            <excludes>
              <exclude>log4j:log4j</exclude>   <!-- Log4Shell — never use this -->
            </excludes>
          </bannedDependencies>

        </rules>
        <fail>true</fail>
      </configuration>
    </execution>
  </executions>
</plugin>
```

**Run the enforcer alone:**

```bash
mvn enforcer:enforce
```

This is critical in large teams — prevents developers from accidentally using banned libraries or wrong runtimes.

---

### Full Quality Gate Pipeline in GitHub Actions

Putting everything together — a production-grade CI pipeline that enforces all quality standards:

```yaml
# .github/workflows/quality.yml
name: Quality Gate

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest

    services:
      sonarqube:
        image: sonarqube:lts-community
        ports:
          - 9000:9000

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0           # SonarQube needs full git history

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

      - name: Enforcer check
        run: ./mvnw enforcer:enforce

      - name: Build, test, coverage, checkstyle, spotbugs
        run: ./mvnw clean verify

      - name: SonarQube analysis
        run: ./mvnw sonar:sonar
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: http://localhost:9000

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: target/site/jacoco/

      - name: Quality Gate result
        uses: sonarsource/sonarqube-quality-gate-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

**What happens on every PR:**

```
1. Enforcer checks Java version + banned deps
2. Checkstyle checks code style
3. Tests run
4. JaCoCo checks ≥80% coverage — fails if below
5. SpotBugs checks for bug patterns
6. SonarQube analyses everything
7. Quality Gate check — PR is blocked if gate fails
8. Coverage report uploaded for review
```

---

### Interview Questions — Maven Day 3

**Q: What is JaCoCo and how do you enforce a coverage threshold?**
JaCoCo is a Java code coverage library. It instruments your tests and measures what percentage of your source code lines/branches are executed. You enforce a threshold using the `check` goal with a `<minimum>` value — the build fails if coverage drops below that number.

**Q: What is the difference between Checkstyle, PMD, and SpotBugs?**
Checkstyle enforces code formatting and naming style rules. PMD analyses source code for potential problems like unused variables and empty catch blocks. SpotBugs analyses compiled bytecode for real bug patterns like null dereferences and resource leaks. They complement each other — use all three.

**Q: What is a SonarQube Quality Gate?**
A Quality Gate is a set of pass/fail conditions on SonarQube metrics — bugs, vulnerabilities, coverage, duplications. If new code in a PR violates the gate, the pipeline fails and the PR is blocked. It's the enforcement mechanism, not just reporting.

**Q: What is the Maven Enforcer plugin?**
A plugin that defines mandatory build rules — required Java version, required Maven version, banned dependencies, no duplicate deps. The build fails immediately if rules are violated, before any compilation happens.

**Q: Why must you sign artifacts for Maven Central?**
GPG signing verifies that a JAR was published by the actual owner of that groupId. Without signing, anyone could publish a malicious JAR claiming to be a legitimate library. Maven Central requires signing for all published artifacts.

**Q: What is a Maven archetype?**
A project template that generates a fully structured Maven project from a single command. Companies create custom archetypes pre-configured with their standards — Checkstyle rules, JaCoCo config, Nexus settings — so every new project starts consistently.

**Q: How do you speed up large multi-module Maven builds?**
Use parallel builds with `-T 1C` (one thread per CPU). Use `-pl` to build only specific modules. Use incremental build tools like the gitflow-incremental-builder to skip unchanged modules. Cache Maven dependencies in CI.

**Q: What does `spring-boot:build-image` do?**
It builds a production-ready OCI/Docker image using Cloud Native Buildpacks — without needing a Dockerfile. Buildpacks automatically configure the image with best practices for the JVM runtime.

---

### End of Day 3 Checklist

- [ ] `spring-boot:run` and `spring-boot:build-image` goals used
- [ ] Checkstyle configured — build fails on style violation
- [ ] SpotBugs configured — build fails on bug pattern
- [ ] JaCoCo configured — build fails below 80% coverage
- [ ] Coverage HTML report opened and reviewed
- [ ] SonarQube running locally via Docker
- [ ] `mvn sonar:sonar` pushed a report to SonarQube dashboard
- [ ] Quality Gate reviewed and understood
- [ ] Maven Enforcer configured with Java version + banned deps
- [ ] Production-grade `pom.xml` written from the reference above
- [ ] Multi-module build run with `-T 1C` parallel threads
- [ ] All 8 interview questions answered in your own words

---

*Next — Maven Day 4: Maven with Kubernetes (Jib plugin, Helm charts from Maven), advanced testing (Failsafe plugin for integration tests, Testcontainers), Maven in GitLab CI, dependency vulnerability scanning with OWASP, and Maven best practices for microservices*
