#  LIQUIBASE + JENKINS CI/CD PIPELINE (ENTERPRISE POC – COMPLETE GUIDE)


# 1.  Objective

This project demonstrates a **Database DevOps CI/CD pipeline** where:

* Database schema is version-controlled using Liquibase
* Changes are automatically deployed via Jenkins
* PostgreSQL acts as the target database
* Every change is tracked, auditable, and rollback-safe


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
Excellent — this file is the:

# MASTER CHANGELOG FILE

This is one of the MOST IMPORTANT concepts in Liquibase architecture.

It acts like:

* Jenkins pipeline controller
* Terraform root module
* Kubernetes master manifest
* Main entry point for DB migrations

---

# COMPLETE FILE

```xml id="v0pjlwm"
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
           http://www.liquibase.org/xml/ns/dbchangelog
           http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <include file="001-init.xml" relativeToChangelogFile="true"/>
    <include file="002-add-email.xml" relativeToChangelogFile="true"/>

</databaseChangeLog>
```

---

# BIG PICTURE FIRST

This file itself does NOT:

* create tables
* add columns
* execute SQL

Instead it says:

# “Run these migration files in this order”

---

# WHAT THIS FILE REALLY IS

Think of it as:

# DATABASE DEPLOYMENT CONTROLLER

---

# ANALOGY WITH JENKINS

Jenkinsfile:

```groovy id="mgvq90"
stage('Build')
stage('Test')
stage('Deploy')
```

Master changelog:

```xml id="8or8y5"
include 001-init.xml
include 002-add-email.xml
```

Same orchestration concept.

---

# WHY MASTER FILE EXISTS

Imagine enterprise project with:

```text id="k9fdlq"
001-users.xml
002-orders.xml
003-payments.xml
004-indexes.xml
005-triggers.xml
006-analytics.xml
```

Without master file:

You would manually run every file.

That becomes chaos.

Master changelog centralizes execution.

---

# STEP-BY-STEP BREAKDOWN

---

# 1. ROOT SECTION

```xml id="1k4e78"
<databaseChangeLog>
```

Meaning:

# This file contains migration orchestration instructions

Same root container as before.

---

# 2. XML NAMESPACE + XSD

```xml id="n97jyu"
xmlns="..."
xmlns:xsi="..."
xsi:schemaLocation="..."
```

Same purpose:

* Liquibase syntax support
* XML validation
* rule enforcement
* schema compatibility

---

# 3. INCLUDE TAG

NOW the most important part.

```xml id="7txff6"
<include file="001-init.xml"
         relativeToChangelogFile="true"/>
```

Meaning:

# Load and execute another Liquibase file

---

# WHAT HAPPENS INTERNALLY

Liquibase reads:

```text id="3dbmkx"
db.changelog-master.xml
```

Then sees:

```xml id="pn0izd"
<include file="001-init.xml"/>
```

Then:

```text id="3qic1n"
Open 001-init.xml
Read all changesets
Execute them
```

---

# EXECUTION ORDER IS CRITICAL

Liquibase processes includes sequentially.

So:

```xml id="6y7d1d"
001-init.xml
002-add-email.xml
```

means:

```text id="n1z6m9"
STEP 1:
Create users table

STEP 2:
Add email column
```

---

# WHY ORDER MATTERS

Because:

```text id="1yc2tt"
email column cannot be added
before users table exists
```

If reversed:

```xml id="vkl04c"
002-add-email.xml
001-init.xml
```

Liquibase would fail:

```text id="26r4it"
ERROR:
table users does not exist
```

---

# 4. relativeToChangelogFile="true"

VERY IMPORTANT.

```xml id="y8lk0x"
relativeToChangelogFile="true"
```

This tells Liquibase:

# Resolve file path relative to current file location

---

# EXAMPLE DIRECTORY

Suppose:

```text id="x0m0jl"
db/changelog/
 ├── db.changelog-master.xml
 ├── 001-init.xml
 └── 002-add-email.xml
```

Liquibase understands:

```text id="4egjlwm"
Master file and included files are in same folder
```

---

# WITHOUT THIS OPTION

Liquibase may search from:

* execution directory
* classpath
* absolute path

leading to path confusion.

---

# ENTERPRISE BEST PRACTICE

Always use:

```xml id="y5o7gl"
relativeToChangelogFile="true"
```

for portability across:

* Jenkins agents
* Docker containers
* Kubernetes pods
* local machines

---

# WHAT LIQUIBASE DOES INTERNALLY

When you run:

```bash id="l2wv1y"
liquibase update
```

Liquibase execution flow:

```text id="m11msl"
1. Read master changelog
2. Read include #1
3. Parse changesets
4. Read include #2
5. Parse changesets
6. Build execution graph
7. Check DATABASECHANGELOG
8. Execute only new changes
```

---

# IMPORTANT UNDERSTANDING

Liquibase does NOT execute files blindly.

It executes:

# CHANGESETS

inside included files.

---

# COMPLETE FLOW

---

# FILE 1

```xml id="0gm5d7"
001-init.xml
```

contains:

```xml id="99ekyx"
<changeSet id="1">
```

Creates:

```text id="c0vdg9"
users table
```

---

# FILE 2

```xml id="3q4msk"
002-add-email.xml
```

contains:

```xml id="dl2sq4"
<changeSet id="2">
```

Adds:

```text id="l9i2i0"
email column
```

---

# MASTER FILE CONTROLS BOTH

```text id="mgcn9k"
Master file
↓
includes child changelog files
↓
Liquibase executes all changesets
```

---

# WHY THIS DESIGN IS POWERFUL

This architecture enables:

| Benefit            | Explanation                              |
| ------------------ | ---------------------------------------- |
| modular migrations | separate files                           |
| team collaboration | different developers own different files |
| ordered execution  | deterministic deployments                |
| CI/CD integration  | single entry point                       |
| scalability        | supports thousands of migrations         |

---

# REAL ENTERPRISE STRUCTURE

Large companies may use:

```text id="4p7nvc"
db/changelog/
 ├── master.xml
 ├── release-1/
 │    ├── users.xml
 │    ├── orders.xml
 │    └── payments.xml
 ├── release-2/
 │    ├── analytics.xml
 │    └── reporting.xml
```

Master file orchestrates everything.

---

# HOW JENKINS USES THIS

Your Jenkins pipeline runs:

```bash id="wwnfe1"
liquibase update
```

Liquibase reads:

```properties id="8b8xpw"
changeLogFile=db/changelog/db.changelog-master.xml
```

This tells Liquibase:

# Start execution from master changelog

Then Liquibase recursively loads all included files.

---

# THIS IS SIMILAR TO

| Technology     | Equivalent              |
| -------------- | ----------------------- |
| Jenkins        | Jenkinsfile             |
| Terraform      | main.tf                 |
| Kubernetes     | kustomization.yaml      |
| Maven          | pom.xml                 |
| Docker Compose | docker-compose.yml      |
| Liquibase      | db.changelog-master.xml |

---

# WHAT DATABASE SEES

Database never sees XML directly.

Liquibase converts included migrations into SQL.

Final SQL sequence becomes:

```sql id="kudjlwm"
CREATE TABLE users (...);

ALTER TABLE users
ADD COLUMN email VARCHAR(255);
```

---

# IMPORTANT ENTERPRISE CONCEPT

This master file creates:

# deterministic database deployment order

Meaning:

Every environment gets same schema evolution:

```text id="2mravj"
DEV
→ TEST
→ STAGING
→ PROD
```

No accidental differences.

---

# FINAL SIMPLE UNDERSTANDING

This master changelog file is the central Liquibase orchestration file that controls the order and inclusion of all database migration files, allowing Liquibase and Jenkins pipelines to execute versioned schema changes in a consistent, scalable, and automated manner across environments.

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
Perfect — this is the CORE of Liquibase.

You are looking at a complete Liquibase migration file.

I’ll explain it like an enterprise database engineer would.

---

# COMPLETE FILE

```xml id="4b8r4n"
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
           http://www.liquibase.org/xml/ns/dbchangelog
           http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <changeSet id="1" author="dev">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="name" type="VARCHAR(100)"/>
        </createTable>
    </changeSet>

</databaseChangeLog>
```

---

# BIG PICTURE FIRST

This file says:

```text id="4ji25d"
Create a database table called users
with:
- id column
- name column
```

Liquibase reads this XML and converts it into SQL.

---

# STEP-BY-STEP BREAKDOWN

---

# 1. ROOT TAG

```xml id="8gl91o"
<databaseChangeLog>
```

This is the:

# ROOT CONTAINER

Meaning:

```text id="0zx71f"
"This file contains database changes"
```

Every Liquibase file starts with this.

Think of it like:

| Technology | Root Element          |
| ---------- | --------------------- |
| HTML       | `<html>`              |
| Kubernetes | `apiVersion`          |
| Jenkins    | `pipeline {}`         |
| Liquibase  | `<databaseChangeLog>` |

---

# 2. XML NAMESPACE

```xml id="sx9bmc"
xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
```

This tells XML parser:

# “Use Liquibase tags and rules”

Without this:

Liquibase would not understand tags like:

```xml id="2mzw6q"
<changeSet>
<createTable>
<column>
```

---

# SIMPLE ANALOGY

Like importing a library in programming:

Python:

```python id="sqw8b7"
import pandas
```

Java:

```java id="k0nlnm"
import java.util.*
```

Similarly:

```xml id="aj8f8w"
xmlns="..."
```

imports Liquibase XML vocabulary.

---

# 3. XMLSCHEMA INSTANCE

```xml id="dh69my"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
```

This enables:

# XML validation support

Meaning:

Liquibase can validate:

* correct syntax
* valid tags
* valid attributes
* allowed structure

---

# 4. SCHEMA LOCATION

```xml id="crw65w"
xsi:schemaLocation="
   http://www.liquibase.org/xml/ns/dbchangelog
   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd"
```

This is VERY IMPORTANT.

It tells Liquibase:

# Which XML rulebook/version to use

---

# WHAT IS XSD?

XSD means:

# XML Schema Definition

Think of it like:

```text id="ymvq9m"
Grammar rules for XML
```

It defines:

* which tags are valid
* nesting rules
* allowed attributes
* datatype validation

---

# ANALOGY

Like API schema validation.

Example:

```json id="wb1zvi"
{
  "name": "Anuj",
  "age": 25
}
```

Schema checks:

```text id="9z4fzj"
age must be integer
name must be string
```

Similarly Liquibase validates your XML.

---

# THIS PART

```xml id="gdy7g4"
dbchangelog-4.3.xsd
```

means:

# Use Liquibase 4.3 XML specification

Different Liquibase versions may support different tags/features.

---

# 5. CHANGESET

Now the MOST IMPORTANT part.

```xml id="v7g4hq"
<changeSet id="1" author="dev">
```

This is:

# ONE DATABASE MIGRATION UNIT

Think of changeset as:

```text id="35tmvh"
One Git commit for database
```

---

# WHY CHANGESET EXISTS

Liquibase tracks execution using:

```text id="dz0xg6"
id + author + filename
```

Example:

| id | author | file         |
| -- | ------ | ------------ |
| 1  | dev    | 001-init.xml |

Stored inside:

# DATABASECHANGELOG table

---

# PURPOSE OF ID

```xml id="6ydvcl"
id="1"
```

Unique migration identifier.

Examples:

```xml id="wyi04x"
id="create-users-table"
id="20260518-01"
id="v1-users"
```

Enterprise teams often use timestamps:

```xml id="l7w5ow"
id="202605181200"
```

to avoid conflicts.

---

# PURPOSE OF AUTHOR

```xml id="p2z2c7"
author="dev"
```

Tracks who created migration.

Useful for:

* auditing
* debugging
* enterprise compliance

---

# 6. CREATE TABLE

```xml id="3q4a2x"
<createTable tableName="users">
```

Meaning:

# Create SQL table called users

Liquibase converts this into DB-specific SQL.

---

# FOR POSTGRESQL

Liquibase internally generates:

```sql id="w1q59h"
CREATE TABLE users (
 ...
);
```

---

# WHY THIS IS POWERFUL

You do NOT manually write vendor-specific SQL.

Liquibase adapts automatically.

For:

* PostgreSQL
* Oracle
* MySQL
* SQL Server

---

# 7. COLUMN DEFINITION

Now inside table:

```xml id="evokpb"
<column name="id" type="BIGINT" autoIncrement="true">
```

This defines:

| Property      | Meaning               |
| ------------- | --------------------- |
| name          | column name           |
| type          | database datatype     |
| autoIncrement | auto-generated values |

---

# WHAT IS BIGINT

```text id="qg1g51"
64-bit integer
```

Used for IDs.

PostgreSQL equivalent:

```sql id="wv90ri"
BIGSERIAL
```

---

# AUTOINCREMENT

```xml id="e9e7o9"
autoIncrement="true"
```

Means:

# Automatically generate IDs

Example inserts:

```sql id="v4y7m0"
INSERT INTO users(name)
VALUES ('Anuj');
```

DB automatically creates:

```text id="px25jv"
id = 1
id = 2
id = 3
```

---

# GENERATED SQL

Liquibase may generate:

```sql id="mjlwm9"
id BIGSERIAL
```

for PostgreSQL.

---

# 8. CONSTRAINTS

Inside column:

```xml id="3bch9n"
<constraints primaryKey="true"/>
```

Meaning:

# Make this column PRIMARY KEY

---

# PRIMARY KEY PURPOSE

Primary key guarantees:

| Feature    | Meaning            |
| ---------- | ------------------ |
| uniqueness | no duplicate IDs   |
| indexing   | faster lookup      |
| identity   | row identification |

---

# GENERATED SQL

Liquibase creates:

```sql id="ygr8lj"
PRIMARY KEY (id)
```

---

# 9. SECOND COLUMN

```xml id="6j5yjj"
<column name="name" type="VARCHAR(100)"/>
```

Creates:

# name column

Datatype:

```text id="5j7z7v"
VARCHAR(100)
```

Meaning:

```text id="t0gvc4"
Variable length text
Maximum 100 characters
```

---

# GENERATED SQL

```sql id="gzbh1h"
name VARCHAR(100)
```

---

# FINAL SQL GENERATED BY LIQUIBASE

Liquibase internally converts everything into:

```sql id="n6h5o5"
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100)
);
```

---

# 10. CLOSING TAGS

```xml id="9g9cye"
</createTable>
</changeSet>
</databaseChangeLog>
```

These simply close XML hierarchy.

---

# EXECUTION FLOW INSIDE LIQUIBASE

When you run:

```bash id="u0fe7l"
liquibase update
```

Liquibase does:

```text id="n5xkzq"
1. Read XML file
2. Validate using XSD
3. Parse changesets
4. Connect to PostgreSQL via JDBC
5. Generate SQL
6. Execute SQL
7. Update DATABASECHANGELOG
```

---

# WHAT GETS STORED IN DATABASECHANGELOG

After execution:

| id | author | filename     |
| -- | ------ | ------------ |
| 1  | dev    | 001-init.xml |

This prevents rerunning same migration.

---

# WHY THIS IS ENTERPRISE-GRADE

Instead of random SQL scripts:

```text id="8yl8mf"
final.sql
new-final.sql
latest-final-v2.sql
```

you get:

# deterministic versioned schema evolution

---

# VERY IMPORTANT CONCEPT

Liquibase XML is:

# Declarative

Meaning:

You describe:

```text id="nkt9s5"
WHAT database should become
```

NOT:

```text id="s2o3qq"
HOW to execute SQL manually
```

Liquibase handles execution safely.

---

# REAL ENTERPRISE BENEFITS

| Problem              | Liquibase Solution    |
| -------------------- | --------------------- |
| manual SQL mistakes  | structured migrations |
| no audit history     | DATABASECHANGELOG     |
| rollback issues      | rollback blocks       |
| schema inconsistency | version-controlled DB |
| multi-env deployment | automated migrations  |

---

# FINAL SIMPLE UNDERSTANDING

This XML file is a structured database migration definition where Liquibase validates the schema, converts it into PostgreSQL SQL commands, executes the changes safely, and records execution history for future CI/CD deployments.

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
Excellent — this file demonstrates one of the MOST IMPORTANT concepts in database DevOps:

# Schema Evolution

Your first file created the database structure.

This second file modifies an existing database safely.

---

# COMPLETE FILE

```xml id="zjlwm6"
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
           http://www.liquibase.org/xml/ns/dbchangelog
           http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <changeSet id="2" author="dev">

        <addColumn tableName="users">
            <column name="email" type="VARCHAR(255)"/>
        </addColumn>

        <rollback>
            <dropColumn tableName="users" columnName="email"/>
        </rollback>

    </changeSet>

</databaseChangeLog>
```

---

# BIG PICTURE FIRST

This file says:

```text id="mj6ji5"
Modify existing users table
and add a new column called email
```

AND:

```text id="ntrv74"
If needed,
remove the column during rollback
```

---

# WHY THIS EXISTS

In real companies:

Requirements constantly change.

Example:

Initial requirement:

```text id="nzb3fr"
Store user names only
```

Later business says:

```text id="rrk9v3"
We now need email addresses
```

Instead of manually altering production DB:

Liquibase tracks schema evolution safely.

---

# STEP-BY-STEP EXPLANATION

---

# 1. ROOT SECTION

```xml id="j6c2oz"
<databaseChangeLog>
```

Same meaning as before.

This file contains:

# database modifications

---

# 2. XML NAMESPACE + XSD

```xml id="dr8b0v"
xmlns="..."
xmlns:xsi="..."
xsi:schemaLocation="..."
```

Purpose:

* XML validation
* Liquibase syntax rules
* supported tag verification

Same as previous file.

---

# 3. CHANGESET

```xml id="4h5o8o"
<changeSet id="2" author="dev">
```

This is:

# Database Migration Version 2

Your previous migration was:

```text id="bimv5v"
id=1 → create users table
```

Now:

```text id="2vzjlwm"
id=2 → modify users table
```

---

# VERY IMPORTANT

Liquibase stores this in:

# DATABASECHANGELOG

Example:

| id | author | status   |
| -- | ------ | -------- |
| 1  | dev    | executed |
| 2  | dev    | executed |

---

# WHY CHANGESET IDs MATTER

Liquibase checks:

```text id="lq1mnh"
Has id=2 already executed?
```

If YES:

```text id="5dxr0j"
skip it
```

If NO:

```text id="wwy3k7"
execute migration
```

---

# THIS PREVENTS

* duplicate migrations
* rerunning ALTER statements
* accidental schema corruption

---

# 4. ADD COLUMN

Now the main operation:

```xml id="b44f11"
<addColumn tableName="users">
```

Meaning:

# Modify existing table

NOT create new table.

Liquibase internally generates:

```sql id="m05zzk"
ALTER TABLE users
ADD COLUMN ...
```

---

# TARGET TABLE

```xml id="9y2tlo"
tableName="users"
```

Means:

```text id="y5rwm4"
Modify users table
```

which was created in:

```text id="v5hl0r"
001-init.xml
```

---

# 5. COLUMN DEFINITION

```xml id="28x6tx"
<column name="email" type="VARCHAR(255)"/>
```

Creates:

# New column named email

Datatype:

```text id="1jv9cx"
VARCHAR(255)
```

Meaning:

```text id="phlq6j"
Variable-length string
Maximum 255 characters
```

Typically used for:

* emails
* usernames
* URLs
* text fields

---

# GENERATED SQL

Liquibase converts this into:

```sql id="b81ncu"
ALTER TABLE users
ADD COLUMN email VARCHAR(255);
```

---

# WHAT HAPPENS TO EXISTING DATA?

Suppose current table:

| id | name |
| -- | ---- |
| 1  | Anuj |
| 2  | John |

After migration:

| id | name | email |
| -- | ---- | ----- |
| 1  | Anuj | NULL  |
| 2  | John | NULL  |

Existing rows remain safe.

New column added with NULL values.

---

# 6. ROLLBACK SECTION

NOW the MOST IMPORTANT enterprise feature.

```xml id="glxq02"
<rollback>
```

This defines:

# How to reverse the migration

---

# WHY ROLLBACK MATTERS

Imagine production deployment fails:

```text id="ig4q9m"
App crashes
API incompatible
Bug discovered
```

You must safely revert database changes.

Without rollback:

```text id="e8xy5p"
manual emergency SQL
high risk
downtime
```

With Liquibase:

```bash id="k7a09y"
liquibase rollbackCount 1
```

safe automated reversal.

---

# DROP COLUMN

Inside rollback:

```xml id="0tt09r"
<dropColumn tableName="users" columnName="email"/>
```

Liquibase generates:

```sql id="cybgrq"
ALTER TABLE users
DROP COLUMN email;
```

---

# COMPLETE EXECUTION FLOW

When you run:

```bash id="yd4xzk"
liquibase update
```

Liquibase does:

```text id="yzs79o"
1. Read changelog
2. Check DATABASECHANGELOG
3. Find unexecuted changeset id=2
4. Generate SQL
5. Execute ALTER TABLE
6. Record migration history
```

---

# ROLLBACK FLOW

When you run:

```bash id="kcd1t5"
liquibase rollbackCount 1
```

Liquibase does:

```text id="b6k0vb"
1. Find last executed changeset
2. Read rollback block
3. Generate reverse SQL
4. Execute DROP COLUMN
5. Remove rollback record
```

---

# BEFORE VS AFTER

---

# BEFORE MIGRATION

```text id="5i3v8r"
users
 ├── id
 └── name
```

---

# AFTER MIGRATION

```text id="m8ed9d"
users
 ├── id
 ├── name
 └── email
```

---

# AFTER ROLLBACK

```text id="r3p4vh"
users
 ├── id
 └── name
```

email removed safely.

---

# WHY THIS IS ENTERPRISE-GRADE

In enterprise systems:

DB changes must be:

| Requirement | Why                |
| ----------- | ------------------ |
| auditable   | compliance         |
| reversible  | disaster recovery  |
| repeatable  | CI/CD              |
| automated   | DevOps             |
| versioned   | team collaboration |

Liquibase solves all of these.

---

# IMPORTANT CONCEPT

This file is NOT:

```text id="l4q5tb"
direct SQL script
```

It is:

# Declarative migration definition

You describe:

```text id="u75c1i"
What change should happen
```

Liquibase decides:

```text id="7l5z7w"
How to execute safely on PostgreSQL
```

---

# WHY XML IS POWERFUL HERE

Liquibase understands high-level operations:

```xml id="wyc1y2"
<addColumn>
<dropColumn>
<createTable>
<createIndex>
```

instead of raw SQL.

This enables:

* DB portability
* validation
* rollback automation
* dependency management

---

# REAL ENTERPRISE SCENARIO

Developer creates:

```text id="9s17je"
002-add-email.xml
```

Pushes to GitHub.

Jenkins pipeline runs:

```bash id="r0w7k4"
liquibase update
```

Production DB evolves automatically.

If issue occurs:

```bash id="7e7y11"
liquibase rollbackCount 1
```

Database safely returns to previous state.

---

# FINAL SIMPLE UNDERSTANDING

This Liquibase file represents database schema version 2, where an existing users table is modified by adding an email column, while also defining a rollback strategy so the schema change can be safely reversed during CI/CD deployments or production failures.

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

# 8. ▶️ LIQUIBASE COMMANDS

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

# 11. 🚀 JENKINS CI/CD PIPELINE (UPDATED FINAL VERSION)

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
