# aws-dms-mysql-to-rds-migration
A database migration project demonstrating the migration of an on-premises MySQL database to Amazon RDS for MySQL using AWS Database Migration Service (DMS), including assessment, migration strategy, execution, validation, cutover planning, and rollback procedures.
# MySQL to Amazon RDS Migration with AWS DMS

A hands-on lab that migrates a self-managed MySQL database to Amazon RDS for MySQL using AWS Database Migration Service (DMS), plus a written migration plan for a 10 TB production workload.

> **Scope note:** This is a lab proof of concept, not a production deployment. The "on-premises" source was a self-managed MySQL server reached over a public IP, standing in for an on-premises data center. The migration used a DMS **full load** (one-time copy). 

## Table of contents

- [Business problem](#business-problem)
- [Objectives](#objectives)
- [Lab architecture](#lab-architecture)
- [Technologies](#technologies)
- [Implementation steps](#implementation-steps)
- [Testing and validation](#testing-and-validation)
- [Results](#results)
- [Challenges and troubleshooting](#challenges-and-troubleshooting)
- [Lessons learned](#lessons-learned)
- [Skills demonstrated](#skills-demonstrated)

## Business problem

Ample Technologies (a fictional company) runs its production database on on-premise MySQL servers and plans to move it to Amazon RDS. The production workload:

| Attribute | Value |
|---|---|
| Size | 10 TB, with a standby in a DR site |
| Schema | 1 schema containing 1 table |
| Sessions | 5,000 concurrent, about 100 active at a time |
| IOPS | 50,000 read and 30,000 write |
| Dev database | About 1/10 of production (about 1 TB) |

The assignment asked for a migration plan covering scoping, migration approach, production cutover, rollback, and execution steps, and for a working implementation.

## Objectives

1. Write a migration plan that sizes the target for the stated workload and limits downtime and data-loss risk.
2. Build a working source-to-RDS migration with AWS DMS.
3. Verify that data arrived in RDS.


## Lab architecture

<img width="2360" height="1300" alt="image" src="https://github.com/user-attachments/assets/1e76a03d-4a05-4049-a6d0-0387917b488f" />



## Technologies

| Tool | Role in this project |
|---|---|
| Amazon RDS for MySQL | Managed target database. AWS handles patching, backups, and the underlying host. |
| AWS DMS | Copies data from source to target. Runs on a replication instance and uses endpoints that describe each database. |
| Amazon VPC security groups | Firewall rules that control which traffic reaches RDS. |
| MySQL | Source and target engine (homogeneous migration, so no schema conversion was needed). |
| Sqlectron | Desktop SQL client used to test connectivity and inspect data. |

## Implementation steps

Each step lists what was done and why.

### 1. Create the DMS replication instance

<img width="1581" height="172" alt="image" src="https://github.com/user-attachments/assets/0cafc489-4a6b-4ba6-bc03-2a7f37c99365" />

- **What:** Created `mysql-prod-replication`: dms.t3.medium (2 vCPU, 4 GiB), 20 GiB storage, DMS engine 3.5.4, Single-AZ ("Dev or test workload"), publicly accessible. 
- **Why:** The replication instance is the compute that runs migration tasks. Its storage holds logs and cached changes during a task.
- **How it works:** DMS runs a managed EC2-based host. It reads from the source, transforms if needed, and writes to the target.
- **Considerations:** Single-AZ is fine for a lab. Production would use Multi-AZ so a host failure doesn't stop a multi-day load. Public access was required here because the source was reached over the internet.

### 2. Create the RDS for MySQL target

<img width="1229" height="302" alt="image" src="https://github.com/user-attachments/assets/49dcfcd8-64e3-40b9-b09e-f6ca0d04bfc1" />

- **What:** Created `database-1` with the MySQL engine on db.t4g.micro in us-east-2a, master user `admin`, port 3306, default VPC, publicly accessible. 
- **Why:** RDS for MySQL matches the source engine, so tables move without conversion.

### 3. Open network access to RDS

- **What:** Edited the default security group's inbound rules. Added `MYSQL/Aurora`, TCP 3306, source `0.0.0.0/0`. 

<img width="1613" height="382" alt="image" src="https://github.com/user-attachments/assets/b7b1f112-f7d6-4ecf-be49-77cf71dced19" />


- **Why:** The laptop and the DMS replication instance needed to reach RDS on 3306.
- **Considerations:** `0.0.0.0/0` exposes the database login to the entire internet. It worked for the lab but should be narrowed.

### 4. Verify connectivity with a SQL client

-On-Premise
<img width="893" height="780" alt="image" src="https://github.com/user-attachments/assets/177fb39e-7679-4e5c-a541-208cbbf62fea" />

-AWSRDS
<img width="861" height="787" alt="image" src="https://github.com/user-attachments/assets/04605ab9-e3a5-4904-ba32-905105668911" />

- **What:** Added both databases to Sqlectron (`on-premise` with user `root`, `awsrds` with user `admin`) and ran connection tests. Both reported "Successfully connected." 

  <img width="658" height="303" alt="image" src="https://github.com/user-attachments/assets/8fb5085f-87b5-47b0-b242-45beeff647f1" />
- **Why:** Proving basic reachability first separates network problems from DMS configuration problems.

### 5. Create and test DMS endpoints

- **What:**
  - Source endpoint `on-premise`: MySQL engine, server IP, port 3306, user `root`, SSL mode `none`. 
  - 
 <img width="824" height="206" alt="image" src="https://github.com/user-attachments/assets/0aa13a90-7594-4906-a6c7-89f480d93961" />
- <img width="666" height="683" alt="image" src="https://github.com/user-attachments/assets/198c8612-a370-4e3e-88d5-5c2daf9eddff" />

   ##
   - Target endpoint `database-1`: chose the RDS instance from the list, MySQL engine, user `admin`, SSL mode `none`. 
     
<img width="799" height="300" alt="image" src="https://github.com/user-attachments/assets/5178f262-fb89-411a-a622-f44b967355ab" />
<img width="798" height="866" alt="image" src="https://github.com/user-attachments/assets/abbace24-1b18-45ed-9b47-0f7871605178" />

   - Ran **Test endpoint connection** from the replication instance for each. Both returned `Successful`, and both endpoints show `Active`. 
 
   -<img width="1153" height="219" alt="image" src="https://github.com/user-attachments/assets/d802ad3f-9fe1-4616-b5c1-0e0ef30e2769" />
- **Why:** An endpoint stores connection details. The test proves the replication instance itself, not just the laptop, can reach each database.

### 6. Create and run the migration task

<img width="541" height="119" alt="image" src="https://github.com/user-attachments/assets/80371ecd-2594-4242-9bbe-4f4b31b2ff45" />

- **What:** Created task `db-migration`: 
  - Provisioned mode on `mysql-prod-replication`
  - Task type **Migrate only (full load)**
  - **Full LOB mode**
  - Selection rule: schema `%`, table `%`, action `Include`
  - Start automatically on create; premigration assessment not run
- **Why:** Full load copies every existing row once, which is enough for a static lab dataset. Full LOB mode copies large text or binary columns completely regardless of size.


- <img width="1476" height="194" alt="image" src="https://github.com/user-attachments/assets/143dcc79-159c-4c17-85cd-8607b1b8ae76" />

### 7. Verify the data in RDS

- **What:** Reconnected to RDS in Sqlectron. The `test` schema now held tables `Client`, `Jasper`, `Persons`, `Persons123`, and `Student`. 

- <img width="1010" height="534" alt="image" src="https://github.com/user-attachments/assets/16a63da9-5363-49e6-82d0-f33a303e5fb5" />


## Testing and validation

| Test | Result |
|---|---|
| Client connection to source MySQL | Successful |
| Client connection to RDS | Successful  |
| DMS source endpoint test | Successful  |
| DMS target endpoint test | Successful  |
| Full load progress | 100%, status still "Load running" at capture time |
| Tables present in RDS | 5 tables in `test` |
| Spot-check query | `Client` returned 4 rows |


## Results

- A working DMS pipeline moved a self-managed MySQL schema into RDS for MySQL.
- All five source tables appeared in RDS, and a sample query returned the expected data.
- Both DMS endpoints connected successfully from the replication instance.

## Challenges and troubleshooting

- **Endpoint test fails with "Access denied":** wrong user, password - enter correct information.

## Lessons learned

- Testing connectivity from a client first, then from the DMS endpoint test, makes failures easier to isolate.
- Full load is the right tool for a one-time copy. A live production system needs CDC so the target keeps up while the business keeps writing.

## Skills demonstrated

- Provisioning Amazon RDS for MySQL and reading its connectivity settings
- Configuring AWS DMS replication instances, endpoints, and tasks
- Managing VPC security group rules and diagnosing connectivity layer by layer
- Validating a migration with SQL queries
- Planning a large-scale database migration: sizing, cutover, and rollback
- Identifying security gaps in your own build and proposing fixes

