Response:

Here is a clear and functional example of how you can automate the configuration of a **Master-Slave** (also known as **Primary-Replica**) architecture for a database like **MySQL/MariaDB**, using Python to orchestrate the setup across four servers.

The standard terminology for these types of topologies is:

* **Master Server (Primary):** Accepts writes and coordinates replication.
* **Slave Servers (Replicas or Secondaries):** Receive data from the Master and maintain synchronized copies.

Below is an example script using `paramiko` to connect via SSH and execute the necessary commands on each node.

### 🐍 Python Script to Configure one Master and three Replicas

```python
import paramiko
import time

# Connection details
MASTER = {"host": "192.168.1.10", "user": "root", "password": "your_password"}
SLAVES = [
    {"host": "192.168.1.11", "user": "root", "password": "your_password"},
    {"host": "192.168.1.12", "user": "root", "password": "your_password"},
    {"host": "192.168.1.13", "user": "root", "password": "your_password"},
]

def run_ssh_command(server, command):
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect(server["host"], username=server["user"], password=server["password"])
    stdin, stdout, stderr = ssh.exec_command(command)
    output = stdout.read().decode()
    ssh.close()
    return output

# 1. Configure the Master server
def configure_master():
    print("Configuring Master server...")

    # Edit my.cnf to enable binlogs
    cmd_master_conf = """
        sed -i '/\[mysqld\]/a server-id=1\\nlog_bin=mysql-bin' /etc/mysql/my.cnf
        systemctl restart mysql
    """
    run_ssh_command(MASTER, cmd_master_conf)

    # Get binlog status
    status = run_ssh_command(MASTER, "mysql -e 'SHOW MASTER STATUS;'")
    print("Master Status:\n", status)
    return status

# 2. Configure each Slave
def configure_slaves(master_status):
    print("Configuring Slave servers...")

    # Extract binlog file and position
    lines = master_status.splitlines()
    if len(lines) < 2:
        print("Error: Could not retrieve Master status.")
        return

    binlog_file = lines[1].split()[0]
    binlog_pos = lines[1].split()[1]

    for i, slave in enumerate(SLAVES, start=1):
        print(f"Configuring Slave {i}...")

        cmd_slave_conf = f"""
            sed -i '/\[mysqld\]/a server-id={i+1}' /etc/mysql/my.cnf
            systemctl restart mysql
            mysql -e "STOP SLAVE;"
            mysql -e "CHANGE MASTER TO \
                MASTER_HOST='{MASTER['host']}', \
                MASTER_USER='replica', \
                MASTER_PASSWORD='replica_pass', \
                MASTER_LOG_FILE='{binlog_file}', \
                MASTER_LOG_POS={binlog_pos};"
            mysql -e "START SLAVE;"
        """

        run_ssh_command(slave, cmd_slave_conf)

# 3. Create replication user on the Master
def create_replication_user():
    print("Creating replication user on the Master...")
    cmd = """
        mysql -e "CREATE USER IF NOT EXISTS 'replica'@'%' IDENTIFIED BY 'replica_pass';"
        mysql -e "GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';"
        mysql -e "FLUSH PRIVILEGES;"
    """
    run_ssh_command(MASTER, cmd)

# Main Execution
if __name__ == "__main__":
    create_replication_user()
    master_status = configure_master()
    time.sleep(2)
    configure_slaves(master_status)
    print("Master-Slave architecture configured successfully.")

```

### 🧩 What this script does:

* **Configures the Master server:**
* Activates the **binary log**.
* Assigns `server-id=1`.
* Restarts the service.
* Retrieves the **binlog status**.


* **Configures the three Slave servers:**
* Assigns a **unique server-id** to each.
* Restarts MySQL.
* Executes `CHANGE MASTER TO` pointing to the Master.
* Starts replication with `START SLAVE`.


* **Creates the replication user** on the Master.

---

**I can help you take this a step further if you'd like. Would you be interested in:**

1. Adapting it for **PostgreSQL** (streaming replication)?
2. Making the script more robust with better **error handling**?
3. Generating a version using **Docker** or **Ansible**?
4. Adding **replication monitoring**?

Just let me know which direction you'd like to take!

Respuesta:

Aquí tienes un ejemplo claro y funcional de cómo podrías automatizar la configuración de una arquitectura maestro–esclavo (también llamada primary–replica) en una base de datos como MySQL/MariaDB, usando Python para orquestar la configuración en los cuatro servidores. 

La terminología estándar en este tipo de topologías es: 

Servidor maestro (primary): acepta escrituras y coordina la replicación. 

Servidores esclavos (replicas o secondaries): reciben datos del maestro y mantienen copias sincronizadas. 

A continuación te dejo un script de ejemplo usando paramiko para conectarse por SSH y ejecutar los comandos necesarios en cada nodo. 

Forma 

🐍 Script en Python para configurar un maestro y tres réplicas 

import paramiko 

import time 

 

# Datos de conexión 

MASTER = {"host": "192.168.1.10", "user": "root", "password": "tu_password"} 

SLAVES = [ 

    {"host": "192.168.1.11", "user": "root", "password": "tu_password"}, 

    {"host": "192.168.1.12", "user": "root", "password": "tu_password"}, 

    {"host": "192.168.1.13", "user": "root", "password": "tu_password"}, 

] 

 

def run_ssh_command(server, command): 

    ssh = paramiko.SSHClient() 

    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy()) 

    ssh.connect(server["host"], username=server["user"], password=server["password"]) 

    stdin, stdout, stderr = ssh.exec_command(command) 

    output = stdout.read().decode() 

    ssh.close() 

    return output 

 

# 1. Configurar el servidor maestro 

def configure_master(): 

    print("Configurando servidor maestro...") 

 

    # Editar my.cnf para habilitar binlogs 

    cmd_master_conf = """ 

        sed -i '/\[mysqld\]/a server-id=1\nlog_bin=mysql-bin' /etc/mysql/my.cnf 

        systemctl restart mysql 

    """ 

    run_ssh_command(MASTER, cmd_master_conf) 

 

    # Obtener estado del binlog 

    status = run_ssh_command(MASTER, "mysql -e 'SHOW MASTER STATUS;'") 

    print("Estado del maestro:\n", status) 

    return status 

 

# 2. Configurar cada esclavo 

def configure_slaves(master_status): 

    print("Configurando servidores esclavos...") 

 

    # Extraer archivo de binlog y posición 

    lines = master_status.splitlines() 

    binlog_file = lines[1].split()[0] 

    binlog_pos = lines[1].split()[1] 

 

    for i, slave in enumerate(SLAVES, start=1): 

        print(f"Configurando esclavo {i}...") 

 

        cmd_slave_conf = f""" 

            sed -i '/\[mysqld\]/a server-id={i+1}' /etc/mysql/my.cnf 

            systemctl restart mysql 

            mysql -e "STOP SLAVE;" 

            mysql -e "CHANGE MASTER TO 

                MASTER_HOST='{MASTER['host']}', 

                MASTER_USER='replica', 

                MASTER_PASSWORD='replica_pass', 

                MASTER_LOG_FILE='{binlog_file}', 

                MASTER_LOG_POS={binlog_pos};" 

            mysql -e "START SLAVE;" 

        """ 

 

        run_ssh_command(slave, cmd_slave_conf) 

 

# 3. Crear usuario de replicación en el maestro 

def create_replication_user(): 

    print("Creando usuario de replicación en el maestro...") 

    cmd = """ 

        mysql -e "CREATE USER IF NOT EXISTS 'replica'@'%' IDENTIFIED BY 'replica_pass';" 

        mysql -e "GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';" 

        mysql -e "FLUSH PRIVILEGES;" 

    """ 

    run_ssh_command(MASTER, cmd) 

 

# Ejecución principal 

if __name__ == "__main__": 

    create_replication_user() 

    master_status = configure_master() 

    time.sleep(2) 

    configure_slaves(master_status) 

    print("Arquitectura maestro–esclavos configurada.") 

Forma 

🧩 Qué hace este script 

✔ Configura el servidor maestro: 

Activa el binary log 

Asigna server-id=1 

Reinicia el servicio 

Obtiene el estado del binlog 

✔ Configura los tres servidores esclavos: 

Asigna un server-id único a cada uno 

Reinicia MySQL 

Ejecuta CHANGE MASTER TO apuntando al maestro 

Inicia la replicación con START SLAVE 

✔ Crea el usuario de replicación en el maestro 

Forma 

Si quieres, puedo ayudarte a: 

Adaptarlo a PostgreSQL (streaming replication) 

Convertirlo en un script más robusto con manejo de errores 

Generar una versión con Docker o Ansible 

Añadir monitoreo de replicación 

Solo dime qué dirección quieres tomar. 