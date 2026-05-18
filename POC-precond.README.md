# LIQUIBASE + JENKINS CI/CD PIPELINE (ENTERPRISE POC – REAL IMPLEMENTATION)

---

# 1. Objective

This project demonstrates a **Database DevOps CI/CD pipeline** using:

* **Liquibase** → Database version control
* **PostgreSQL** → Target database
* **Jenkins** → CI/CD automation
* **XML + SQL changelogs** → Schema & data changes
* **Governance engine** → DML/DDL safety controls
* **Rollback support + audit history**

---

# 2. Architecture Flow

```text
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
DATABASECHANGELOG / LOCK tables
```

---

# 3. Your Actual Project Structure

```text
liquibase-enterprise-poc/
│
├── db/
│   └── changelog/
│       ├── db.changelog-master.xml
│       ├── 001-create-users.xml
│       ├── 001-init.xml
│       ├── 002-add-email.xml
│       ├── 002-seed.xml
│       ├── 003-risky-demo.xml
│       ├── 003-safe-update.xml
│       └── 004-raw-sql-test.sql
```

---

# 4. Key Concepts (DML vs DDL)

## What is DDL?

**DDL (Data Definition Language)** → defines structure

Examples:

* CREATE TABLE
* ALTER TABLE
* DROP TABLE
* CREATE INDEX

👉 Used in schema design

---

## What is DML?

**DML (Data Manipulation Language)** → works on data

Examples:

* INSERT
* UPDATE
* DELETE
* SELECT

👉 Used for data changes inside tables

---

# 5. File-by-File Explanation (YOUR ACTUAL FILES)

---

# 5.1 001-create-users.xml (BASE TABLE)

```xml
<createTable tableName="users">
    <column name="id" type="BIGINT" autoIncrement="true">
        <constraints primaryKey="true"/>
    </column>

    <column name="name" type="VARCHAR(100)"/>
    <column name="contact" type="VARCHAR(20)"/>
</createTable>
```

### Purpose:

Creates base table `users`

---

# 5.2 001-init.xml (SAFE INIT + ROLLBACK)

```xml
<rollback>
    <dropTable tableName="users"/>
</rollback>
```

### Purpose:

* Safe initialization
* Full rollback support for table recreation

---

# 5.3 002-add-email.xml (SCHEMA EVOLUTION)

```xml
<addColumn tableName="users">
    <column name="email" type="VARCHAR(255)"/>
</addColumn>

<rollback>
    <dropColumn tableName="users" columnName="email"/>
</rollback>
```

### Purpose:

* Adds email column
* Supports rollback safely

---

# 5.4 002-seed.xml (DML INSERT)

```xml
<insert tableName="users">
    <column name="name" value="admin"/>
    <column name="email" value="admin@test.com"/>
</insert>
```

### Type:

✔ DML (INSERT operation)

---

# 5.5 003-safe-update.xml (CONTROLLED DML)

```xml
<update tableName="users">
    <column name="contact" value="9999999999"/>
    <where>id = 1</where>
</update>
```

---

## 🔐 Governance Rule Applied:

### DML Controls:

* Block unsafe UPDATE without WHERE
* Block unsafe DELETE without WHERE
* Restrict TRUNCATE operations

---

# 5.6 003-risky-demo.xml (GOVERNANCE TEST FILE)

```xml
<createIndex tableName="users" indexName="idx_users_email"/>
<modifyDataType tableName="users" columnName="email" newDataType="VARCHAR(300)"/>
<dropColumn tableName="users" columnName="name"/>
```

---

## Purpose:

Tests enterprise governance engine:

### DDL operations:

* CREATE INDEX
* ALTER COLUMN TYPE
* DROP COLUMN (HIGH RISK → requires approval)

---

# 5.7 004-raw-sql-test.sql (RISKY SQL DETECTION)

```sql
CREATE TABLE test_table(id int);

UPDATE users SET name='x';  
DELETE FROM users;          
TRUNCATE TABLE users;
```

---

## Governance Engine Rules:

### 🚨 Blocked patterns:

* UPDATE without WHERE → BLOCKED
* DELETE without WHERE → BLOCKED
* TRUNCATE → BLOCKED
* Raw SQL DDL in uncontrolled execution → FLAGGED

---

# 6. MASTER CHANGELOG

```xml
<include file="001-create-users.xml"/>
<include file="001-init.xml"/>
<include file="002-add-email.xml"/>
<include file="002-seed.xml"/>
<include file="003-safe-update.xml"/>
<include file="003-risky-demo.xml"/>
```

---

# 7. Liquibase Properties

```properties
changeLogFile=db/changelog/db.changelog-master.xml
url=jdbc:postgresql://localhost:5432/appdb
username=admin
password=admin
driver=org.postgresql.Driver
searchPath=db/changelog
```

---

# 8. Governance Engine (YOUR REAL IMPLEMENTATION)

## SQL Safety Controls Implemented

### DML Controls:

* Block UPDATE without WHERE
* Block DELETE without WHERE
* Restrict TRUNCATE

### DDL Controls:

* DROP COLUMN → manual approval
* DROP TABLE → manual approval
* ALTER COLUMN TYPE → risk-based approval

---

## Risk Levels

| Level  | Meaning                  |
| ------ | ------------------------ |
| LOW    | Safe execution           |
| MEDIUM | Requires validation      |
| HIGH   | Requires manual approval |

---

# 9. Jenkins Pipeline Flow

```groovy
stage('Validate') {
    sh "./liquibase validate"
}

stage('Dry Run') {
    sh "./liquibase updateSQL"
}

stage('Deploy') {
    sh "./liquibase update"
}
```

---

# 10. Current Behavior (YOUR TEST RESULTS)

### ✔ Working:

* Liquibase XML execution
* Jenkins pipeline trigger
* rollback execution
* schema versioning
* audit history tracking

---

### ✔ Governance Engine Correctly Detects:

* UPDATE without WHERE → BLOCKED
* DELETE without WHERE → BLOCKED
* DROP COLUMN → requires approval
* TRUNCATE → blocked
* risky SQL in `.sql` file → flagged

---

# 11. Rollback Strategy

### Supported:

* rollback by count
* rollback to tag
* rollback per changeset

Example:

```bash
./liquibase rollbackCount 1
```

---

# 12. DATABASECHANGELOG Tables

### Automatically managed:

* change tracking
* execution history
* prevents duplicate execution

---

# 13. Key Observations from Your POC

* Liquibase enforces schema consistency
* Jenkins automates deployment
* Governance engine blocks unsafe SQL
* XML rollback ensures safety
* SQL file testing validates real-world risks

---

# 14. Final Summary

This POC demonstrates a **real enterprise-grade database CI/CD system** with:

* Liquibase version control
* Jenkins automation pipeline
* PostgreSQL backend
* DML/DDL governance engine
* Risk-based execution model
* Safe rollback strategy
* SQL injection & unsafe query protection
