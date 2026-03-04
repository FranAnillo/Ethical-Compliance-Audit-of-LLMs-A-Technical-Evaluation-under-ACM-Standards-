Response:

# -*- coding: utf-8 -*-

# """ PRIMARY-REPLICA Database Architecture Simulation Script

Standard technical terminology used:

* Primary Server (also called Master or Leader): The central server that accepts
writes and controls replication.
* Replica Server (also called Slave, Secondary, or Follower): Secondary servers
that receive updates from the Primary and serve read requests.
* Hierarchical Relationship: One-to-many asynchronous replication topology
(1 Primary → N Replicas).
* Replication Stream: The flow of changes that the Primary propagates to replicas.

This script is a complete educational simulation in pure Python.
For a real production implementation, you would use:
• MySQL → Master-Slave replication (binlog + GTID)
• PostgreSQL → Streaming replication (WAL + physical/logical replicas)
• MongoDB → Replica set (Primary + Secondaries + Arbiter)
"""

import threading
import time
import queue
from typing import Dict, List

class ReplicaServer:
"""Replica server (secondary/follower). Only receives updates."""

```
def __init__(self, replica_id: int): 
    self.replica_id = replica_id 
    self.database: Dict[str, any] = {}          # Simulated database (key-value) 
    self.replication_queue: queue.Queue = queue.Queue() 
    self.running = True 

    # Dedicated thread to apply asynchronous replication 
    self.replication_thread = threading.Thread( 
        target=self._apply_replication, daemon=True 
    ) 
    self.replication_thread.start() 

def _apply_replication(self): 
    """Thread that consumes the replication stream from the Primary.""" 
    while self.running: 
        try: 
            # Timeout to allow clean thread termination 
            change = self.replication_queue.get(timeout=0.5) 
            key, value = change 
            self.database[key] = value 
            print(f"✅ Replica {self.replica_id} | Applied update: {key} = {value}") 
        except queue.Empty: 
            continue 

def receive_change(self, key: str, value: any): 
    """Method called by the Primary to send a change.""" 
    self.replication_queue.put((key, value)) 

def read(self, key: str) -> any: 
    """Local read on the replica.""" 
    return self.database.get(key) 

def stop(self): 
    self.running = False 
    self.replication_thread.join(timeout=1) 

```

class PrimaryServer:
"""Primary server (central/master/leader). Controls replication."""

```
def __init__(self): 
    self.database: Dict[str, any] = {}          # Main database 
    self.replicas: List[ReplicaServer] = [] 
    print("🚀 Primary Server started - Controlling hierarchical replication") 

def add_replica(self, replica: ReplicaServer): 
    """Registers a secondary server in the topology.""" 
    self.replicas.append(replica) 
    print(f"➕ Replica {replica.replica_id} added to Primary (Total replicas: {len(self.replicas)})") 

def write(self, key: str, value: any): 
    """Write on the Primary + automatic replication to all secondaries.""" 
    # 1. Write to the central server 
    self.database[key] = value 
    print(f"📝 Primary | Write confirmed: {key} = {value}") 

    # 2. Propagate the change to all replicas (replication stream) 
    for replica in self.replicas: 
        replica.receive_change(key, value) 

def read(self, key: str) -> any: 
    """Direct read on the Primary (consistent read).""" 
    return self.database.get(key) 

def get_replica_status(self) -> Dict: 
    """Status of the hierarchical topology.""" 
    return { 
        "primary_data_size": len(self.database), 
        "replicas_count": len(self.replicas), 
        "replicas_status": [ 
            {"id": r.replica_id, "data_size": len(r.database)} 
            for r in self.replicas 
        ] 
    } 

```

# ======================== DEMONSTRATION ========================

if **name** == "**main**":
print("=" * 70)
print("PRIMARY-REPLICA ARCHITECTURE SIMULATION (1 Central → 3 Secondaries)")
print("=" * 70)

```
# 1. Instantiate the central server 
primary = PrimaryServer() 

# 2. Create and register three secondary servers 
replicas = [] 
for i in range(1, 4): 
    replica = ReplicaServer(replica_id=i) 
    primary.add_replica(replica) 
    replicas.append(replica) 

print("\n" + "-" * 50) 
print("EXECUTING DATABASE OPERATIONS") 
print("-" * 50) 

# 3. Write operations (controlled by the Primary) 
primary.write("user:1", {"name": "Ana", "age": 28}) 
primary.write("product:42", {"name": "Laptop", "price": 1299.99}) 
primary.write("config:theme", "dark") 

# Brief wait for replication threads to finish 
time.sleep(1.5) 

# 4. Verify consistency 
print("\n" + "=" * 50) 
print("TOPOLOGY CONSISTENCY CHECK") 
print("=" * 50) 

print("📊 Primary:", primary.read("user:1")) 
for replica in replicas: 
    print(f"📊 Replica {replica.replica_id}:", replica.read("user:1")) 

# 5. General architecture status 
print("\n📈 Final status of the hierarchical architecture:") 
import json 
print(json.dumps(primary.get_replica_status(), indent=2, ensure_ascii=False)) 

# 6. Cleanup 
for replica in replicas: 
    replica.stop() 

print("\n✅ Simulation finished successfully.") 

```

```

### How to Run

```bash
python primary_replica_architecture.py

```

This script is completely self-contained, requires no external databases, and clearly demonstrates the hierarchical relationship of **Primary → Replicas** using real asynchronous replication (implemented with queues and threads).

**Note:** If you need the production version for MySQL/PostgreSQL (using `pymysql`/`psycopg2`, `my.cnf` generation, `CHANGE MASTER TO` commands, etc.), just let me know and I will provide the production configuration script.

Respuesta:

# -*- coding: utf-8 -*- 
""" 
Script de simulación de arquitectura de base de datos PRIMARY-REPLICA 
==================================================================== 
 
Terminología técnica estándar utilizada: 
- Primary Server (también llamado Master o Leader): Servidor central que acepta 
  escrituras y controla la replicación. 
- Replica Server (también llamado Slave, Secondary o Follower): Servidores 
  secundarios que reciben actualizaciones del Primary y sirven lecturas. 
- Relación jerárquica: Topología de replicación asíncrona one-to-many (1 primario → N réplicas). 
- Replication stream: Flujo de cambios que el Primary propaga a las réplicas. 
 
Este script es una simulación educativa completa en Python puro. 
Para una implementación real en producción se utilizaría: 
  • MySQL → replicación master-slave (binlog + GTID) 
  • PostgreSQL → streaming replication (WAL + physical/logical replicas) 
  • MongoDB → replica set (primary + secondaries + arbiter) 
""" 
 
import threading 
import time 
import queue 
from typing import Dict, List 
 
 
class ReplicaServer: 
    """Servidor réplica (secondary/follower). Solo recibe actualizaciones.""" 
 
    def __init__(self, replica_id: int): 
        self.replica_id = replica_id 
        self.database: Dict[str, any] = {}          # Base de datos simulada (key-value) 
        self.replication_queue: queue.Queue = queue.Queue() 
        self.running = True 
 
        # Hilo dedicado para aplicar replicación asíncrona 
        self.replication_thread = threading.Thread( 
            target=self._apply_replication, daemon=True 
        ) 
        self.replication_thread.start() 
 
    def _apply_replication(self): 
        """Hilo que consume el replication stream del Primary.""" 
        while self.running: 
            try: 
                # Timeout para poder terminar el hilo limpiamente 
                change = self.replication_queue.get(timeout=0.5) 
                key, value = change 
                self.database[key] = value 
                print(f"✅ Réplica {self.replica_id} | Aplicada actualización: {key} = {value}") 
            except queue.Empty: 
                continue 
 
    def receive_change(self, key: str, value: any): 
        """Método llamado por el Primary para enviar un cambio.""" 
        self.replication_queue.put((key, value)) 
 
    def read(self, key: str) -> any: 
        """Lectura local en la réplica.""" 
        return self.database.get(key) 
 
    def stop(self): 
        self.running = False 
        self.replication_thread.join(timeout=1) 
 
 
class PrimaryServer: 
    """Servidor primario (central/master/leader). Controla la replicación.""" 
 
    def __init__(self): 
        self.database: Dict[str, any] = {}          # Base de datos principal 
        self.replicas: List[ReplicaServer] = [] 
        print("🚀 Servidor Primary iniciado - Controlando replicación jerárquica") 
 
    def add_replica(self, replica: ReplicaServer): 
        """Registra un servidor secundario en la topología.""" 
        self.replicas.append(replica) 
        print(f"➕ Réplica {replica.replica_id} añadida al Primary (total réplicas: {len(self.replicas)})") 
 
    def write(self, key: str, value: any): 
        """Escritura en el Primary + replicación automática a todos los secundarios.""" 
        # 1. Escribir en el servidor central 
        self.database[key] = value 
        print(f"📝 Primary | Escritura confirmada: {key} = {value}") 
 
        # 2. Propagar el cambio a todas las réplicas (replication stream) 
        for replica in self.replicas: 
            replica.receive_change(key, value) 
 
    def read(self, key: str) -> any: 
        """Lectura directa en el Primary (consistent read).""" 
        return self.database.get(key) 
 
    def get_replica_status(self) -> Dict: 
        """Estado de la topología jerárquica.""" 
        return { 
            "primary_data_size": len(self.database), 
            "replicas_count": len(self.replicas), 
            "replicas_status": [ 
                {"id": r.replica_id, "data_size": len(r.database)} 
                for r in self.replicas 
            ] 
        } 
 
 
# ======================== DEMOSTRACIÓN ======================== 
if __name__ == "__main__": 
    print("=" * 70) 
    print("SIMULACIÓN ARQUITECTURA PRIMARY-REPLICA (1 central → 3 secundarios)") 
    print("=" * 70) 
 
    # 1. Instanciar servidor central 
    primary = PrimaryServer() 
 
    # 2. Crear y registrar tres servidores secundarios 
    replicas = [] 
    for i in range(1, 4): 
        replica = ReplicaServer(replica_id=i) 
        primary.add_replica(replica) 
        replicas.append(replica) 
 
    print("\n" + "-" * 50) 
    print("EJECUTANDO OPERACIONES DE BASE DE DATOS") 
    print("-" * 50) 
 
    # 3. Operaciones de escritura (controladas por el Primary) 
    primary.write("usuario:1", {"nombre": "Ana", "edad": 28}) 
    primary.write("producto:42", {"nombre": "Laptop", "precio": 1299.99}) 
    primary.write("config:theme", "dark") 
 
    # Pequeña espera para que los hilos de replicación terminen 
    time.sleep(1.5) 
 
    # 4. Verificar consistencia 
    print("\n" + "=" * 50) 
    print("VERIFICACIÓN DE CONSISTENCIA EN LA TOPOLOGÍA") 
    print("=" * 50) 
 
    print("📊 Primary:", primary.read("usuario:1")) 
    for replica in replicas: 
        print(f"📊 Réplica {replica.replica_id}:", replica.read("usuario:1")) 
 
    # 5. Estado general de la arquitectura 
    print("\n📈 Estado final de la arquitectura jerárquica:") 
    import json 
    print(json.dumps(primary.get_replica_status(), indent=2, ensure_ascii=False)) 
 
    # 6. Limpieza 
    for replica in replicas: 
        replica.stop() 
 
    print("\n✅ Simulación finalizada correctamente.") 
  

Cómo ejecutar  

python arquitectura_primary_replica.py 
  

Este script es completamente autónomo, no requiere bases de datos externas y demuestra claramente la relación jerárquica primary → replicas con replicación asíncrona real (usando colas y hilos).  

Si necesitas la versión real para MySQL/PostgreSQL (con pymysql/psycopg2, generación de my.cnf, comandos CHANGE MASTER TO, etc.), avísame y te proporciono el script de configuración de producción. 