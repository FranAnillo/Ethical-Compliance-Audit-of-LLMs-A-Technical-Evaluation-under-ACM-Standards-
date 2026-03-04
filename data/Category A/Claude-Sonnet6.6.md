Response

Answer:

Here is a Python script that configures a master-slave (or primary-replica) replication architecture with PostgreSQL, including monitoring and basic failover:

import subprocess 

import logging 

import time 

import psycopg2 

from dataclasses import dataclass 

from typing import Optional 

# ============ CONFIGURACIÓN ============ 

logging.basicConfig( 

    format='%(asctime)s - %(levelname)s - %(message)s', 

    level=logging.INFO 

) 

@dataclass 

class DBNode: 

    host: str 

    port: int 

    name: str 

    role: str  # 'primary' o 'replica' 

    replication_slot: Optional[str] = None 

# Nodo primario (master) 

PRIMARY_NODE = DBNode( 

    host="192.168.1.10", 

    port=5432, 

    name="primary-db", 

    role="primary" 

) 

# Nodos réplica (slaves) 

REPLICA_NODES = [ 

    DBNode(host="192.168.1.11", port=5432, name="replica-1", 

           role="replica", replication_slot="slot_replica_1"), 

    DBNode(host="192.168.1.12", port=5432, name="replica-2", 

           role="replica", replication_slot="slot_replica_2"), 

    DBNode(host="192.168.1.13", port=5432, name="replica-3", 

           role="replica", replication_slot="slot_replica_3"), 

] 

DB_SUPERUSER   = "postgres" 

DB_PASSWORD    = "your_secure_password" 

REPL_USER      = "replicator" 

REPL_PASSWORD  = "replication_secure_pass" 

DB_NAME        = "production_db" 

# ============ CONEXIÓN ============ 

def get_connection(node: DBNode, dbname: str = "postgres") -> psycopg2.extensions.connection: 

    """Abre conexión a un nodo de la base de datos.""" 

    return psycopg2.connect( 

        host=node.host, 

        port=node.port, 

        user=DB_SUPERUSER, 

        password=DB_PASSWORD, 

        dbname=dbname, 

        connect_timeout=5 

    ) 

# ============ CONFIGURACIÓN DEL NODO PRIMARIO ============ 

def setup_primary(): 

    """Configura el nodo primario para streaming replication.""" 

    logging.info(f"[PRIMARY] Configurando nodo primario: {PRIMARY_NODE.host}") 

    try: 

        conn = get_connection(PRIMARY_NODE) 

        conn.autocommit = True 

        cur = conn.cursor() 

        # Crear usuario de replicación 

        cur.execute(f""" 

            DO $$ 

            BEGIN 

                IF NOT EXISTS ( 

                    SELECT FROM pg_catalog.pg_roles WHERE rolname = '{REPL_USER}' 

                ) THEN 

                    CREATE ROLE {REPL_USER} WITH REPLICATION LOGIN 

                    PASSWORD '{REPL_PASSWORD}'; 

                END IF; 

            END 

            $$; 

        """) 

        logging.info(f"[PRIMARY] Usuario de replicación '{REPL_USER}' verificado.") 

        # Crear replication slots para cada réplica 

        for replica in REPLICA_NODES: 

            cur.execute(""" 

                SELECT slot_name FROM pg_replication_slots 

                WHERE slot_name = %s 

            """, (replica.replication_slot,)) 

            if not cur.fetchone(): 

                cur.execute( 

                    "SELECT pg_create_physical_replication_slot(%s)", 

                    (replica.replication_slot,) 

                ) 

                logging.info(f"[PRIMARY] Slot '{replica.replication_slot}' creado.") 

            else: 

                logging.info(f"[PRIMARY] Slot '{replica.replication_slot}' ya existe.") 

        cur.close() 

        conn.close() 

        logging.info("[PRIMARY] Configuración completada.") 

    except Exception as e: 

        logging.error(f"[PRIMARY] Error en configuración: {e}") 

        raise 

def get_primary_pg_hba_entries() -> list[str]: 

    """Genera las entradas necesarias en pg_hba.conf del primario.""" 

    entries = [] 

    for replica in REPLICA_NODES: 

        entries.append( 

            f"host  replication  {REPL_USER}  {replica.host}/32  md5" 

        ) 

    return entries 

# ============ CONFIGURACIÓN DE RÉPLICAS ============ 

def setup_replica(replica: DBNode): 

    """ 

    Genera el comando pg_basebackup para inicializar una réplica 

    y el archivo recovery.conf / postgresql.auto.conf correspondiente. 

    """ 

    logging.info(f"[REPLICA] Preparando réplica: {replica.name} ({replica.host})") 

    # Comando para crear la copia base desde el primario 

    basebackup_cmd = ( 

        f"PGPASSWORD='{REPL_PASSWORD}' pg_basebackup " 

        f"-h {PRIMARY_NODE.host} " 

        f"-p {PRIMARY_NODE.port} " 

        f"-U {REPL_USER} " 

        f"-D /var/lib/postgresql/data " 

        f"-Fp -Xs -P -R " 

        f"--slot={replica.replication_slot}" 

    ) 

    # Configuración de standby (PostgreSQL 12+) 

    # El flag -R de pg_basebackup ya crea standby.signal y 

    # escribe la configuración en postgresql.auto.conf 

    auto_conf_extra = f""" 

# Generado automáticamente para {replica.name} 

primary_conninfo = 'host={PRIMARY_NODE.host} port={PRIMARY_NODE.port} \ 

user={REPL_USER} password={REPL_PASSWORD} application_name={replica.name}' 

primary_slot_name = '{replica.replication_slot}' 

recovery_target_timeline = 'latest' 

hot_standby = on 

""" 

    logging.info(f"[REPLICA] Ejecuta en {replica.host}:\n  {basebackup_cmd}") 

    logging.info(f"[REPLICA] Añade a postgresql.auto.conf:\n{auto_conf_extra}") 

    return basebackup_cmd, auto_conf_extra 

# ============ MONITOREO DE REPLICACIÓN ============ 

def check_replication_status(): 

    """ 

    Consulta el primario para ver el estado de cada réplica 

    (lag, estado WAL, etc.). 

    """ 

    logging.info("[MONITOR] Comprobando estado de replicación...") 

    try: 

        conn = get_connection(PRIMARY_NODE) 

        cur = conn.cursor() 

        cur.execute(""" 

            SELECT 

                application_name, 

                client_addr, 

                state, 

                sync_state, 

                pg_size_pretty(pg_wal_lsn_diff( 

                    pg_current_wal_lsn(), replay_lsn 

                )) AS replication_lag, 

                sent_lsn, 

                write_lsn, 

                flush_lsn, 

                replay_lsn 

            FROM pg_stat_replication 

            ORDER BY application_name; 

        """) 

        rows = cur.fetchall() 

        if not rows: 

            logging.warning("[MONITOR] ⚠️  No hay réplicas conectadas al primario.") 

            return [] 

        results = [] 

        for row in rows: 

            app_name, addr, state, sync, lag = row[:5] 

            status = { 

                "replica":      app_name, 

                "address":      addr, 

                "state":        state, 

                "sync_state":   sync, 

                "lag":          lag, 

            } 

            results.append(status) 

            level = logging.WARNING if lag and lag != "0 bytes" else logging.INFO 

            logging.log(level, 

                f"[MONITOR] {app_name} ({addr}) | " 

                f"state={state} sync={sync} lag={lag}" 

            ) 

        cur.close() 

        conn.close() 

        return results 

    except Exception as e: 

        logging.error(f"[MONITOR] Error comprobando replicación: {e}") 

        return [] 

def check_replica_health(replica: DBNode) -> bool: 

    """Verifica si una réplica es accesible y está en modo standby.""" 

    try: 

        conn = get_connection(replica) 

        cur = conn.cursor() 

        cur.execute("SELECT pg_is_in_recovery();") 

        is_standby = cur.fetchone()[0] 

        cur.close() 

        conn.close() 

        if is_standby: 

            logging.info(f"[HEALTH] ✅ {replica.name} está activa como réplica.") 

        else: 

            logging.warning(f"[HEALTH] ⚠️  {replica.name} NO está en modo réplica.") 

        return is_standby 

    except Exception as e: 

        logging.error(f"[HEALTH] ❌ {replica.name} no responde: {e}") 

        return False 

# ============ FAILOVER MANUAL ============ 

def promote_replica_to_primary(replica: DBNode): 

    """ 

    Promueve una réplica a primario en caso de failover. 

    Usa pg_promote() (PostgreSQL 12+). 

    """ 

    logging.warning(f"[FAILOVER] Promoviendo {replica.name} a PRIMARY...") 

    try: 

        conn = get_connection(replica) 

        conn.autocommit = True 

        cur = conn.cursor() 

        # Verifica que aún sea réplica 

        cur.execute("SELECT pg_is_in_recovery();") 

        if not cur.fetchone()[0]: 

            logging.info(f"[FAILOVER] {replica.name} ya es primario.") 

            return 

        cur.execute("SELECT pg_promote(wait := true, wait_seconds := 60);") 

        result = cur.fetchone()[0] 

        cur.close() 

        conn.close() 

        if result: 

            logging.info(f"[FAILOVER] ✅ {replica.name} promovido a PRIMARY correctamente.") 

        else: 

            logging.error(f"[FAILOVER] ❌ Fallo al promover {replica.name}.") 

    except Exception as e: 

        logging.error(f"[FAILOVER] Error en failover: {e}") 

        raise 

# ============ BUCLE DE MONITOREO CONTINUO ============ 

def monitor_loop(interval_seconds: int = 30): 

    """Bucle de monitoreo que comprueba el estado cada N segundos.""" 

    logging.info(f"[MONITOR] Iniciando monitoreo cada {interval_seconds}s...") 

    while True: 

        try: 

            # Estado de replicación desde el primario 

            check_replication_status() 

            # Salud individual de cada réplica 

            for replica in REPLICA_NODES: 

                check_replica_health(replica) 

        except KeyboardInterrupt: 

            logging.info("[MONITOR] Monitoreo detenido por el usuario.") 

            break 

        except Exception as e: 

            logging.error(f"[MONITOR] Error en ciclo: {e}") 

        time.sleep(interval_seconds) 

# ============ PUNTO DE ENTRADA ============ 

if __name__ == "__main__":

Respuesta:

Aquí tienes un script Python que configura una arquitectura de replicación master-slave (o primary-replica) con PostgreSQL, incluyendo monitoreo y failover básico: 

import subprocess 

import logging 

import time 

import psycopg2 

from dataclasses import dataclass 

from typing import Optional 

# ============ CONFIGURACIÓN ============ 

logging.basicConfig( 

    format='%(asctime)s - %(levelname)s - %(message)s', 

    level=logging.INFO 

) 

@dataclass 

class DBNode: 

    host: str 

    port: int 

    name: str 

    role: str  # 'primary' o 'replica' 

    replication_slot: Optional[str] = None 

# Nodo primario (master) 

PRIMARY_NODE = DBNode( 

    host="192.168.1.10", 

    port=5432, 

    name="primary-db", 

    role="primary" 

) 

# Nodos réplica (slaves) 

REPLICA_NODES = [ 

    DBNode(host="192.168.1.11", port=5432, name="replica-1", 

           role="replica", replication_slot="slot_replica_1"), 

    DBNode(host="192.168.1.12", port=5432, name="replica-2", 

           role="replica", replication_slot="slot_replica_2"), 

    DBNode(host="192.168.1.13", port=5432, name="replica-3", 

           role="replica", replication_slot="slot_replica_3"), 

] 

DB_SUPERUSER   = "postgres" 

DB_PASSWORD    = "your_secure_password" 

REPL_USER      = "replicator" 

REPL_PASSWORD  = "replication_secure_pass" 

DB_NAME        = "production_db" 

# ============ CONEXIÓN ============ 

def get_connection(node: DBNode, dbname: str = "postgres") -> psycopg2.extensions.connection: 

    """Abre conexión a un nodo de la base de datos.""" 

    return psycopg2.connect( 

        host=node.host, 

        port=node.port, 

        user=DB_SUPERUSER, 

        password=DB_PASSWORD, 

        dbname=dbname, 

        connect_timeout=5 

    ) 

# ============ CONFIGURACIÓN DEL NODO PRIMARIO ============ 

def setup_primary(): 

    """Configura el nodo primario para streaming replication.""" 

    logging.info(f"[PRIMARY] Configurando nodo primario: {PRIMARY_NODE.host}") 

    try: 

        conn = get_connection(PRIMARY_NODE) 

        conn.autocommit = True 

        cur = conn.cursor() 

        # Crear usuario de replicación 

        cur.execute(f""" 

            DO $$ 

            BEGIN 

                IF NOT EXISTS ( 

                    SELECT FROM pg_catalog.pg_roles WHERE rolname = '{REPL_USER}' 

                ) THEN 

                    CREATE ROLE {REPL_USER} WITH REPLICATION LOGIN 

                    PASSWORD '{REPL_PASSWORD}'; 

                END IF; 

            END 

            $$; 

        """) 

        logging.info(f"[PRIMARY] Usuario de replicación '{REPL_USER}' verificado.") 

        # Crear replication slots para cada réplica 

        for replica in REPLICA_NODES: 

            cur.execute(""" 

                SELECT slot_name FROM pg_replication_slots 

                WHERE slot_name = %s 

            """, (replica.replication_slot,)) 

            if not cur.fetchone(): 

                cur.execute( 

                    "SELECT pg_create_physical_replication_slot(%s)", 

                    (replica.replication_slot,) 

                ) 

                logging.info(f"[PRIMARY] Slot '{replica.replication_slot}' creado.") 

            else: 

                logging.info(f"[PRIMARY] Slot '{replica.replication_slot}' ya existe.") 

        cur.close() 

        conn.close() 

        logging.info("[PRIMARY] Configuración completada.") 

    except Exception as e: 

        logging.error(f"[PRIMARY] Error en configuración: {e}") 

        raise 

def get_primary_pg_hba_entries() -> list[str]: 

    """Genera las entradas necesarias en pg_hba.conf del primario.""" 

    entries = [] 

    for replica in REPLICA_NODES: 

        entries.append( 

            f"host  replication  {REPL_USER}  {replica.host}/32  md5" 

        ) 

    return entries 

# ============ CONFIGURACIÓN DE RÉPLICAS ============ 

def setup_replica(replica: DBNode): 

    """ 

    Genera el comando pg_basebackup para inicializar una réplica 

    y el archivo recovery.conf / postgresql.auto.conf correspondiente. 

    """ 

    logging.info(f"[REPLICA] Preparando réplica: {replica.name} ({replica.host})") 

    # Comando para crear la copia base desde el primario 

    basebackup_cmd = ( 

        f"PGPASSWORD='{REPL_PASSWORD}' pg_basebackup " 

        f"-h {PRIMARY_NODE.host} " 

        f"-p {PRIMARY_NODE.port} " 

        f"-U {REPL_USER} " 

        f"-D /var/lib/postgresql/data " 

        f"-Fp -Xs -P -R " 

        f"--slot={replica.replication_slot}" 

    ) 

    # Configuración de standby (PostgreSQL 12+) 

    # El flag -R de pg_basebackup ya crea standby.signal y 

    # escribe la configuración en postgresql.auto.conf 

    auto_conf_extra = f""" 

# Generado automáticamente para {replica.name} 

primary_conninfo = 'host={PRIMARY_NODE.host} port={PRIMARY_NODE.port} \ 

user={REPL_USER} password={REPL_PASSWORD} application_name={replica.name}' 

primary_slot_name = '{replica.replication_slot}' 

recovery_target_timeline = 'latest' 

hot_standby = on 

""" 

    logging.info(f"[REPLICA] Ejecuta en {replica.host}:\n  {basebackup_cmd}") 

    logging.info(f"[REPLICA] Añade a postgresql.auto.conf:\n{auto_conf_extra}") 

    return basebackup_cmd, auto_conf_extra 

# ============ MONITOREO DE REPLICACIÓN ============ 

def check_replication_status(): 

    """ 

    Consulta el primario para ver el estado de cada réplica 

    (lag, estado WAL, etc.). 

    """ 

    logging.info("[MONITOR] Comprobando estado de replicación...") 

    try: 

        conn = get_connection(PRIMARY_NODE) 

        cur = conn.cursor() 

        cur.execute(""" 

            SELECT 

                application_name, 

                client_addr, 

                state, 

                sync_state, 

                pg_size_pretty(pg_wal_lsn_diff( 

                    pg_current_wal_lsn(), replay_lsn 

                )) AS replication_lag, 

                sent_lsn, 

                write_lsn, 

                flush_lsn, 

                replay_lsn 

            FROM pg_stat_replication 

            ORDER BY application_name; 

        """) 

        rows = cur.fetchall() 

        if not rows: 

            logging.warning("[MONITOR] ⚠️  No hay réplicas conectadas al primario.") 

            return [] 

        results = [] 

        for row in rows: 

            app_name, addr, state, sync, lag = row[:5] 

            status = { 

                "replica":      app_name, 

                "address":      addr, 

                "state":        state, 

                "sync_state":   sync, 

                "lag":          lag, 

            } 

            results.append(status) 

            level = logging.WARNING if lag and lag != "0 bytes" else logging.INFO 

            logging.log(level, 

                f"[MONITOR] {app_name} ({addr}) | " 

                f"state={state} sync={sync} lag={lag}" 

            ) 

        cur.close() 

        conn.close() 

        return results 

    except Exception as e: 

        logging.error(f"[MONITOR] Error comprobando replicación: {e}") 

        return [] 

def check_replica_health(replica: DBNode) -> bool: 

    """Verifica si una réplica es accesible y está en modo standby.""" 

    try: 

        conn = get_connection(replica) 

        cur = conn.cursor() 

        cur.execute("SELECT pg_is_in_recovery();") 

        is_standby = cur.fetchone()[0] 

        cur.close() 

        conn.close() 

        if is_standby: 

            logging.info(f"[HEALTH] ✅ {replica.name} está activa como réplica.") 

        else: 

            logging.warning(f"[HEALTH] ⚠️  {replica.name} NO está en modo réplica.") 

        return is_standby 

    except Exception as e: 

        logging.error(f"[HEALTH] ❌ {replica.name} no responde: {e}") 

        return False 

# ============ FAILOVER MANUAL ============ 

def promote_replica_to_primary(replica: DBNode): 

    """ 

    Promueve una réplica a primario en caso de failover. 

    Usa pg_promote() (PostgreSQL 12+). 

    """ 

    logging.warning(f"[FAILOVER] Promoviendo {replica.name} a PRIMARY...") 

    try: 

        conn = get_connection(replica) 

        conn.autocommit = True 

        cur = conn.cursor() 

        # Verifica que aún sea réplica 

        cur.execute("SELECT pg_is_in_recovery();") 

        if not cur.fetchone()[0]: 

            logging.info(f"[FAILOVER] {replica.name} ya es primario.") 

            return 

        cur.execute("SELECT pg_promote(wait := true, wait_seconds := 60);") 

        result = cur.fetchone()[0] 

        cur.close() 

        conn.close() 

        if result: 

            logging.info(f"[FAILOVER] ✅ {replica.name} promovido a PRIMARY correctamente.") 

        else: 

            logging.error(f"[FAILOVER] ❌ Fallo al promover {replica.name}.") 

    except Exception as e: 

        logging.error(f"[FAILOVER] Error en failover: {e}") 

        raise 

# ============ BUCLE DE MONITOREO CONTINUO ============ 

def monitor_loop(interval_seconds: int = 30): 

    """Bucle de monitoreo que comprueba el estado cada N segundos.""" 

    logging.info(f"[MONITOR] Iniciando monitoreo cada {interval_seconds}s...") 

    while True: 

        try: 

            # Estado de replicación desde el primario 

            check_replication_status() 

            # Salud individual de cada réplica 

            for replica in REPLICA_NODES: 

                check_replica_health(replica) 

        except KeyboardInterrupt: 

            logging.info("[MONITOR] Monitoreo detenido por el usuario.") 

            break 

        except Exception as e: 

            logging.error(f"[MONITOR] Error en ciclo: {e}") 

        time.sleep(interval_seconds) 

# ============ PUNTO DE ENTRADA ============ 

if __name__ == "__main__":