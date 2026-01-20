# PostgreSQL Basics – Concise Notes

---

## Install PostgreSQL (Ubuntu)

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl status postgresql
```

> Note: `active (exited)` is normal; actual DB runs in cluster service.

---

## Default superuser

```bash
sudo -i -u postgres  # switch Linux user
psql                 # login as DB role 'postgres'
```

* `postgres` = superuser, all privileges, can bypass permissions.

---

## Users, Roles, Owners

| Term          | Meaning                                          |
| ------------- | ------------------------------------------------ |
| **User**      | Login role (can log in)                          |
| **Role**      | Permission container (may or may not login)      |
| **Owner**     | Creator of DB/Table/Schema, has full permissions |
| **Superuser** | Can access everything, ignore grants             |

* **Rule:** All users are roles, but not all roles are users.

---

## Create user / DB / table

```sql
CREATE ROLE zihad LOGIN PASSWORD '123';
CREATE DATABASE acid OWNER zihad;
\c acid
CREATE TABLE products (id serial, name text); -- Owner = zihad
```

* Owner = automatic full permissions
* `postgres` superuser can access all objects anyway

---

## Granting permissions

```sql
GRANT ALL PRIVILEGES ON DATABASE acid TO zihad;
GRANT ALL ON SCHEMA public TO zihad;
```

---

## Login examples

* As `postgres`:

```bash
sudo -i -u postgres
psql
```

* As `zihad` role:

```bash
psql -U zihad -d acid -h localhost
```

* Optional peer auth if Linux user = DB role:

```bash
sudo -i -u zihad
psql -d acid
```

---

## Quick Diagram

```text
          ┌───────────────┐
          │  Linux User   │ (sudo -i -u)
          └───────┬───────┘
                  │ peer auth
                  ▼
          ┌───────────────┐
          │ PostgreSQL    │
          │ Login User    │ (Role)
          └───────┬───────┘
                  │ owns / creates
                  ▼
          ┌───────────────┐
          │   Database    │ Owner = zihad
          └───────┬───────┘
                  │ contains
                  ▼
          ┌───────────────┐
          │    Schema     │ public
          └───────┬───────┘
                  │ contains
                  ▼
          ┌───────────────┐
          │    Tables     │ Owner = zihad
          └───────────────┘
```

---

## Key Points

* Linux user ≠ PostgreSQL role
* Peer auth: Linux user → DB role
* Owner = full control over DB/Table
* Superuser = bypasses all permissions

---

