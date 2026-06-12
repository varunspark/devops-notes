# Maven Notes

---

## Maven Day 4 — Kubernetes, Advanced Testing & Microservices

---

### What Day 3 Covered (Quick Recap)

Before Day 4 — make sure these are solid:

```
Spring Boot plugin    →  spring-boot:run, spring-boot:build-image, fat JAR
Checkstyle            →  code style enforcement, build fails on violation
SpotBugs              →  bytecode bug pattern detection
JaCoCo                →  test coverage with minimum threshold enforcement
SonarQube             →  full code quality dashboard + Quality Gate
Enforcer plugin       →  banned deps, required Java/Maven version
Production pom.xml    →  complete reference with all plugins wired together
Monorepo strategies   →  parallel builds, incremental builds, module ordering
```

If any of these feel fuzzy — re-read Day 3 before continuing.

---

### What You Will Learn Today

- Jib plugin — build Docker images without Docker daemon
- Maven with Kubernetes — generating manifests, Helm from Maven
- Failsafe plugin — integration tests separate from unit tests
- Testcontainers — real databases in tests with zero setup
- Maven in GitLab CI — full pipeline with caching and stages
- OWASP Dependency-Check — vulnerability scanning
- Maven best practices for microservices
- The complete DevOps mental model — how everything connects

---

### Jib — Build Docker Images Without Docker

**The problem with standard Docker builds:**

```
Standard flow:
  mvn package → produces JAR
  docker build → needs Docker daemon running
  docker push  → needs Docker credentials configured

Problems:
  - Docker daemon must be installed and running
  - Requires root/privileged access in many CI environments
  - Rebuilds entire image even when only your code changed
  - Slow — copies all layers every time
```

**Jib solves all of this.** It builds and pushes a Docker image directly from Maven — no Docker daemon, no Dockerfile, no Docker installed at all.

```
Jib flow:
  mvn compile jib:build
  → builds optimised image
  → pushes directly to registry
  → no Docker required anywhere
```

**How Jib optimises images — layer separation:**

```
Standard Docker image (one fat layer):
  Layer 1: JRE base image
  Layer 2: everything else — deps + your code (300MB, rebuilt every time)

Jib image (smart layers):
  Layer 1: JRE base image          (cached forever)
  Layer 2: dependencies            (cached until pom.xml changes)
  Layer 3: resources               (cached until resources change)
  Layer 4: your compiled classes   (only this rebuilds on code change — tiny)
```

Result: rebuilds are 10–50x faster because only your code layer changes.

**Add Jib to pom.xml:**

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.4.0</version>
  <configuration>

    <from>
      <!-- Base image — use distroless for security (no shell, no package manager) -->
      <image>gcr.io/distroless/java17-debian12</image>
    </from>

    <to>
      <!-- Target registry and image name -->
      <image>myregistry.azurecr.io/my-service</image>
      <tags>
        <tag>${project.version}</tag>
        <tag>latest</tag>
      </tags>
    </to>

    <container>
      <mainClass>com.myapp.Application</mainClass>
      <ports>
        <port>8080</port>
      </ports>
      <environment>
        <JAVA_OPTS>-Xms256m -Xmx512m</JAVA_OPTS>
      </environment>
      <!-- Add build time as image label -->
      <labels>
        <version>${project.version}</version>
      </labels>
      <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
    </container>

  </configuration>
</plugin>
```

**Key Jib commands:**

```bash
# Build and push to remote registry
mvn compile jib:build

# Build to local Docker daemon (requires Docker, for testing)
mvn compile jib:dockerBuild

# Build to a local tar file (no Docker, no registry)
mvn compile jib:buildTar

# Build and push as part of the Maven lifecycle
mvn package jib:build -DskipTests
```

**Authenticate to a registry in CI:**

```bash
# For Docker Hub
mvn jib:build \
  -Djib.to.auth.username=$DOCKER_USERNAME \
  -Djib.to.auth.password=$DOCKER_PASSWORD

# For AWS ECR
mvn jib:build \
  -Djib.to.auth.username=AWS \
  -Djib.to.auth.password=$(aws ecr get-login-password)

# For Azure ACR — set in environment
export JIB_TO_AUTH_USERNAME=$ACR_USERNAME
export JIB_TO_AUTH_PASSWORD=$ACR_PASSWORD
mvn jib:build
```

---

### Maven with Kubernetes

**Full flow — from code to running in Kubernetes:**

```
Code change
    ↓
mvn compile jib:build     → Docker image pushed to registry
    ↓
kubectl apply             → Kubernetes pulls image, deploys pod
```

**Generate Kubernetes manifests from Maven using the fabric8 plugin:**

```xml
<plugin>
  <groupId>io.fabric8</groupId>
  <artifactId>kubernetes-maven-plugin</artifactId>
  <version>6.13.0</version>
  <configuration>
    <namespace>my-namespace</namespace>
    <images>
      <image>
        <name>myregistry.azurecr.io/my-service:${project.version}</name>
      </image>
    </images>
  </configuration>
</plugin>
```

**Commands:**

```bash
# Generate Kubernetes YAML manifests
mvn k8s:resource

# Apply manifests to Kubernetes cluster
mvn k8s:apply

# Build image + deploy to Kubernetes in one command
mvn package k8s:build k8s:push k8s:apply -DskipTests

# Delete all deployed resources
mvn k8s:undeploy
```

**Generated manifests appear in:**

```
target/
└── classes/
    └── META-INF/
        └── jkube/
            ├── kubernetes.yml     ← Deployment + Service
            └── kubernetes/
                ├── deployment.yml
                └── service.yml
```

**Helm chart generation from Maven:**

```xml
<plugin>
  <groupId>io.fabric8</groupId>
  <artifactId>helm-maven-plugin</artifactId>
  <version>6.13.0</version>
  <configuration>
    <chartName>my-service</chartName>
    <chartVersion>${project.version}</chartVersion>
    <outputDir>${project.build.directory}/helm</outputDir>
  </configuration>
</plugin>
```

```bash
# Generate Helm chart
mvn helm:init helm:generate

# Package the chart into a .tgz
mvn helm:package

# Push chart to Helm repo
mvn helm:upload
```

**Full CI/CD pipeline with Kubernetes:**

```
Developer pushes code
    ↓
CI: mvn clean verify           (tests + quality gate)
    ↓
CI: mvn jib:build              (image → registry)
    ↓
CD: helm upgrade --install     (deploy to Kubernetes)
    ↓
Kubernetes: rolling update     (zero downtime)
```

---

### Failsafe Plugin — Integration Tests

**The problem with putting everything in unit tests:**

```
Unit tests:         fast, isolated, mock everything — run in milliseconds
Integration tests:  slow, need real DB/services — run in seconds/minutes

If you mix them: mvn test runs ALL tests, CI takes 15 minutes.
You want: mvn test (unit only, fast), mvn verify (unit + integration, thorough)
```

**Failsafe separates them cleanly:**

```
maven-surefire-plugin   →  runs unit tests         (mvn test phase)
maven-failsafe-plugin   →  runs integration tests  (mvn verify phase)
```

**Naming convention — Failsafe picks up tests automatically:**

```
Unit tests (Surefire):           Integration tests (Failsafe):
  *Test.java                       *IT.java
  *Tests.java                      *ITCase.java
  *TestCase.java                   *IntegrationTest.java

UserServiceTest.java    ✅         UserServiceIT.java      ✅
OrderControllerTest.java ✅        PaymentFlowIT.java      ✅
```

**Add Failsafe:**

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-failsafe-plugin</artifactId>
  <version>3.2.5</version>
  <executions>
    <execution>
      <goals>
        <goal>integration-test</goal>   <!-- runs ITs -->
        <goal>verify</goal>             <!-- fails build if ITs failed -->
      </goals>
    </execution>
  </executions>
</plugin>
```

**Both goals are required.** `integration-test` runs the tests but does NOT fail the build immediately — it waits. `verify` then checks the results and fails if needed. This lets teardown steps (stopping containers etc.) always run, even after a failure.

**Running tests selectively:**

```bash
mvn test                    # unit tests only — fast (seconds)
mvn verify                  # unit + integration tests — thorough (minutes)
mvn verify -DskipITs        # skip integration tests
mvn verify -DskipTests      # skip everything
mvn failsafe:integration-test -Dit.test=UserServiceIT   # run one IT
```

---

### Testcontainers — Real Databases in Tests

**The problem with mocking databases:**

```
Mocked DB in tests:         Real DB in production:
  H2 in-memory SQL             PostgreSQL
  No transactions              Real transactions
  No stored procedures         Stored procedures
  No indexes                   Indexes and query plans
  Passes tests                 Fails in production ← the real problem
```

**Testcontainers spins up a real Docker container for each test run — automatically.**

```
mvn test
  → Testcontainers starts a PostgreSQL container
  → Tests run against real PostgreSQL
  → Container is stopped and deleted after tests
  → No leftovers, no conflicts between developers
```

**Add Testcontainers:**

```xml
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers</artifactId>
  <version>1.19.3</version>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId>
  <version>1.19.3</version>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>1.19.3</version>
  <scope>test</scope>
</dependency>
```

**Integration test with a real PostgreSQL container:**

```java
// UserRepositoryIT.java
@SpringBootTest
@Testcontainers
class UserRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // Override Spring datasource to point to the container
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndRetrieveUser() {
        User user = new User("Alice", "alice@example.com");
        userRepository.save(user);

        Optional<User> found = userRepository.findByEmail("alice@example.com");

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }
}
```

**Testcontainers available for:**

```
PostgreSQL          →  new PostgreSQLContainer<>()
MySQL               →  new MySQLContainer<>()
MongoDB             →  new MongoDBContainer<>()
Redis               →  new GenericContainer<>("redis:7-alpine")
Kafka               →  new KafkaContainer()
RabbitMQ            →  new RabbitMQContainer()
Any Docker image    →  new GenericContainer<>("image:tag")
```

**Reuse containers across tests (faster):**

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withReuse(true);    // container stays running between test classes
```

**In CI — Testcontainers needs Docker:**

```yaml
# GitHub Actions — Docker is available by default on ubuntu-latest
- name: Run integration tests
  run: ./mvnw verify
  # Docker is already available — Testcontainers just works
```

---

### Maven in GitLab CI

GitLab CI is widely used in enterprise environments. Maven integrates cleanly with GitLab's pipeline stages.

**`.gitlab-ci.yml` — full pipeline:**

```yaml
# .gitlab-ci.yml

image: maven:3.9-eclipse-temurin-17

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  MAVEN_CLI_OPTS: "--batch-mode --no-transfer-progress"

# Cache Maven deps between pipeline runs
cache:
  key: "$CI_JOB_NAME"
  paths:
    - .m2/repository/

stages:
  - build
  - test
  - quality
  - package
  - publish
  - deploy

# ─── Stage 1: Compile ────────────────────────────────────────
compile:
  stage: build
  script:
    - ./mvnw $MAVEN_CLI_OPTS compile
  artifacts:
    paths:
      - target/classes/
    expire_in: 1 hour

# ─── Stage 2: Unit Tests ─────────────────────────────────────
unit-test:
  stage: test
  script:
    - ./mvnw $MAVEN_CLI_OPTS test
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/*.xml    # GitLab reads this natively
    paths:
      - target/surefire-reports/
    expire_in: 1 week

# ─── Stage 3: Integration Tests ──────────────────────────────
integration-test:
  stage: test
  services:
    - docker:dind                             # Docker-in-Docker for Testcontainers
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - ./mvnw $MAVEN_CLI_OPTS verify -DskipUTs
  artifacts:
    when: always
    reports:
      junit: target/failsafe-reports/*.xml

# ─── Stage 4: Code Quality ───────────────────────────────────
sonar:
  stage: quality
  script:
    - ./mvnw $MAVEN_CLI_OPTS verify sonar:sonar
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.token=$SONAR_TOKEN
  only:
    - main
    - merge_requests

owasp-scan:
  stage: quality
  script:
    - ./mvnw $MAVEN_CLI_OPTS dependency-check:check
  artifacts:
    paths:
      - target/dependency-check-report.html
    expire_in: 1 week

# ─── Stage 5: Package ────────────────────────────────────────
package:
  stage: package
  script:
    - ./mvnw $MAVEN_CLI_OPTS package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week

# ─── Stage 6: Publish Image ──────────────────────────────────
publish-image:
  stage: publish
  script:
    - ./mvnw $MAVEN_CLI_OPTS compile jib:build
      -Djib.to.image=$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
      -Djib.to.auth.username=$CI_REGISTRY_USER
      -Djib.to.auth.password=$CI_REGISTRY_PASSWORD
  only:
    - tags                   # only publish on git tags

# ─── Stage 7: Deploy ─────────────────────────────────────────
deploy-staging:
  stage: deploy
  script:
    - kubectl set image deployment/my-service
        my-service=$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
  environment:
    name: staging
    url: https://staging.myapp.com
  only:
    - tags
```

**Key GitLab CI concepts for Maven:**

```
cache          →  persist .m2 between runs — saves 3–5 min per pipeline
artifacts      →  pass files between stages (compiled classes, JARs)
reports        →  junit: key makes test results appear in GitLab UI natively
services       →  docker:dind needed for Testcontainers in GitLab
only/rules     →  control which branches/tags trigger which stages
```

---

### OWASP Dependency-Check — Vulnerability Scanning

**The problem:** Your project uses 50 third-party libraries. One of them has a known security vulnerability (CVE). You don't know about it. You ship it to production.

**OWASP Dependency-Check scans every dependency against the National Vulnerability Database (NVD) and fails the build if known CVEs are found.**

```
Real-world examples of what this catches:
  Log4Shell (CVE-2021-44228)   →  log4j 2.x before 2.17.1
  Spring4Shell (CVE-2022-22965) → Spring Framework before 5.3.18
  Jackson CVEs                  → older jackson-databind versions
```

**Add the plugin:**

```xml
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>9.0.7</version>
  <configuration>

    <!-- Fail if a dependency has a CVSS score >= 7.0 (High severity) -->
    <failBuildOnCVSS>7</failBuildOnCVSS>

    <!-- Output formats -->
    <formats>
      <format>HTML</format>
      <format>JSON</format>
      <format>XML</format>
    </formats>

    <!-- Suppress false positives -->
    <suppressionFile>owasp-suppressions.xml</suppressionFile>

    <!-- NVD API key for faster downloads (register free at nvd.nist.gov) -->
    <nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>

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

**Run it:**

```bash
mvn dependency-check:check          # scan and fail on CVEs
mvn dependency-check:aggregate      # scan entire multi-module project
```

**First run is slow** — downloads the full NVD database (~500MB). Subsequent runs use the cached database and are fast.

**Suppress a false positive:**

```xml
<!-- owasp-suppressions.xml -->
<suppressions>
  <suppress>
    <notes>This CVE affects a feature we don't use.</notes>
    <gav regex="true">^com\.example:my-lib:.*$</gav>
    <cve>CVE-2023-12345</cve>
  </suppress>
</suppressions>
```

**CVSS score guide:**

```
0.1 – 3.9   →  Low
4.0 – 6.9   →  Medium
7.0 – 8.9   →  High      ← fail build here (recommended threshold)
9.0 – 10.0  →  Critical
```

**In CI — cache the NVD database:**

```yaml
- name: Cache OWASP NVD database
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository/org/owasp/dependency-check-data
    key: owasp-nvd-${{ github.run_id }}
    restore-keys: owasp-nvd-

- name: OWASP vulnerability scan
  run: ./mvnw dependency-check:check
  env:
    NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
```

---

### Maven Best Practices for Microservices

Microservices have different challenges from monoliths. Here are the patterns used by real teams.

**1. One repo per service (polyrepo) — each has its own pom.xml:**

```
user-service/
├── pom.xml                 ← standalone, inherits from company parent
├── Dockerfile / Jib config
└── src/...

order-service/
├── pom.xml
└── src/...
```

**2. Company parent POM — shared standards across all services:**

```xml
<!-- Published to Nexus: com.mycompany:company-parent:1.0.0 -->
<project>
  <groupId>com.mycompany</groupId>
  <artifactId>company-parent</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>

  <!-- All services inherit: Java version, plugins, quality tools -->
  <properties>
    <java.version>17</java.version>
  </properties>

  <build>
    <pluginManagement>
      <plugins>
        <!-- JaCoCo pre-configured for every service -->
        <plugin>
          <groupId>org.jacoco</groupId>
          <artifactId>jacoco-maven-plugin</artifactId>
          <version>0.8.11</version>
          <!-- ... standard config ... -->
        </plugin>
        <!-- Checkstyle pointing to company rules -->
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-checkstyle-plugin</artifactId>
          <configuration>
            <configLocation>
              https://nexus.mycompany.com/checkstyle/company-checks.xml
            </configLocation>
          </configuration>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```

**Each service's pom.xml just inherits:**

```xml
<parent>
  <groupId>com.mycompany</groupId>
  <artifactId>company-parent</artifactId>
  <version>1.0.0</version>
</parent>

<artifactId>user-service</artifactId>
<version>1.3.0-SNAPSHOT</version>

<!-- Only what's unique to this service -->
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

**3. Version properties — never hardcode versions inside `<dependencies>`:**

```xml
<!-- Bad — version scattered everywhere -->
<dependency>
  <groupId>com.mycompany</groupId>
  <artifactId>common-lib</artifactId>
  <version>2.1.0</version>        <!-- hardcoded -->
</dependency>

<!-- Good — version in properties, easy to update -->
<properties>
  <common-lib.version>2.1.0</common-lib.version>
</properties>

<dependency>
  <groupId>com.mycompany</groupId>
  <artifactId>common-lib</artifactId>
  <version>${common-lib.version}</version>
</dependency>
```

**4. Always use the Maven Wrapper — never `mvn` bare:**

```bash
# Bad — assumes Maven is installed, version unknown
mvn clean install

# Good — uses pinned version from .mvn/wrapper/
./mvnw clean install
```

**5. Build info in the artifact — know exactly what's running:**

```xml
<!-- spring-boot-maven-plugin generates build-info.properties -->
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>build-info</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

```bash
# After build — query the running service
curl http://localhost:8080/actuator/info

# Returns:
{
  "build": {
    "version": "1.3.0",
    "artifact": "user-service",
    "time": "2024-01-15T10:30:00Z",
    "group": "com.mycompany"
  }
}
```

**6. Separate test types clearly:**

```
src/test/java/
├── unit/
│   └── UserServiceTest.java        ← fast, no containers
└── integration/
    └── UserRepositoryIT.java       ← Testcontainers, slower
```

**7. Pin every dependency version — never rely on ranges:**

```xml
<!-- Bad — unpredictable, breaks on new release -->
<version>[1.0,2.0)</version>

<!-- Good — exact, reproducible -->
<version>1.5.3</version>
```

---

### The Complete DevOps Mental Model — Everything Connected

This is the full picture of how Maven fits into a real DevOps pipeline end-to-end.

```
Developer workstation
─────────────────────
./mvnw clean verify          ← runs locally before pushing
  ├── compile
  ├── unit tests (Surefire)
  ├── integration tests (Failsafe + Testcontainers)
  ├── Checkstyle
  ├── SpotBugs
  └── JaCoCo coverage check

Git push to feature branch
──────────────────────────
CI pipeline triggered (GitHub Actions / GitLab CI / Jenkins)
  │
  ├── Stage 1: Build & Unit Test
  │     ./mvnw clean test
  │
  ├── Stage 2: Integration Test
  │     ./mvnw verify (Failsafe + Testcontainers)
  │
  ├── Stage 3: Quality Gate
  │     ./mvnw sonar:sonar         → SonarQube Quality Gate
  │     ./mvnw dependency-check:check  → OWASP CVE scan
  │
  ├── Stage 4: Package
  │     ./mvnw package jib:build   → Docker image pushed to registry
  │
  └── Stage 5: Deploy (main branch / tags only)
        helm upgrade --install      → Kubernetes rolling deployment

Production
──────────
Kubernetes pulls image from registry
Rolling update — zero downtime
/actuator/info shows exact version deployed
Monitoring picks up health metrics
```

**Every tool's role in one table:**

| Tool | Role | Fails build if |
|---|---|---|
| Maven Wrapper | Pinned build tool version | Wrong Maven version |
| Enforcer | Build prerequisites | Wrong Java, banned dep |
| Surefire | Unit tests | Any unit test fails |
| Failsafe | Integration tests | Any integration test fails |
| Testcontainers | Real containers in tests | Container fails to start |
| JaCoCo | Coverage measurement | Coverage below threshold |
| Checkstyle | Code style | Style rule violated |
| SpotBugs | Bug patterns | Bug pattern detected |
| SonarQube | Overall quality gate | Gate conditions not met |
| OWASP | CVE scanning | CVE score above threshold |
| Jib | Image build + push | Build or push fails |
| Release Plugin | Version + tag + deploy | Tests fail, uncommitted changes |

---

### Interview Questions — Maven Day 4

**Q: What is Jib and why is it better than a standard Dockerfile for Java apps?**
Jib is a Maven plugin by Google that builds and pushes Docker images without needing a Docker daemon or Dockerfile. It splits the image into smart layers — base image, dependencies, resources, and classes. On rebuilds, only the classes layer changes, making rebuilds 10–50x faster. It also works in restricted CI environments where Docker is not available.

**Q: What is the difference between Surefire and Failsafe plugins?**
Surefire runs unit tests during the `test` phase — files matching `*Test.java`. Failsafe runs integration tests during the `verify` phase — files matching `*IT.java`. The separation means `mvn test` is fast (unit only) and `mvn verify` is thorough (unit + integration). Failsafe also ensures teardown always runs even if tests fail.

**Q: What is Testcontainers and why is it better than H2 for testing?**
Testcontainers spins up real Docker containers (PostgreSQL, MySQL, Kafka etc.) during tests and tears them down afterwards. H2 is an in-memory database that behaves differently from production databases — it doesn't support all SQL features, transaction semantics, or stored procedures. Testcontainers tests against the exact same database engine as production.

**Q: What does OWASP Dependency-Check do?**
It scans all your project's dependencies (including transitive ones) against the National Vulnerability Database (NVD) and reports known CVEs. You configure a CVSS score threshold — typically 7.0 — and the build fails if any dependency has a vulnerability at or above that score.

**Q: What is the Kubernetes Maven Plugin (fabric8)?**
A Maven plugin that generates Kubernetes YAML manifests from your project configuration and can apply them to a Kubernetes cluster. It integrates image building, manifest generation, and deployment into Maven goals — so `mvn k8s:apply` deploys your app.

**Q: What is a company parent POM and why do microservices teams use it?**
A parent POM published to Nexus that all company services inherit from. It centralises Java version, plugin configuration, quality tool setup, and repository settings. New services inherit all standards automatically and updating the parent version propagates changes to every service that uses it.

**Q: How do you speed up OWASP scans in CI?**
Cache the NVD database between pipeline runs — it's large (~500MB) but only changes daily. Use a NVD API key (free registration) to download updates faster. Run the OWASP scan in a separate optional stage so it doesn't block the main build/deploy flow.

**Q: What is distroless and why use it as a base image with Jib?**
Distroless images (by Google) contain only the runtime — no shell, no package manager, no system tools. The result is a much smaller and more secure image because there is no shell for attackers to exploit and no unnecessary packages with their own CVEs.

---

### End of Day 4 Checklist

- [ ] Jib plugin configured — `./mvnw jib:build` pushes an image to a registry
- [ ] Jib layer separation understood — why it's faster than standard Docker builds
- [ ] Failsafe plugin configured — `*IT.java` tests run on `mvn verify`
- [ ] Testcontainers used in one integration test — real PostgreSQL container
- [ ] `mvn test` (fast) vs `mvn verify` (thorough) — difference understood and practiced
- [ ] GitLab CI pipeline written with stages, cache, and artifacts
- [ ] OWASP scan run — `mvn dependency-check:check`
- [ ] CVSS threshold set — build fails on High severity CVE
- [ ] Company parent POM concept understood
- [ ] Full DevOps mental model diagram drawn from memory
- [ ] All 8 interview questions answered in your own words

---

### Complete Maven Series — Final Summary

```
Day 1  →  Core concepts: POM, lifecycle, dependencies, plugins, profiles, multi-module
Day 2  →  DevOps integration: BOM, Wrapper, Nexus, deploy, Jenkins, GitHub Actions, Docker
Day 3  →  Code quality: Spring Boot plugin, Checkstyle, SpotBugs, JaCoCo, SonarQube, Enforcer
Day 4  →  Advanced: Jib, Kubernetes, Failsafe, Testcontainers, GitLab CI, OWASP, microservices
```

**You now know everything needed to use Maven professionally in a DevOps role.**

---

*Suggested next series: Gradle — the modern alternative to Maven. Covers Kotlin DSL, incremental builds, build caching, Android builds, and migrating from Maven.*
