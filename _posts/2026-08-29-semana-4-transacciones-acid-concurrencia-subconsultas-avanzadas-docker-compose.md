---
title: "Semana 4: Subconsultas Avanzadas, Integridad Referencial en Cascada, Transacciones ACID, Concurrencia y Docker Compose"
date: 2026-08-29 10:00:00 -0500
categories: [Bases de Datos, Semana 4]
tags: [sql-avanzado, transacciones, acid, concurrencia, 2pl, mvcc, docker-compose, postgresql]
description: "Estudio riguroso de subconsultas avanzadas, acciones de integridad referencial autorreferenciales, teoría del procesamiento de transacciones ACID, fenómenos de concurrencia, bloqueo 2PL/MVCC y orquestación con Docker Compose."
toc: true
math: true
mermaid: true
pin: false
---

## 1. Subconsultas Avanzadas, Tablas Derivadas y Manipulación DML Compleja

En aplicaciones empresariales, las consultas y manipulaciones de datos trascienden las estructuras lineales simples. El estándar SQL permite incrustar consultas derivadas directamente en la cláusula `FROM` (conocidas formalmente como **tablas derivadas** o *inline views*), así como utilizar predicados existenciales correlacionados para gobernar transformaciones masivas en `UPDATE` y `DELETE`.

```mermaid
flowchart TD
    subgraph TABLA_DERIVADA["Tabla Derivada en FROM"]
        Q_INT["Subconsulta Interna (Calcula Agregados por Grupo)"] -->|Materializa relación intermedia| T_DER["Tabla Virtual (Alias Obligatorio)"]
    end
    subgraph CONSULTA_EXTERNA["Consulta Externa"]
        T_DER --> JOIN_OP["JOIN con Relación Base"]
        JOIN_OP --> PROY["Proyección Final de Atributos"]
    end
```

### 1.1 Tablas Derivadas en la Cláusula `FROM`
Cuando se requiere comparar atributos individuales contra agregados calculados en subconjuntos (por ejemplo, comparar el salario de un empleado contra el salario promedio de su propia dependencia), la inclusión de una subconsulta en la cláusula `FROM` evita subconsultas correlacionadas redundantes:

```sql
-- Empleados cuyo salario supera el promedio de su respectivo departamento
SELECT 
    E.cedula,
    E.nombre,
    E.dep_cod,
    E.salario,
    ROUND(DEP_STATS.salario_promedio, 2) AS promedio_departamento
FROM empleados E
INNER JOIN (
    SELECT 
        dep_cod, 
        AVG(salario) AS salario_promedio
    FROM empleados
    GROUP BY dep_cod
) AS DEP_STATS ON E.dep_cod = DEP_STATS.dep_cod
WHERE E.salario > DEP_STATS.salario_promedio;
```

---

### 1.2 Predicados de Existencia: `EXISTS` vs `NOT EXISTS`

El operador `EXISTS` evalúa si el conjunto de tuplas resultante de una subconsulta correlacionada es no vacío:

$$\text{EXISTS}(Q) = \begin{cases} \text{TRUE} & \text{si } |Q| > 0 \\ \text{FALSE} & \text{si } |Q| = 0 \end{cases}$$

```sql
-- Departamentos que NO cuentan con ningún empleado asignado (usando NOT EXISTS)
SELECT 
    D.dep_cod,
    D.nombre
FROM dependencias D
WHERE NOT EXISTS (
    SELECT 1 
    FROM empleados E 
    WHERE E.dep_cod = D.dep_cod
);
```

> [!TIP]
> A diferencia de `NOT IN`, que colapsa a `UNKNOWN` y descarta resultados si la subconsulta contiene un solo valor `NULL` (debido a la lógica trivaluada), `NOT EXISTS` es inmune a la presencia de nulos porque evalúa cardinalidad de tuplas físicas ($|Q| = 0$), garantizando un comportamiento determinista y seguro.

---

## 2. Integridad Referencial en Esquemas Jerárquicos y Acciones en Cascada

En modelos de datos con relaciones recursivas o autorreferenciales (e.g. una estructura jerárquica organizacional donde un empleado reporta a un supervisor que también es empleado), las reglas de integridad referencial gobiernan el ciclo de vida de los registros dependientes.

```mermaid
erDiagram
    EMPLEADOS ||--o{ EMPLEADOS : "supervisa (jefe_cedula)"
    EMPLEADOS {
        varchar cedula PK
        varchar nombre
        varchar cargo
        numeric salario
        varchar jefe_cedula FK
        integer dep_cod FK
    }
    DEPENDENCIAS ||--o{ EMPLEADOS : "pertenece"
    DEPENDENCIAS {
        integer dep_cod PK
        varchar nombre
    }
```

### Comportamiento de las Cláusulas de Integridad ante Operaciones `DELETE`

```sql
-- Definición formal de la relación autorreferencial
CREATE TABLE empleados_jerarquia (
    cedula VARCHAR(15) PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL,
    cargo VARCHAR(30) NOT NULL,
    salario NUMERIC(10, 2) NOT NULL,
    jefe_cedula VARCHAR(15),
    dep_cod INTEGER NOT NULL,
    CONSTRAINT fk_jefe FOREIGN KEY (jefe_cedula)
        REFERENCES empleados_jerarquia(cedula)
        ON DELETE SET NULL
        ON UPDATE CASCADE
);
```

| Acción `ON DELETE` | Mecánica de Ejecución | Impacto en la Jerarquía | Caso de Uso Recomendado |
| :--- | :--- | :--- | :--- |
| **`RESTRICT / NO ACTION`** | Aborta la transacción si existen tuplas hijas referenciando la tupla padre. | Impide la eliminación de supervisores con personal a cargo. | Preservación estricta de auditoría y bloqueos de seguridad. |
| **`CASCADE`** | Elimina recursivamente todas las tuplas hijas asociadas al registro padre. | **Peligro**: La eliminación de un gerente general borra a todos sus subordinados en cascada. | Tablas débilmente ligadas (e.g. `facturas` $\to$ `detalle_facturas`). |
| **`SET NULL`** | Asigna `NULL` a la clave foránea en las tuplas subordinadas. | Los empleados dependientes quedan temporalmente sin jefe asignado. | Estructuras organizacionales dinámicas y reasignaciones. |

> [!WARNING]
> En esquemas autorreferenciales, el uso inadvertido de `ON DELETE CASCADE` puede provocar el vaciado masivo e irreversible de tablas completas al eliminar un único nodo raíz de la jerarquía.

---

## 3. Procesamiento de Transacciones y Principios ACID

Una **Transacción** ($T$) es una unidad lógica de trabajo indivisible compuesta por una secuencia de operaciones de lectura ($r(X)$) y escritura ($w(X)$) sobre el almacenamiento persistente. Para preservar la integridad del sistema ante fallos de hardware, caídas de software o accesos concurrentes masivos, todo RDBMS transaccional debe garantizar formalmente las cuatro propiedades **ACID**:

```mermaid
flowchart LR
    subgraph ACID["Garantías Transaccionales ACID"]
        A["Atomicidad (A)<br/>Todo o Nada"]
        C["Consistencia (C)<br/>Preservación de Invariantes"]
        I["Aislamiento (I)<br/>Serializabilidad Concurrente"]
        D["Durabilidad (D)<br/>Persistencia ante Fallos (WAL)"]
    end
```

### 1. Atomicidad (Atomicity)
Garantiza que la transacción se ejecuta en su totalidad o no se ejecuta en absoluto (*All-or-Nothing*). Si ocurre cualquier error antes de alcanzar la instrucción `COMMIT`, el motor ejecuta un `ROLLBACK` automático, restaurando el estado previo de las tuplas modificadas.

$$\text{Transacción Transferencia Bancaria: } T = \langle r(A), A \gets A - 500, w(A), r(B), B \gets B + 500, w(B), \text{commit} \rangle$$

### 2. Consistencia (Consistency)
Garantiza que una transacción solo puede transformar la base de datos de un estado válido a otro estado válido, respetando estrictamente todas las restricciones de integridad de esquema (`PRIMARY KEY`, `FOREIGN KEY`, `CHECK`), disparadores y reglas de negocio.

### 3. Aislamiento (Isolation)
Establece que la ejecución concurrente de múltiples transacciones activas ($\{T_1, T_2, \dots, T_k\}$) no debe resultar en estados inconsistentes, garantizando que el efecto final sea equivalente a alguna ejecución estrictamente serial.

### 4. Durabilidad (Durability)
Asegura que una vez una transacción recibe la confirmación de `COMMIT`, sus modificaciones quedan permanentemente grabadas en medios no volátiles, sobreviviendo incluso a caídas intempestivas del servidor o cortes eléctricos. Esto se implementa mediante el **Write-Ahead Logging (WAL)**:

```mermaid
flowchart LR
    T_WRITE["Transacción w(X)"] --> WAL_BUFFER["Log Buffer en Memoria"]
    WAL_BUFFER -->|fsync Obligatorio| WAL_DISK[("Disco: WAL Log File")]
    WAL_DISK --> DB_BUFFERS["Shared Buffers de Datos"]
    DB_BUFFERS -.->|Checkpoint Periódico| DATA_FILES[("Archivos de Tablas en Disco")]
```

---

## 4. Control de Concurrencia, Bloqueos y Fenómenos Anómalos ANSI SQL

Cuando múltiples transacciones acceden y modifican concurrentemente los mismos bloques de datos, pueden manifestarse fenómenos de interferencia o anomalías de concurrencia.

### Fenómenos Anómalos de Concurrencia:

```mermaid
mindmap
  root((Anomalías de Concurrencia))
    Dirty_Read["Lectura Sucia: T2 lee datos modificados por T1 antes de que T1 haga ROLLBACK"]
    Non_Repeatable_Read["Lectura No Repetible: T1 lee una tupla, T2 modifica esa tupla y confirma; T1 relee y ve un valor diferente"]
    Phantom_Read["Lectura Fantasma: T1 ejecuta una consulta de rango, T2 inserta tuplas en ese rango y confirma; T1 relee y ve tuplas nuevas"]
    Lost_Update["Actualización Perdida: T1 y T2 leen el mismo dato y escriben basándose en él; la última escritura sobreescribe a la primera"]
```

---

### Niveles de Aislamiento ANSI SQL y Matriz de Fenómenos

| Nivel de Aislamiento | Lectura Sucia (*Dirty Read*) | Lectura No Repetible (*Non-Repeatable Read*) | Lectura Fantasma (*Phantom Read*) |
| :--- | :---: | :---: | :---: |
| **`READ UNCOMMITTED`** | **Permitida** | **Permitida** | **Permitida** |
| **`READ COMMITTED`** (Default en PostgreSQL) | **Prevenida** | **Permitida** | **Permitida** |
| **`REPEATABLE READ`** | **Prevenida** | **Prevenida** | **Permitida / Prevenida\*** |
| **`SERIALIZABLE`** | **Prevenida** | **Prevenida** | **Prevenida** |

*\*Nota: En PostgreSQL, la implementación de `REPEATABLE READ` basada en Snapshot Isolation previene tanto lecturas no repetibles como lecturas fantasma mediante control multiversión.*

---

### Mecanismos de Control de Concurrencia: Bloqueos (2PL) vs MVCC

#### 1. Protocolo de Bloqueo en Dos Fases (Two-Phase Locking - 2PL)
Emplea dos tipos de candados lógicos sobre los recursos:
- **Bloqueo Compartido (Shared Lock - $S$)**: Adquirido para lecturas; múltiples transacciones pueden retenerlo simultáneamente.
- **Bloqueo Exclusivo (Exclusive Lock - $X$)**: Adquirido para escrituras; confiere acceso exclusivo al recurso.

```mermaid
flowchart TD
    subgraph FASE_CRECIMIENTO["Fase 1: Crecimiento (Growing Phase)"]
        G1["Adquisición de bloqueos S y X"] --> G2["Prohibido liberar cualquier bloqueo"]
    end
    subgraph PUNTO_BLOQUEO["Punto de Bloqueo (Lock Point)"]
        LP["Máximo nivel de recursos adquiridos"]
    end
    subgraph FASE_CONTRACCION["Fase 2: Contracción (Shrinking Phase)"]
        S1["Liberación progresiva de bloqueos"] --> S2["Prohibido adquirir nuevos bloqueos"]
    end

    FASE_CRECIMIENTO --> PUNTO_BLOQUEO --> FASE_CONTRACCION
```

> [!WARNING]
> **Interbloqueo (Deadlock)**: Ocurre cuando $T_1$ retiene un bloqueo sobre el ítem $A$ y espera por $B$, mientras $T_2$ retiene $B$ y espera por $A$. El motor RDBMS monitorea continuamente el grafo de espera (*Wait-For Graph*) y aborta automáticamente la transacción más económica mediante un *Deadlock Detector*.

#### 2. Control de Concurrencia Multiversión (MVCC)
Los motores modernos como **PostgreSQL** utilizan **MVCC (Multi-Version Concurrency Control)**. En lugar de bloquear tuplas durante la lectura:
- Cada tupla almacena metadatos de visibilidad transaccional (`xmin` y `xmax`).
- Las lecturas obtienen una instantánea (*Snapshot*) coherente en el tiempo del estado de la base de datos.
- **Principio Fundamental**: *Los lectores nunca bloquean a los escritores, y los escritores nunca bloquean a los lectores*.

---

## 5. Modelado Práctico y Consultas Avanzadas: E-Commerce y Videojuegos

Para consolidar la teoría de consultas avanzadas, considere un esquema relacional para una plataforma de distribución de videojuegos:

```sql
-- Creación del catálogo maestro de videojuegos
CREATE TABLE videojuegos (
    videojuego_id SERIAL PRIMARY KEY,
    titulo VARCHAR(100) NOT NULL,
    genero VARCHAR(30) NOT NULL,
    precio_base NUMERIC(8, 2) NOT NULL CHECK (precio_base >= 0),
    descuento_porcentaje NUMERIC(5, 2) DEFAULT 0.0 CHECK (descuento_porcentaje BETWEEN 0 AND 100),
    clasificacion CHAR(1) NOT NULL CHECK (clasificacion IN ('E', 'T', 'M', 'A'))
);

-- Creación de la relación de detalle de ventas
CREATE TABLE ventas_detalle (
    venta_id INTEGER NOT NULL,
    videojuego_id INTEGER NOT NULL,
    cantidad INTEGER NOT NULL CHECK (cantidad > 0),
    precio_unitario_cobrado NUMERIC(8, 2) NOT NULL,
    PRIMARY KEY (venta_id, videojuego_id),
    FOREIGN KEY (videojuego_id) REFERENCES videojuegos(videojuego_id)
);
```

### Consultas Analíticas con Cálculos Condicionales y Filtros de Texto:

```sql
-- Consulta analítica: Cálculo de precios finales, rangos y categorización
SELECT 
    titulo,
    genero,
    precio_base,
    descuento_porcentaje,
    ROUND(precio_base * (1 - (descuento_porcentaje / 100.0)), 2) AS precio_con_descuento,
    CASE 
        WHEN clasificacion = 'E' THEN 'Todo Público'
        WHEN clasificacion = 'T' THEN 'Adolescentes'
        WHEN clasificacion = 'M' THEN 'Maduro (+17)'
        ELSE 'Adultos Únicamente'
    END AS segmento_audiencia
FROM videojuegos
WHERE (titulo ILIKE '%Special Edition%' OR titulo ILIKE '%Remastered%')
  AND precio_base > 20.00
ORDER BY precio_con_descuento DESC;
```

---

## 6. Arquitecturas Multicapa, Conectividad Programática y Docker Compose

En la ingeniería de software actual, las bases de datos no operan aisladas, sino como nodos de persistencia en arquitecturas multicapa integradas con servicios de backend y sistemas de caché en memoria.

```mermaid
flowchart LR
    CLIENTE["Cliente Web / Móvil"] -->|HTTP REST| BACKEND["Backend API (FastAPI / Node.js)"]
    BACKEND -->|TCP Pool 5432| POSTGRES[("RDBMS (PostgreSQL)")]
    BACKEND -->|TCP 6379| REDIS[("Caché In-Memory (Redis)")]
    
    subgraph DOCKER_COMPOSE["Orquestación Docker Compose (Red Bridge Compartida)"]
        BACKEND
        POSTGRES
        REDIS
    end
```

---

### Orquestación Declarativa de Microservicios con `docker-compose.yml`

El archivo `docker-compose.yml` permite orquestar de manera reproducible la pila tecnológica completa:

```yaml
version: '3.8'

services:
  database:
    image: postgres:16-alpine
    container_name: udea_postgres_srv
    environment:
      POSTGRES_DB: ecommerce_db
      POSTGRES_USER: dbadmin
      POSTGRES_PASSWORD: SecureDatabasePassword2026!
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app_network
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    container_name: udea_redis_srv
    ports:
      - "6379:6379"
    networks:
      - app_network

  api_backend:
    build: ./backend
    container_name: udea_api_srv
    environment:
      DATABASE_URL: postgresql://dbadmin:SecureDatabasePassword2026!@database:5432/ecommerce_db
      REDIS_URL: redis://cache:6379/0
    ports:
      - "8000:8000"
    depends_on:
      - database
      - cache
    networks:
      - app_network

volumes:
  postgres_data:
    driver: local

networks:
  app_network:
    driver: bridge
```

---

### Conectividad Programática y Prevención de Inyección SQL

La interacción con el motor RDBMS desde lenguajes de alto nivel (como Python mediante bibliotecas como `psycopg2` o `asyncpg`) requiere gestionar cursores y utilizar **consultas parametrizadas (Prepared Statements)** para inmunizar el sistema contra vulnerabilidades de Inyección SQL.

```python
import psycopg2
from psycopg2 import sql

def registrar_venta(cliente_id: int, videojuego_id: int, cantidad: int):
    # Conexión al cluster RDBMS orquestado en Docker
    conn = psycopg2.connect(
        dbname="ecommerce_db",
        user="dbadmin",
        password="SecureDatabasePassword2026!",
        host="localhost",
        port=5432
    )
    
    try:
        # Inicio implícito de transacción
        with conn.cursor() as cursor:
            # 1. Obtener precio actual (Consulta parametrizada segura)
            cursor.execute(
                "SELECT precio_base FROM videojuegos WHERE videojuego_id = %s;",
                (videojuego_id,)
            )
            resultado = cursor.fetchone()
            if not resultado:
                raise ValueError("El videojuego especificado no existe.")
            
            precio_unitario = resultado[0]
            
            # 2. Insertar registro en detalle de ventas
            cursor.execute(
                """
                INSERT INTO ventas_detalle (venta_id, videojuego_id, cantidad, precio_unitario_cobrado)
                VALUES (%s, %s, %s, %s);
                """,
                (101, videojuego_id, cantidad, precio_unitario)
            )
            
        # Confirmación atómica de la transacción
        conn.commit()
        print("Transacción confirmada exitosamente.")
        
    except Exception as error:
        # Reversión total ante anomalías
        conn.rollback()
        print(f"Fallo transaccional detectado. Rollback ejecutado: {error}")
    finally:
        conn.close()
```

> [!IMPORTANT]
> El uso de placeholders parametrizados (`%s` en `psycopg2`) garantiza que los argumentos sean tratados como literales de datos puros por el parser SQL del motor, imposibilitando la alteración maliciosa de la estructura sintáctica del comando.

---

## 7. Pregunta de la Semana

> **Pregunta:**  
> En un sistema transaccional de alta concurrencia que opera bajo el nivel de aislamiento `READ COMMITTED` mediante el mecanismo MVCC (como PostgreSQL), ¿cómo se previene la anomalía de *Actualización Perdida (Lost Update)* cuando dos transacciones concurrentes ejecutan simultáneamente una operación de débito sobre el mismo saldo bancario, y por qué un simple `SELECT` seguido de un `UPDATE` en la capa de aplicación resulta vulnerable?
>
> **Respuesta:**  
> Si una aplicación ejecuta un flujo ingenuo compuesto por:
> 1. `SELECT saldo FROM cuentas WHERE cuenta_id = 1;` (lee $1000)
> 2. Calcula localmente en la memoria de la aplicación: `nuevo_saldo = 1000 - 200 = 800`
> 3. `UPDATE cuentas SET saldo = 800 WHERE cuenta_id = 1;`
>
> Dos transacciones concurrentes ($T_1$ y $T_2$) pueden leer ambas el saldo inicial de $1000$ bajo MVCC (ya que los lectores no se bloquean entre sí ni leen bloqueos de escritura no confirmados). Si $T_1$ resta $200$ y actualiza a $800$, y $T_2$ resta $300$ y actualiza a $700$, la última transacción que confirme sobrescribirá ciegamente el cambio de la primera, perdiéndose uno de los débitos.
>
> **Soluciones de Ingeniería:**
> 1. **Actualización Atómica In-Place**: Realizar el cómputo directamente en el motor:
>    ```sql
>    UPDATE cuentas SET saldo = saldo - 200 WHERE cuenta_id = 1;
>    ```
>    El motor RDBMS adquiere automáticamente un bloqueo exclusivo a nivel de fila (`FOR UPDATE`), forzando a la segunda transacción a re-evaluar la condición sobre la versión más reciente de la tupla.
> 2. **Bloqueo Pesimista Explícito**:
>    ```sql
>    SELECT saldo FROM cuentas WHERE cuenta_id = 1 FOR UPDATE;
>    ```
>    Esto adquiere un bloqueo exclusivo de lectura/escritura sobre la fila desde el primer instante, serializando el acceso de transacciones competidoras.

---

## 8. Referencias y Enlaces de Interés

1. **Gray, J. (1981)**: *The Transaction Concept: Virtues and Limitations*. Proceedings of the 7th International Conference on Very Large Data Bases (VLDB), 144–154. [http://hdl.handle.net/10068/491563](http://hdl.handle.net/10068/491563).
2. **PostgreSQL Documentation**: *Transaction Isolation, Explicit Locking and Concurrency Control*. [https://www.postgresql.org/docs/current/mvcc.html](https://www.postgresql.org/docs/current/mvcc.html).
3. **Bernstein, P. A., Hadzilacos, V., & Goodman, N. (1987)**: *Concurrency Control and Recovery in Database Systems*. Addison-Wesley.
4. **Docker Documentation**: *Compose file version 3 reference and Multi-Container Application Architecture*. [https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/).
5. **Silberschatz, A., Korth, H. F., & Sudarshan, S. (2020)**: *Database System Concepts* (7th ed.), Chapter 14: Transactions, Chapter 15: Concurrency Control. McGraw-Hill Education.

---

*Publicado por [Mateo Vargas Tirado](https://github.com/araigumadev) como parte del registro de aprendizaje en el curso Bases de Datos 2026/2 - Universidad de Antioquia.*
