# Chapter 13 — Project 2: Java Spring Boot Expense Tracker (Maven + MySQL + Compose)

## Objective
Deploy a Java Spring Boot "Expense Tracker" web application, built using Maven, with a MySQL database, fully automated with a multi-stage Dockerfile and Docker Compose — including how to correctly translate Spring Boot's `application.properties` values into Docker environment variables.

## Prerequisites
* OS: Ubuntu EC2 instance with Docker + Docker Compose installed
* Required software: Docker, Docker Compose, git
* Required repository: A Spring Boot "Expense Tracker" project (example: `laundeShubham153/expenses-tracker`, originally forked from Mohammad Al-Sawwi's project)
* Required ports: Port 8080 (opened via AWS Security Group)
* Required knowledge: Chapters 5, 8, 10 (Dockerfile, Compose, Multi-stage builds)

## Architecture / Flow
```text
User Browser
     │  (HTTP request to Port 8080)
     ▼
Spring Boot Container ("expenses_app")
     │  runs on internal Tomcat server, port 8080
     ▼
MySQL Container ("mysql_db")
```
This project is a **two-tier** architecture (application + database), unlike Chapter 12's three-tier (Nginx + app + database), because this app is accessed directly on its own port without a reverse proxy.

## Directory / File Structure
```text
expenses-tracker/
├── src/
│   └── main/
│       ├── java/...
│       └── resources/
│           └── application.properties
├── pom.xml
├── mvnw
├── Dockerfile
└── docker-compose.yml
```

## Step 1 — Clone the Project
```bash
cd ~/projects
git clone https://github.com/LaundeShubham153/expenses-tracker
cd expenses-tracker
```
> Always credit the original author when reusing someone's open-source project — this project was originally built by Mohammad Al-Sawwi and made available under the MIT License.

## Step 2 — Understand Key Project Files

| File | Purpose |
|---|---|
| `mvnw`, `.mvn/` | Maven Wrapper — lets you build the project without installing Maven separately; comes bundled with the project. |
| `src/main/java/...` | The actual Java source code (developer's responsibility, not something a DevOps engineer needs to edit). |
| `src/main/resources/static/` | Front-end assets (CSS, icons, JavaScript for styling the pages). |
| `src/main/resources/application.properties` | The MOST important file for a DevOps engineer — it lists all configuration variables the app needs, including database connection settings. |
| `pom.xml` | Maven's configuration file — declares which dependencies/frameworks the project uses (Spring Boot, Thymeleaf, MySQL connector, validation libraries, etc.), similar in purpose to `requirements.txt` in Python or `package.json` in Node.js. |

## Step 3 — Understand `application.properties`
```bash
cat src/main/resources/application.properties
```
**Expected content (example):**
```properties
spring.application.name=expenses-app
spring.datasource.url=jdbc:mysql://mysql:3306/expense_tracker
spring.datasource.username=root
spring.datasource.password=test@123
```
**Reading this carefully tells you:**
* The MySQL container's name must be `mysql`.
* The MySQL port used is `3306`.
* The database name must be `expense_tracker`.
* The application's own name is `expenses_app`.
* Username: `root`, Password: `test@123`.

## Step 4 — Write the Multi-stage Dockerfile

This app needs **Maven** (to build/compile) plus **Java (JDK)** to run the resulting `.jar` file — a perfect use case for the multi-stage build technique from Chapter 10.

```bash
rm Dockerfile docker-compose.yml   # remove any existing ones, we build from scratch
nano Dockerfile
```

```dockerfile
# ---------- Stage 1: Build the JAR using Maven ----------
FROM maven:3.8.3-openjdk-17 AS builder

WORKDIR /app

COPY . .

RUN mvn clean install -DskipTests=true

# ---------- Stage 2: Run the JAR using a lightweight JDK image ----------
FROM openjdk:17-alpine

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

CMD ["java", "-jar", "app.jar"]
```

**Explanation:**

| Line | Meaning |
|---|---|
| `FROM maven:3.8.3-openjdk-17 AS builder` | Stage 1: A base image that has BOTH Maven (version 3.8.3) AND OpenJDK 17 pre-installed — needed to compile the Spring Boot project into a runnable `.jar` file. |
| `WORKDIR /app` | Sets up the working directory for the build stage. |
| `COPY . .` | Copies your entire project source code into the build stage. |
| `RUN mvn clean install -DskipTests=true` | Runs Maven's build command, producing an executable `.jar` file inside a `target/` folder. `-DskipTests=true` skips running the project's test cases (in this project, the test files were left empty by the developer, so skipping them avoids unnecessary build failures/time). |
| `FROM openjdk:17-alpine` | Stage 2: A much smaller image containing ONLY the Java Runtime needed to RUN a `.jar` file — Maven is not needed here anymore. |
| `COPY --from=builder /app/target/*.jar app.jar` | Copies ONLY the compiled `.jar` file from Stage 1's `target/` folder into the final small image, renaming it `app.jar`. |
| `CMD ["java", "-jar", "app.jar"]` | Runs the compiled application using Java's built-in `-jar` execution mode. |

**Save and exit.**

## Step 5 — Build and Test the Image Standalone
```bash
docker build -t expenses-app .
```
**Expected output:** Maven downloads dependencies, compiles the code, prints `BUILD SUCCESS`, then Docker finishes building the small final image.

**Verify size:**
```bash
docker images
```
**Expected result:** `expenses-app` should be quite small (well under 400 MB) thanks to the multi-stage build — proving the same size-reduction concept from Chapter 10 applies here too.

## Step 6 — Write docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
version: "3.8"

services:
  java_app:
    build:
      context: .
    container_name: expenses_app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: "jdbc:mysql://mysql_db:3306/expense_tracker?allowPublicKeyRetrieval=true&useSSL=false"
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: test@123
    networks:
      - expenses_app_network
    depends_on:
      mysql_db:
        condition: service_healthy
    restart: always

  mysql_db:
    image: mysql:latest
    container_name: mysql_db
    environment:
      MYSQL_ROOT_PASSWORD: test@123
      MYSQL_DATABASE: expense_tracker
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - expenses_app_network
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-ptest@123"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s
    restart: always

volumes:
  mysql_data:

networks:
  expenses_app_network:
```

**Critical concept — Converting `application.properties` values into environment variables:**
Spring Boot has a special rule: any property in `application.properties` written as `spring.datasource.url` can be OVERRIDDEN by setting an environment variable with the SAME name, but written in **UPPERCASE with dots replaced by underscores**:
```text
spring.datasource.url        →  SPRING_DATASOURCE_URL
spring.datasource.username   →  SPRING_DATASOURCE_USERNAME
spring.datasource.password   →  SPRING_DATASOURCE_PASSWORD
```
This is a standard Spring Boot / Maven convention — you don't need to memorize it, just remember the pattern: **uppercase + underscores instead of dots.**

## Step 7 — Build and Run
```bash
docker compose up -d --build
```

**Verify:**
```bash
docker ps
```
**Expected result:** Both `expenses_app` and `mysql_db` containers running.

## Step 8 — Open Port and Test
1. AWS Security Group → Inbound rules → Add rule → Custom TCP, port **8080**, source Anywhere.
2. Browse to:
```text
http://<YOUR_EC2_PUBLIC_IP>:8080
```
3. Sign up, log in, and add an expense entry (e.g., category "Entertainment," amount "850," description).

**Expected result:** The Expense Tracker app loads and successfully saves data.

## The Real Debugging Journey (Important Learning — Read Carefully)

While building this project, a real connection error was hit and solved. Understanding this teaches you real-world Spring Boot + MySQL debugging.

**The error encountered:**
```text
Communications link failure / Access denied / could not connect to MySQL
```
even though the container name, port, username, and password all looked correct in the Compose file.

**Root cause found:**
Modern versions of the MySQL Docker image use a security feature that, by default, **blocks retrieving the public key over an insecure connection** unless explicitly told it's allowed. Spring Boot's JDBC connection string needed an additional parameter:
```text
allowPublicKeyRetrieval=true
```

**The fix:**
Update the `SPRING_DATASOURCE_URL` value to include this parameter (as already shown correctly in Step 6 above):
```text
jdbc:mysql://mysql_db:3306/expense_tracker?allowPublicKeyRetrieval=true&useSSL=false
```
- `allowPublicKeyRetrieval=true` → Allows the Java MySQL driver to retrieve the server's public key when needed for authentication.
- `useSSL=false` → Disables SSL enforcement for this simple learning setup (in production, you would configure proper SSL instead of disabling it).

**Lesson:** When your Spring Boot app cannot connect to MySQL and everything else (container name, network, credentials) looks correct, always check if `allowPublicKeyRetrieval=true` is needed in the JDBC URL — this is a very common, easy-to-miss requirement with newer MySQL versions.

## Troubleshooting

### Error 1
```text
Public Key Retrieval is not allowed
```
**Reason:** Explained above.
**Fix:** Add `?allowPublicKeyRetrieval=true&useSSL=false` to the end of your `SPRING_DATASOURCE_URL`.

### Error 2
```text
Unknown database 'expense_tracker'
```
**Reason:** `MYSQL_DATABASE` on the `mysql_db` service does not match the database name inside the `SPRING_DATASOURCE_URL` on the `java_app` service.

**Fix:** Make sure both values match exactly (`expense_tracker` in both places).

### Error 3
```text
container name "/mysql_db" is already in use
```
**Reason:** An old container from a previous test run wasn't removed.

**Fix:**
```bash
docker ps -a
docker rm <old_container_id>
docker compose up -d --build
```

### Error 4
Application immediately exits with `no active profile set`.
**Reason:** Normal Spring Boot startup log message, not necessarily an error — it appears when no explicit Spring "profile" (like `dev`/`prod`) is configured, and the app just proceeds with defaults. If the app still starts fine afterward, this is not the actual failure — keep checking the logs further down for the real error.

**Fix:** Read the FULL logs (`docker logs <container_id>`) rather than stopping at the first unusual-looking line.

## Cleanup
```bash
docker compose down
```
To also remove the data volume:
```bash
docker compose down -v
```

## Final Result
* A two-tier Spring Boot + MySQL Expense Tracker application runs fully via Docker Compose.
* The Dockerfile uses a multi-stage build (Maven builder stage + slim OpenJDK runtime stage), keeping the final image small.
* You understand how to convert `application.properties` keys into Docker environment variables (`spring.datasource.url` → `SPRING_DATASOURCE_URL`).
* You understand and fixed a real-world MySQL connection issue using `allowPublicKeyRetrieval=true`.
* The app is accessible at `http://<YOUR_EC2_PUBLIC_IP>:8080`, and data is saved and confirmed working.

## What's Next
Chapter 14 covers mapping a real domain name (instead of a raw IP address) to your running application, using DNS A records.
