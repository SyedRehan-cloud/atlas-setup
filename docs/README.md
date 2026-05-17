# LIQUIBASE ENTERPRISE POC

## Overview

This project demonstrates automated database schema management using:

- Liquibase (Database migration tool)
- Jenkins (CI/CD pipeline)
- PostgreSQL (Database)

---

## Architecture

Developer → Git → Jenkins → Liquibase → PostgreSQL

---

## Features

✔ Schema versioning  
✔ Automated migration  
✔ Audit tracking  
✔ Rollback support  
✔ CI/CD integration  

---

## How it works

1. Developer modifies XML changelog
2. Jenkins pipeline is triggered
3. Liquibase validates changes
4. Liquibase applies migration
5. DATABASECHANGELOG tracks execution
6. Rollback available if needed

---

## Enterprise Benefits

✔ No manual SQL execution  
✔ Full audit trail  
✔ Safe production deployments  
✔ Controlled schema evolution  
✔ Regulatory compliance ready  

---

## Key Tables

- DATABASECHANGELOG → stores migration history  
- DATABASECHANGELOGLOCK → prevents parallel execution  

---

## Rollback

Liquibase supports rollback using:

- rollbackCount
- rollback tags inside changesets

---

## Conclusion

This system ensures safe, automated, and traceable database evolution using CI/CD.
