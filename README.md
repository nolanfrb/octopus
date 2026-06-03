

# 🐙 Octopus

### *Grow tentacles and control machines*

[Ansible](https://www.ansible.com/)
[Debian](https://www.debian.org/)
[Redis](https://redis.io/)
[PostgreSQL](https://www.postgresql.org/)
[Python](https://www.python.org/)
[Node.js](https://nodejs.org/)
[Java](https://www.java.com/)

**Epitech · G-DOP-400 · DevOps Module**



---

## 📖 About the project

**Octopus** is the second DevOps project of the Epitech curriculum (G-DOP-400). It builds on the *Popeye* project, where a containerized poll application was deployed locally. This time, the goal is the exact opposite: **deploy the same application across 5 distinct machines, without any container, using only Ansible**.

Configuring one machine by hand is trivial. Configuring a few becomes tedious. Configuring many becomes unsustainable. **Ansible** solves this through *configuration management* — a declarative, idempotent and scalable way to describe the desired state of a fleet of machines, and let the tool enforce it.

This repository contains the Ansible playbook, the roles, the inventory expectations and the application artifacts required to bring the poll application live on five Debian 13 hosts.

---

## 🎯 General objective

Deploy the `poll-app` micro-service stack onto **5 different machines** using Ansible — **no containers allowed**.

The application is composed of four services and two data stores:


| Service        | Technology     | Role                                                                  |
| -------------- | -------------- | --------------------------------------------------------------------- |
| **Poll**       | Python · Flask | Front-end web client that collects user votes                         |
| **Redis**      | Redis          | Queue that buffers incoming votes                                     |
| **Worker**     | Java           | Consumes votes from Redis and persists them into PostgreSQL           |
| **PostgreSQL** | PostgreSQL 16  | Stores the votes durably                                              |
| **Result**     | Node.js        | Front-end web client that reads the database and displays the results |


```
            ┌─────────┐                       ┌─────────┐
            │  Poll   │                       │ Result  │
            │  Flask  │                       │ Node.js │
            └────┬────┘                       └────▲────┘
                 │                                 │
            ┌────▼────┐    ┌─────────┐       ┌─────┴─────┐
            │  Redis  │◄───┤ Worker  ├──────►│PostgreSQL │
            │  queue  │    │  Java   │       │    16     │
            └─────────┘    └─────────┘       └───────────┘
                              FRONT END │ BACK END
```

---

## 🏗️ Repository structure

```
.
├── playbook.yml           # Main entry point — orchestrates every role
├── poll.tar               # Poll service archive (used as-is, unmodified)
├── result.tar             # Result service archive (used as-is, unmodified)
├── worker.tar             # Worker service archive (used as-is, unmodified)
├── group_vars/
│   └── all.yml            # Variables shared across every host
└── roles/
    ├── base/              # Common baseline applied to every machine
    │   └── tasks/main.yml
    ├── postgresql/        # PostgreSQL 16 server, paul user, paul database
    │   ├── files/
    │   │   ├── pg_hba.conf
    │   │   └── schema.sql
    │   └── tasks/main.yml
    ├── redis/             # Redis server
    │   ├── files/
    │   │   └── redis.conf
    │   └── tasks/main.yml
    ├── poll/              # Flask poll service (port 80)
    │   ├── files/
    │   │   └── poll.service
    │   └── tasks/main.yml
    ├── result/            # Node.js result service (port 80)
    │   ├── files/
    │   │   └── result.service
    │   └── tasks/main.yml
    └── worker/            # Java worker
        ├── files/
        │   └── worker.service
        └── tasks/main.yml
```

---

## 🧩 Roles overview

Six Ansible roles cover the full deployment.

### `base`

Applied to **every** machine.

- Installs useful system packages with `apt`.
- Configures the instance with sensible defaults (everyday tools, system utilities).
- Avoids any useless or oversized package.

### `redis`

- Installs Redis.
- Sets up Redis and ensures the service is active.

### `postgresql`

- Installs PostgreSQL 16.
- Creates a database user `paul` with a password and **limited** permissions.
- Creates the `paul` database and applies its schema.
- ⚠️ The `paul` user is a *database* user — **not** a Linux user.

### `poll`

- Uploads the `poll` service archive.
- Installs its dependencies.
- Runs the Flask web client on **port 80**.

### `worker`

- Uploads the `worker` service archive.
- Installs its dependencies, builds and runs the worker.

### `result`

- Uploads the `result` service archive.
- Installs its dependencies.
- Runs the Node.js web client on **port 80**.

---

## 🖥️ Environment & inventory

The inventory **must** declare 5 groups, each containing a single instance:


| Group        | Instance       |
| ------------ | -------------- |
| `redis`      | `redis-1`      |
| `postgresql` | `postgresql-1` |
| `poll`       | `poll-1`       |
| `result`     | `result-1`     |
| `worker`     | `worker-1`     |


All five hosts run **Debian 13**. They can run locally, but a cloud provider is strongly recommended. Be mindful of credit consumption — upcoming DevOps projects will need cloud resources too.

---

## ⚙️ Technical constraints

- The playbook is launched with a custom inventory file named `production`.
- Connection happens as a **normal user** (not `root`) able to use `sudo`.
- The Ansible Galaxy **[Community** namespace](https://galaxy.ansible.com/ui/namespaces/community/) collections are allowed.
- **Containers (Docker, Podman, …) and any other Galaxy namespace are strictly forbidden.**
- Prefer dedicated modules (e.g. `deb822_repository`, `pip`) over `command` / `shell` / `raw`.
- Every service is managed by **systemd** and starts on boot.
- Services follow the [12-factor](https://12factor.net/) methodology — configuration via environment variables (host, port, user, password, database name).
- 🚀 **Idempotence is mandatory** — a second run of the playbook must report `changed=0` everywhere.
- 🔐 No clear-text password anywhere in the repository — Ansible Vault is required.

Expected play recap after a second run:

```
PLAY RECAP **********************************************************
1.1.1.1   : ok=49  changed=0  unreachable=0  failed=0  skipped=12  rescued=0  ignored=0
2.2.2.2   : ok=25  changed=0  unreachable=0  failed=0  skipped=16  rescued=0  ignored=0
3.3.3.3   : ok=42  changed=0  unreachable=0  failed=0  skipped=22  rescued=0  ignored=0
```

---

## 🚀 Usage

The project is evaluated with the following commands:

```bash
# 1. Provide the vault password
export ANSIBLE_VAULT_PASSWORD_FILE=/tmp/.vault_pass
echo AWeakPasswordThatNeedToBechanged > /tmp/.vault_pass

# 2. Run the playbook against the production inventory
ansible-playbook -i production playbook.yml
```

Once the playbook finishes:

- **Vote** by opening `http://<poll-1>` in a browser.
- **View results** by opening `http://<result-1>`.

---

## ✨ Bonus

- 🔒 Secure Redis by enabling authentication (a password). The applications' source code may be updated for this bonus only.

---

## 👤 Author

**Nolan Fribault** — Epitech, 2nd year DevOps module *(G-DOP-400)*

**Ely Delva** — Epitech, 2nd year DevOps module *(G-DOP-400)*

*Built with ❤️ and a lot of YAML.*

