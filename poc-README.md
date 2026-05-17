#  LIQUIBASE + JENKINS CI/CD PIPELINE (ENTERPRISE POC – COMPLETE GUIDE)


# 1.  Objective

This project demonstrates a **Database DevOps CI/CD pipeline** where:

* Database schema is version-controlled using Liquibase
* Changes are automatically deployed via Jenkins
* PostgreSQL acts as the target database
* Every change is tracked, auditable, and rollback-safe

---

# 2.  Architecture Flow

```
Developer
   ↓
Git Repository
   ↓
Jenkins Pipeline
   ↓
Liquibase Engine (Java)
   ↓
JDBC Driver
   ↓
PostgreSQL Database
   ↓
DATABASECHANGELOG (Audit Table)
```

---

# 3. ⚙️ Prerequisites Installation (FULL SETUP)

---

## 3.1 Install Java (Required for Liquibase + Jenkins)

```bash
sudo apt update -y
sudo apt install openjdk-17-jdk -y
java -version
```

### Why Java?

* Liquibase runs on JVM
* Jenkins is Java-based
* JDBC drivers are Java libraries

---

## 3.2 Install PostgreSQL (DATABASE SERVER)

```bash
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

---

## 3.3 Create Database + User (VERY IMPORTANT)

```bash
sudo -u postgres psql
```

Inside PostgreSQL:

```sql
CREATE DATABASE appdb;
CREATE USER admin WITH PASSWORD 'admin';
GRANT ALL PRIVILEGES ON DATABASE appdb TO admin;
\q
```

---

###  What is happening here?

| Object     | Purpose                           |
| ---------- | --------------------------------- |
| appdb      | Application database              |
| admin      | DB user used by Liquibase         |
| privileges | allows Liquibase to create tables |

---

# 4.  JDBC DRIVER (CRITICAL COMPONENT)

```bash
wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar
sudo mkdir -p /opt/liquibase/lib
sudo mv postgresql-42.7.3.jar /opt/liquibase/lib/
```

---

### Why JDBC is required?

Liquibase cannot talk directly to PostgreSQL.

```
Liquibase → JDBC Driver → PostgreSQL
```

Without JDBC:
No DB connection
No migrations possible

---

# 5. Install Liquibase

```bash
wget https://github.com/liquibase/liquibase/releases/download/v4.25.0/liquibase-4.25.0.tar.gz

tar -xvzf liquibase-4.25.0.tar.gz
sudo mv liquibase /opt/liquibase
export PATH=$PATH:/opt/liquibase
```

---

# 6.  CREATE PROJECT STRUCTURE

```bash
mkdir -p liquibase-jenkins-pipeline/db/changelog
cd liquibase-jenkins-pipeline
```

---

# 7.  CREATE FILES (USING cat << EOF)

---

## 7.1 liquibase.properties (CONFIG FILE)

```bash
cat << 'EOF' > liquibase.properties
changeLogFile=db/changelog/db.changelog-master.xml
url=jdbc:postgresql://localhost:5432/appdb
username=admin
password=admin
driver=org.postgresql.Driver
searchPath=db/changelog
EOF
```

---

###  Purpose

This file defines:

* DB connection (JDBC URL)
* Credentials
* Entry changelog file
* Search path for XML files

---

# 7.2 MASTER CHANGELOG (CONTROLLER FILE)

```bash
cat << 'EOF' > db/changelog/db.changelog-master.xml
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
    http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <include file="001-init.xml"/>
    <include file="002-add-email.xml"/>

</databaseChangeLog>
EOF
```

---

###  Purpose

* Entry point of Liquibase
* Controls execution order
* Acts like pipeline controller

---

# 7.3 001-init.xml (INITIAL DATABASE CREATION)

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

###  What database object is created?

###  Table: `users`

| Column | Type                                 |
| ------ | ------------------------------------ |
| id     | BIGINT (Primary Key, Auto Increment) |
| name   | VARCHAR(100)                         |

---

###  Meaning

This is your:

INITIAL DATABASE VERSION (v1.0)

---

# 7.4 002-add-email.xml (SCHEMA EVOLUTION)

```bash
cat << 'EOF' > db/changelog/002-add-email.xml
<databaseChangeLog>

    <changeSet id="2" author="dev">

        <addColumn tableName="users">
            <column name="email" type="VARCHAR(255)"/>
        </addColumn>

        <rollback>
            <dropColumn tableName="users" columnName="email"/>
        </rollback>

    </changeSet>

</databaseChangeLog>
EOF
```

---

### What changes?

Adds:

| Table | Column |
| ----- | ------ |
| users | email  |

---

### Meaning

Database evolves due to new business requirement

---

# 8.  LIQUIBASE COMMANDS

---

## 8.1 Validate

```bash
liquibase validate --defaultsFile=liquibase.properties
```

✔ Checks XML
✔ Validates file paths
✔ Ensures safe execution

---

## 8.2 Apply Migration

```bash
liquibase update --defaultsFile=liquibase.properties
```

### What happens internally:

```
Read XML
↓
Check DATABASECHANGELOG
↓
Run only new changesets
↓
Update PostgreSQL
↓
Store audit logs
```

---

## 8.3 Check Database

```bash
psql -U admin -d appdb -h localhost
```

Inside:

```sql
\dt
\d users
SELECT * FROM users;
```

---

# 9. DATABASE OBJECTS CREATED (VERY IMPORTANT)

---

## 9.1 users (APPLICATION TABLE)

Created by:

```xml
001-init.xml
```

### Structure:

| Column | Type        |
| ------ | ----------- |
| id     | Primary Key |
| name   | text        |
| email  | added later |

---

## 9.2 DATABASECHANGELOG (AUTO CREATED)

### Purpose:

Stores migration history:

| Field    | Meaning        |
| -------- | -------------- |
| id       | changeset id   |
| author   | developer      |
| date     | execution time |
| checksum | validation     |

This is your AUDIT LOG

---

## 9.3 DATABASECHANGELOGLOCK (AUTO CREATED)

### Purpose:

Prevents:

* parallel deployments
* race conditions

---

# 10. ROLLBACK FEATURE

```bash
liquibase rollbackCount 1 --defaultsFile=liquibase.properties
```

### What happens:

* Removes last executed changeset
* Drops email column (based on rollback block)

---

# 11. JENKINS CI/CD PIPELINE (UPDATED FINAL VERSION)

```bash
cat << 'EOF' > Jenkinsfile
pipeline {
    agent any

    environment {
        LIQUIBASE = "/home/ubuntu/liquibase-home/liquibase-cli"
        PROPS = "liquibase.properties"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://your-repo-url.git'
            }
        }

        stage('Validate Changelog') {
            steps {
                sh "${LIQUIBASE} validate --defaultsFile=${PROPS}"
            }
        }

        stage('Database Connection Check') {
            steps {
                sh "psql -U admin -d appdb -c '\\dt'"
            }
        }

        stage('Run Liquibase Migration') {
            steps {
                sh "${LIQUIBASE} update --defaultsFile=${PROPS}"
            }
        }

        stage('Verify Schema') {
            steps {
                sh "psql -U admin -d appdb -c '\\d users'"
            }
        }

        stage('Show History') {
            steps {
                sh "${LIQUIBASE} history --defaultsFile=${PROPS}"
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS: Database updated"
        }
        failure {
            echo "Pipeline FAILED: Check logs"
        }
    }
}
EOF
```

---

# 12. END-TO-END FLOW

```
1. Developer writes XML
2. Pushes to Git
3. Jenkins triggers pipeline
4. Liquibase validate runs
5. Liquibase update runs
6. PostgreSQL schema updated
7. DATABASECHANGELOG stores history
```

---

# 13. FINAL ONE-LINE UNDERSTANDING

Liquibase is a Java-based database version control tool that uses JDBC to connect to PostgreSQL and applies structured schema changes through CI/CD pipelines like Jenkins while maintaining full audit history and rollback capability.

---

# 14. 🏁 FINAL COMMAND SUMMARY

liquibase validate --defaultsFile=liquibase.properties

<img width="1919" height="803" alt="image" src="https://github.com/user-attachments/assets/048945ec-82e8-4058-98ef-8e8312b6983d" />

liquibase update --defaultsFile=liquibase.properties

<img width="1919" height="845" alt="image" src="https://github.com/user-attachments/assets/d5684e9b-3940-435d-9c69-78add67e2bc3" />

liquibase rollbackCount 1 --defaultsFile=liquibase.properties

<img width="1919" height="832" alt="image" src="https://github.com/user-attachments/assets/ae6d42fb-065e-4ec5-8646-6d2718e7004a" />


psql -U admin -d appdb -h localhost
\dt
\d users

<img width="960" height="443" alt="image" src="https://github.com/user-attachments/assets/b8453965-ce98-43db-8d9f-bfab550192fd" />

