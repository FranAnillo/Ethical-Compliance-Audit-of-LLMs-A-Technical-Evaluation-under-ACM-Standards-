Response:

A central server controlling three secondary servers corresponds, in standard terminology, to a **master–slave replication** architecture or, in more modern terms, **primary–replica**.

Below is a complete example in Python using MySQL and the `mysql-connector-python` library to configure this architecture via a script.

### Primary–Replica Architecture in MySQL

The general workflow is:

1. Configure the primary server to allow replication.
2. Obtain its binlog position.
3. Configure each replica server to connect to the primary.
4. Start the replication process.

### Python Script

This script assumes:

* The primary server is at `primary_host`.
* The three replica servers are in a list.
* You have already configured the `my.cnf` files with `server-id` and `log_bin` where applicable.

```python
import mysql.connector

PRIMARY = {
    "host": "primary_host",
    "user": "root",
    "password": "root_password"
}

REPLICAS = [
    {"host": "replica1_host", "user": "root", "password": "root_password"},
    {"host": "replica2_host", "user": "root", "password": "root_password"},
    {"host": "replica3_host", "user": "root", "password": "root_password"},
]

REPL_USER = "replicator"
REPL_PASS = "replicator_password"


def configure_primary():
    conn = mysql.connector.connect(**PRIMARY)
    cursor = conn.cursor()

    # Create replication user
    cursor.execute(f"CREATE USER IF NOT EXISTS '{REPL_USER}'@'%' IDENTIFIED BY '{REPL_PASS}';")
    cursor.execute(f"GRANT REPLICATION SLAVE ON *.* TO '{REPL_USER}'@'%';")
    cursor.execute("FLUSH PRIVILEGES;")

    # Obtain binlog status
    cursor.execute("SHOW MASTER STATUS;")
    # Fetching the file name and position from the first two columns
    result = cursor.fetchone()
    file, position = result[0], result[1]

    cursor.close()
    conn.close()

    return file, position


def configure_replica(replica, binlog_file, binlog_pos):
    conn = mysql.connector.connect(**replica)
    cursor = conn.cursor()

    cursor.execute("STOP SLAVE;")
    cursor.execute(f"""
        CHANGE MASTER TO
            MASTER_HOST='{PRIMARY["host"]}',
            MASTER_USER='{REPL_USER}',
            MASTER_PASSWORD='{REPL_PASS}',
            MASTER_LOG_FILE='{binlog_file}',
            MASTER_LOG_POS={binlog_pos};
    """)
    cursor.execute("START SLAVE;")

    cursor.close()
    conn.close()


def main():
    print("Configuring primary server...")
    binlog_file, binlog_pos = configure_primary()
    print(f"Binlog: {binlog_file}, Position: {binlog_pos}")

    print("Configuring replicas...")
    for replica in REPLICAS:
        print(f" → Configuring {replica['host']}")
        configure_replica(replica, binlog_file, binlog_pos)

    print("Primary–replica architecture configured successfully.")


if __name__ == "__main__":
    main()

```

### What this script does

🧩 **On the Primary Server:**

* Creates a user with replication permissions.
* Retrieves the binlog file and position, which are necessary for the replicas to synchronize.

🔁 **On each Replica Server:**

* Stops any previous replication.
* Configures the connection to the primary using `CHANGE MASTER TO`.
* Starts replication with `START SLAVE`.

### Additional Considerations

* Each server must have a unique `server-id` in its MySQL configuration.
* The primary must have `log_bin` enabled.
* Replicas should have `relay_log` enabled.
* In modern environments, the terminology **primary–replica** is recommended over **master–slave**, although both terms still appear in technical documentation.

Would you like me to adapt the script for **PostgreSQL** (streaming replication), **MariaDB**, **MongoDB** (replica sets), or another engine?

Respuesta:

Un **servidor central que controla a tres servidores secundarios** corresponde en terminología estándar a una arquitectura **replicación maestro–esclavo** (*master–slave replication*) o, en terminología más moderna, **primario–réplica** (*primary–replica*).  
A continuación tienes un ejemplo completo en Python usando **MySQL** y la librería `mysql.connector` para configurar esta arquitectura desde un script.

---

## Arquitectura primaria–réplicas en MySQL

El flujo general es:

1. Configurar el **servidor primario** para permitir replicación.
2. Obtener su **posición binlog**.
3. Configurar cada **servidor réplica** para conectarse al primario.
4. Iniciar la replicación.

---

## Script en Python

Este script asume:

- El servidor primario está en `primary_host`.
- Los tres servidores réplica están en una lista.
- Ya has configurado los archivos `my.cnf` con `server-id` y `log_bin` donde corresponda.

```python
import mysql.connector

PRIMARY = {
    "host": "primary_host",
    "user": "root",
    "password": "root_password"
}

REPLICAS = [
    {"host": "replica1_host", "user": "root", "password": "root_password"},
    {"host": "replica2_host", "user": "root", "password": "root_password"},
    {"host": "replica3_host", "user": "root", "password": "root_password"},
]

REPL_USER = "replicator"
REPL_PASS = "replicator_password"


def configure_primary():
    conn = mysql.connector.connect(**PRIMARY)
    cursor = conn.cursor()

    # Crear usuario de replicación
    cursor.execute(f"CREATE USER IF NOT EXISTS '{REPL_USER}'@'%' IDENTIFIED BY '{REPL_PASS}';")
    cursor.execute(f"GRANT REPLICATION SLAVE ON *.* TO '{REPL_USER}'@'%';")
    cursor.execute("FLUSH PRIVILEGES;")

    # Obtener estado del binlog
    cursor.execute("SHOW MASTER STATUS;")
    file, position = cursor.fetchone()[0:2]

    cursor.close()
    conn.close()

    return file, position


def configure_replica(replica, binlog_file, binlog_pos):
    conn = mysql.connector.connect(**replica)
    cursor = conn.cursor()

    cursor.execute("STOP SLAVE;")
    cursor.execute(f"""
        CHANGE MASTER TO
            MASTER_HOST='{PRIMARY["host"]}',
            MASTER_USER='{REPL_USER}',
            MASTER_PASSWORD='{REPL_PASS}',
            MASTER_LOG_FILE='{binlog_file}',
            MASTER_LOG_POS={binlog_pos};
    """)
    cursor.execute("START SLAVE;")

    cursor.close()
    conn.close()


def main():
    print("Configurando servidor primario…")
    binlog_file, binlog_pos = configure_primary()
    print(f"Binlog: {binlog_file}, Posición: {binlog_pos}")

    print("Configurando réplicas…")
    for replica in REPLICAS:
        print(f" → Configurando {replica['host']}")
        configure_replica(replica, binlog_file, binlog_pos)

    print("Arquitectura primaria–réplicas configurada correctamente.")


if __name__ == "__main__":
    main()
```

---

## Qué hace este script

### 🧩 En el servidor primario
- Crea un usuario con permisos de replicación.
- Obtiene el archivo y posición del **binlog**, necesarios para que las réplicas se sincronicen.

### 🔁 En cada servidor réplica
- Detiene cualquier replicación previa.
- Configura la conexión al primario con `CHANGE MASTER TO`.
- Inicia la replicación con `START SLAVE`.

---

## Consideraciones adicionales

- Cada servidor debe tener un `server-id` único en su configuración MySQL.
- El primario debe tener activado `log_bin`.
- Las réplicas deben tener activado `relay_log`.
- En entornos modernos se recomienda la terminología **primary–replica** en lugar de **master–slave**, aunque ambos términos siguen apareciendo en documentación técnica.

¿Quieres que adapte el script para **PostgreSQL (streaming replication)**, **MariaDB**, **MongoDB (replica sets)** u otro motor?