# 🚀 LIQUIBASE + JENKINS — FULL CI/CD PIPELINE (NO DOCKER)

---

# 1. Objective

This setup automates:

* Database schema versioning
* Schema changes using Liquibase
* CI/CD execution using Jenkins
* Safe deployment to PostgreSQL
* Audit tracking + rollback support

---

# 2. Tools Used

| Tool       | Purpose                          |
| ---------- | -------------------------------- |
| Liquibase  | Database versioning & migrations |
| Jenkins    | Pipeline automation              |
| PostgreSQL | Target database                  |

---

# 3. Project Structure

Run this:

```bash
mkdir liquibase-jenkins-pipeline
cd liquibase-jenkins-pipeline

mkdir -p db/changelog
mkdir jenkins
mkdir scripts
```

---

# 4. Files to Create

```bash
touch db/changelog/db.changelog-master.xml
touch db/changelog/001-init.xml
touch db/changelog/002-add-email.xml
touch liquibase.properties
touch Jenkinsfile
```

---

# 5. Database Setup (Manual PostgreSQL)

Install PostgreSQL:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

Create DB:

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE appdb;
CREATE USER admin WITH PASSWORD 'admin';
GRANT ALL PRIVILEGES ON DATABASE appdb TO admin;
```

---

# 6. Liquibase Master File

```bash
cat << 'EOF' > db/changelog/db.changelog-master.xml
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
    http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.0.xsd">

    <include file="001-init.xml" path="db/changelog"/>
    <include file="002-add-email.xml" path="db/changelog"/>

</databaseChangeLog>
EOF
```

---

# 7. Initial Schema Migration

```bash
cat << 'EOF' > db/changelog/001-init.xml
<databaseChangeLog>
    <changeSet id="1" author="dev">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="name" type="VARCHAR(100)"/>
        </createTable>
    </changeSet>
</databaseChangeLog>
EOF
```

---

# 8. Add Column Migration

```bash
cat << 'EOF' > db/changelog/002-add-email.xml
<databaseChangeLog>
    <changeSet id="2" author="dev">
        <addColumn tableName="users">
            <column name="email" type="VARCHAR(255)"/>
        </addColumn>
    </changeSet>
</databaseChangeLog>
EOF
```

---

# 9. Liquibase Properties File

```bash
cat << 'EOF' > liquibase.properties
changeLogFile=db/changelog/db.changelog-master.xml
url=jdbc:postgresql://localhost:5432/appdb
username=admin
password=admin
driver=org.postgresql.Driver
EOF
```

---

# 10. Install Liquibase (Manual)

```bash
wget https://github.com/liquibase/liquibase/releases/download/v4.25.0/liquibase-4.25.0.tar.gz

tar -xvzf liquibase-4.25.0.tar.gz

sudo mv liquibase /opt/liquibase
export PATH=$PATH:/opt/liquibase
```

Test:

```bash
liquibase --version
```

---

# 11. Run Liquibase Manually (Test)

```bash
liquibase update
```

👉 This applies migrations

---

# 12. Jenkins Pipeline (CI/CD)

```bash
cat << 'EOF' > Jenkinsfile
pipeline {
    agent any

    environment {
        DB_URL = 'jdbc:postgresql://localhost:5432/appdb'
        DB_USER = 'admin'
        DB_PASS = 'admin'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/your-repo/liquibase-demo.git'
            }
        }

        stage('Install Liquibase') {
            steps {
                sh '''
                    wget https://github.com/liquibase/liquibase/releases/download/v4.25.0/liquibase-4.25.0.tar.gz
                    tar -xvzf liquibase-4.25.0.tar.gz
                    export PATH=$PATH:$WORKSPACE/liquibase
                '''
            }
        }

        stage('Validate Changes') {
            steps {
                sh './liquibase validate --defaultsFile=liquibase.properties'
            }
        }

        stage('Run Migrations') {
            steps {
                sh './liquibase update --defaultsFile=liquibase.properties'
            }
        }

        stage('Audit Check') {
            steps {
                sh './liquibase history --defaultsFile=liquibase.properties'
            }
        }

        stage('Verify DB') {
            steps {
                sh '''
                    psql -U admin -d appdb -c "SELECT * FROM users;"
                '''
            }
        }
    }
}
EOF
```

---

# 13. CI/CD Flow

```txt
Developer changes XML
        ↓
Git Push
        ↓
Jenkins Trigger
        ↓
Liquibase validate
        ↓
Liquibase update
        ↓
PostgreSQL updated
        ↓
Audit stored in DATABASECHANGELOG
```

---

# 14. Liquibase Audit Tables

Automatically created:

### 1. DATABASECHANGELOG

Stores:

* changeSet ID
* author
* execution time

---

### 2. DATABASECHANGELOGLOCK

Prevents multiple pipelines running simultaneously.

---

# 15. Rollback (Enterprise Feature)

You can rollback:

```bash
liquibase rollbackCount 1
```

or tag-based rollback:

```bash
liquibase rollback myTag
```

---

# 16. What This Pipeline Achieves

✔ Fully automated DB migrations
✔ No manual SQL execution
✔ Version-controlled database
✔ Jenkins CI/CD integration
✔ Audit logging
✔ Rollback support

---

# 17. Enterprise-Level Architecture

```txt
Developer
   ↓
Git Repository
   ↓
Jenkins Pipeline
   ↓
Liquibase Engine
   ↓
PostgreSQL Database
   ↓
Audit Tables (history tracking)
```

---

# 18. Why This is Enterprise Ready

✔ Used in banks & SaaS companies
✔ Strong audit trail
✔ Safe rollback support
✔ Controlled schema evolution
✔ CI/CD automation
✔ Production-safe migrations

---

# 19. Viva One-Liner (IMPORTANT)

> Liquibase + Jenkins provides an enterprise-grade database DevOps pipeline where schema changes are version-controlled using changelogs and automatically deployed through CI/CD with full auditing and rollback support.

Yes — this is the most important part for viva and interviews. I’ll break it down **file by file + what each one actually does in the system**, so you understand the *real working*, not just syntax.

---

# 🧠 Liquibase + Jenkins Project — File-wise Explanation

We built this structure:

```text
liquibase-jenkins-pipeline/
│
├── db/changelog/
│   ├── db.changelog-master.xml
│   ├── 001-init.xml
│   ├── 002-add-email.xml
│
├── liquibase.properties
├── Jenkinsfile
```

Now let’s understand each part.

---

# 📁 1. db/changelog/ (MAIN MIGRATION FOLDER)

This folder is the **heart of Liquibase**.

👉 It contains all database changes written as **versioned changelogs**

Think of it like:

> 📌 “Git commits for your database schema”

---

## 🧾 1.1 db.changelog-master.xml

### 👉 Purpose:

This is the **entry point of Liquibase**

It tells Liquibase:

> “Which migration files should I execute, and in what order?”

---

### 🔧 What it does:

```xml
<include file="001-init.xml"/>
<include file="002-add-email.xml"/>
```

This means:

1. Run `001-init.xml`
2. Then run `002-add-email.xml`

---

### 🧠 Why it is important:

Without this file:

* Liquibase won’t know migrations exist
* No ordering system
* No execution flow

👉 It is the **controller of all migrations**

---

# 🧾 1.2 001-init.xml (FIRST MIGRATION)

### 👉 Purpose:

This file creates the **initial database structure**

---

### What it does:

```xml
<createTable tableName="users">
```

It creates:

| Column | Type        |
| ------ | ----------- |
| id     | Primary Key |
| name   | VARCHAR     |

---

### 🧠 Meaning in real world:

👉 “This is the first version of your database”

Like:

```text
v1.0 → initial database release
```

---

### Why it is important:

* First schema baseline
* All future changes depend on this
* Stored in audit history table

---

# 🧾 1.3 002-add-email.xml (SECOND MIGRATION)

### 👉 Purpose:

This modifies existing schema.

---

### What it does:

```xml
<addColumn tableName="users">
    <column name="email" type="VARCHAR(255)"/>
</addColumn>
```

It adds:

| Table | New Column |
| ----- | ---------- |
| users | email      |

---

### 🧠 Real-world meaning:

👉 “Business requirement changed”

Example:

* Login now requires email
* So DB must evolve

---

### Why important:

* Shows **schema evolution**
* Demonstrates **database versioning**
* Enables rollback tracking

---

# 📁 2. liquibase.properties

### 👉 Purpose:

This is the **connection configuration file**

It tells Liquibase:

> “Where is my database and how do I connect to it?”

---

### What it contains:

```properties
url=jdbc:postgresql://localhost:5432/appdb
username=admin
password=admin
```

---

### 🧠 Real meaning:

| Field         | Meaning              |
| ------------- | -------------------- |
| url           | DB location          |
| username      | DB user              |
| password      | authentication       |
| changelogFile | migration entry file |

---

### Why it is important:

Without this:

* Liquibase cannot connect to DB
* No migrations will run

---

# 📁 3. Jenkinsfile (CI/CD PIPELINE)

### 👉 Purpose:

This file defines the **automation pipeline**

👉 It tells Jenkins:

> “When code changes, automatically update database”

---

## 🧠 Step-by-step explanation

---

## 🔹 Stage 1: Checkout Code

```groovy
git 'https://github.com/your-repo'
```

### Meaning:

* Jenkins pulls latest project code from Git

👉 This ensures:

* latest migrations are always used

---

## 🔹 Stage 2: Install Liquibase

```bash
wget liquibase
```

### Meaning:

* Jenkins installs Liquibase tool inside pipeline

👉 Because Jenkins machine may not have it pre-installed

---

## 🔹 Stage 3: Validate

```bash
liquibase validate
```

### Meaning:

Checks:

* Is XML correct?
* Any broken migration?
* Missing files?

👉 Prevents deployment failures

---

## 🔹 Stage 4: Update (MAIN STEP)

```bash
liquibase update
```

### Meaning:

👉 This is the **actual migration execution**

It:

1. Reads changelog
2. Checks DATABASECHANGELOG table
3. Runs only new migrations
4. Updates database

---

## 🔹 Stage 5: Audit Check

```bash
liquibase history
```

### Meaning:

Shows:

* which migrations ran
* when they ran
* success/failure status

---

## 🔹 Stage 6: Verify DB

```sql
SELECT * FROM users;
```

### Meaning:

Final validation step:

* ensures DB is updated

---

# 📁 4. DATABASECHANGELOG (AUTO GENERATED)

This is NOT manually created.

Liquibase creates it automatically.

---

## 👉 Purpose:

It stores:

| Column       | Meaning         |
| ------------ | --------------- |
| ID           | migration id    |
| AUTHOR       | who wrote it    |
| DATEEXECUTED | when it ran     |
| STATUS       | success/failure |

---

## 🧠 Why important:

👉 This is the **brain of Liquibase**

It ensures:

* no duplicate execution
* no re-running old migrations
* full audit trail

---

# 📁 5. DATABASECHANGELOGLOCK (AUTO GENERATED)

### 👉 Purpose:

Prevents multiple deployments at same time.

---

## Example:

If 2 Jenkins jobs run:

* Job A runs migration
* Job B waits

👉 Avoids:

* DB corruption
* race conditions

---

# 🧠 FULL SYSTEM FLOW (VERY IMPORTANT FOR VIVA)

```text
Developer changes XML
        ↓
Push to Git
        ↓
Jenkins triggered
        ↓
Liquibase validate
        ↓
Liquibase reads changelog
        ↓
Checks DATABASECHANGELOG
        ↓
Runs only new migrations
        ↓
Updates PostgreSQL
        ↓
Stores history + audit
```

---

# 🎯 SIMPLE ONE-LINE UNDERSTANDING

* `001-init.xml` → creates database
* `002-add-email.xml` → modifies database
* `master.xml` → controls execution order
* `properties` → connects to DB
* `Jenkinsfile` → automates everything
* `DATABASECHANGELOG` → tracks everything

---

# 🚀 REAL-WORLD ANALOGY

Think like this:

| Component       | Real World                  |
| --------------- | --------------------------- |
| changelog XML   | Engineering design changes  |
| master file     | project manager             |
| Jenkins         | automation robot            |
| database        | building under construction |
| changelog table | construction logbook        |

---

If you want next, I can also explain:

✔ how rollback actually works internally
✔ how enterprise banks use Liquibase
✔ or give you a **viva Q&A sheet (very high scoring)**
