---
title: "Semana 14: Bases de Datos No Relacionales (NoSQL), Arquitectura de MongoDB, BSON, Modelado de Documentos y Pipelines de Agregación"
date: 2026-11-07 10:00:00 -0500
categories: [Bases de Datos, Semana 14]
tags: [nosql, mongodb, bson, modelado-documental, aggregation-pipeline, indices, rag, bases-de-datos]
description: "Estudio técnico del paradigma NoSQL orientado a documentos con MongoDB: representación binaria BSON y estructura del ObjectId, patrones de diseño (Embedding vs. Referencing), operaciones CRUD en mongosh, tuberías de agregación e indexación especializada (texto, geoespacial 2dsphere y búsqueda vectorial para RAG)."
toc: true
math: true
mermaid: true
pin: false
---

## 1. El Paradigma NoSQL (*Not Only SQL*) y Bases de Datos Orientadas a Documentos

El término **NoSQL** (*Not Only SQL*) engloba una familia de motores de almacenamiento diseñados para superar las limitaciones de rigidez esquemática y escalabilidad horizontal de los Sistemas de Gestión de Bases de Datos Relacionales (RDBMS) tradicionales cuando se enfrentan a cargas de trabajo distribuidas, datos semiestructurados o esquemas altamente evolutivos.

Dentro de la taxonomía NoSQL —que incluye almacenes clave-valor (Redis, DynamoDB), motores orientados a familias de columnas anchas (Cassandra, HBase) y bases de datos de grafos (Neo4j)—, las **bases de datos orientadas a documentos** como **MongoDB** almacenan la información en estructuras jerárquicas autodescriptivas (documentos JSON/BSON) agrupadas en **colecciones**.

### 1.1. Contraste Arquitectónico: Descomposición Relacional vs. Agregado Documental

En el modelo relacional clásico, la aplicación de las Formas Normales (1FN, 2FN, 3FN, BCNF) exige descomponer una entidad del dominio en múltiples tablas para eliminar grupos repetitivos y dependencias transitivas. Por ejemplo, modelar el perfil completo de un usuario con múltiples direcciones, teléfonos, intereses y redes sociales requiere hasta **seis tablas normalizadas** vinculadas mediante claves foráneas:

- `USUARIOS(`<u>`id_usuario`</u>`, nombre, email, fecha_registro)`
- `DIRECCIONES(`<u>`id_direccion`</u>`, id_usuario, calle, ciudad, pais)`
- `TELEFONOS(`<u>`id_telefono`</u>`, id_usuario, tipo, numero)`
- `INTERESES(`<u>`id_interes`</u>`, nombre_interes)`
- `USUARIO_INTERESES(`<u>`id_usuario, id_interes`</u>`)`
- `REDES_SOCIALES(`<u>`id_red`</u>`, id_usuario, plataforma, handle)`

Reconstruir el perfil en memoria para enviarlo a una aplicación cliente obliga al motor relacional a evaluar una consulta con múltiples uniones:

$$U_{\text{completo}} = R_{\text{usuarios}} \bowtie R_{\text{direcciones}} \bowtie R_{\text{telefonos}} \bowtie (R_{\text{usrIntereses}} \bowtie R_{\text{intereses}}) \bowtie R_{\text{redes}}$$

En contraste, el modelo orientado a documentos trata al perfil del usuario como un **agregado cohesivo**. Toda la jerarquía relacionada se almacena de forma contigua en un único documento con subdocumentos y arreglos embebidos:

```mermaid
flowchart LR
    subgraph Relacional["Modelo Relacional (Normalizado - 6 Tablas)"]
        direction TB
        T1["USUARIOS"]
        T2["DIRECCIONES"]
        T3["TELEFONOS"]
        T4["USUARIO_INTERESES"]
        T5["INTERESES"]
        T6["REDES_SOCIALES"]
        T1 -->|JOIN PK/FK| T2
        T1 -->|JOIN PK/FK| T3
        T1 -->|JOIN PK/FK| T4
        T4 -->|JOIN PK/FK| T5
        T1 -->|JOIN PK/FK| T6
    end

    subgraph Documental["Modelo Documental en MongoDB (1 Documento BSON)"]
        direction TB
        D1["Documento Usuario {_id: ObjectId(...)}<br/>├── nombre, email<br/>├── direcciones: [ {calle, ciudad}, ... ]<br/>├── telefonos: [ '300...', '604...' ]<br/>├── intereses: [ 'IA', 'Bases de Datos' ]<br/>└── redes_sociales: { github: '...', linkedin: '...' }"]
    end

    Relacional -.->|"Transformación a agregado<br/>(Lectura en 1 sola operación I/O)"| Documental

    style Relacional fill:#fff3e0,stroke:#e65100
    style Documental fill:#e8f5e9,stroke:#2e7d32
```

> **Eliminación del Desajuste de Impedancia (*Object-Relational Impedance Mismatch*)**: En el desarrollo de software orientado a objetos o funcional (JavaScript/TypeScript, Python, Java, Go), las estructuras de datos en memoria son grafos jerárquicos de objetos y listas. Al persistir directamente documentos JSON/BSON, MongoDB elimina la sobrecarga de traducción entre filas planas y objetos de dominio que tradicionalmente realizan los ORMs (*Object-Relational Mappers*).

### 1.2. Criterios de Decisión: ¿Cuándo elegir MongoDB frente a un SGBD Relacional?

Ningún paradigma sustituye universalmente al otro; la elección depende de los patrones de acceso, la volatilidad del esquema y las garantías transaccionales requeridas:

| Dimensión de Ingeniería | Base de Datos Documental (MongoDB) | Base de Datos Relacional (PostgreSQL, MySQL, Oracle) |
| :--- | :--- | :--- |
| **Estructura del esquema** | **Esquema flexible / polimórfico**: Documentos de una misma colección pueden poseer atributos distintos (ej. catálogo de *e-commerce* donde un monitor tiene `panel: "IPS"` y una prenda tiene `talla: "M"`). | **Esquema estático y homogéneo**: Todas las tuplas de una tabla cumplen exactamente la misma cabecera $R(A_1, A_2, \dots, A_n)$. |
| **Patrón de lectura dominante** | Lectura de entidades completas o agregados jerárquicos en una única operación de E/S sin `JOIN`. | Consultas analíticas *ad-hoc* que cruzan múltiples dimensiones independientes mediante `JOIN` complejos. |
| **Escalabilidad** | Diseñada nativamente para **escalamiento horizontal (*Sharding*)** y alta disponibilidad mediante *Replica Sets*. | Históricamente orientada a **escalamiento vertical (*Scale-up*)**, aunque existen extensiones distribuidas. |
| **Casos de uso ideales** | Catálogos de productos polimórficos, gestores de contenido (CMS), perfiles de usuario, telemetría IoT, geolocalización en tiempo real y almacenes vectoriales para arquitecturas **RAG / LLMs**. | Núcleos contables y financieros (*Core Banking*), facturación electrónica, nómina e inventarios altamente normalizados con reglas estrictas de integridad referencial. |

---

## 2. Arquitectura Interna de Datos: JSON, BSON y Anatomía del `ObjectId`

Aunque el desarrollador interactúa con MongoDB utilizando la sintaxis legible de **JSON (*JavaScript Object Notation*)** o diccionarios nativos del lenguaje de programación, el motor de almacenamiento (**WiredTiger**) serializa y persiste los datos en disco y a través de la red en formato **BSON (*Binary JSON*)**.

### 2.1. Representación Binaria BSON y Límite de $16\text{ MB}$

JSON puro presenta tres deficiencias críticas para un motor de base de datos de alto rendimiento:
1. Es un formato textual que requiere *parsing* carácter por carácter para localizar un campo anidado.
2. Solo soporta un conjunto reducido de tipos primitivos (`string`, `number`, `boolean`, `null`, `array`, `object`), sin distinguir entre enteros de $32\text{ bits}$, enteros de $64\text{ bits}$, flotantes IEEE 754 o decimales de precisión exacta.
3. No posee tipos nativos para fechas (`Date`), datos binarios crudos (`BinData`) ni identificadores únicos (`ObjectId`).

**BSON** resuelve estas limitaciones codificando cada documento en binario con **prefijos explícitos de longitud en bytes** al inicio de cada objeto, cadena y arreglo. Esto permite al motor saltar directamente en memoria hacia un campo específico sin decodificar los campos precedentes (*Fast Scan*).

$$S_{\text{max}} \le 16\text{ MB} \quad (16{,}777{,}216\text{ bytes})$$

> **Razón técnica del límite de $16\text{ MB}$**: Este umbral físico impide que un único documento monopolice la memoria RAM del servidor o sature el ancho de banda de red durante la replicación y transferencia hacia los clientes. Si un activo supera los $16\text{ MB}$ (como videos, audio o modelos pesados), MongoDB provee la especificación **GridFS**, que fragmenta el archivo en trozos (*chunks*) binarios de $255\text{ KB}$ almacenados en una colección dedicada.

#### Tipos de Datos Nativos en BSON

| Tipo BSON | Representación / Ejemplo en `mongosh` | Uso Técnico Recomendado |
| :--- | :--- | :--- |
| **`ObjectId`** | `ObjectId("65b3f1a2c9d4e50012a3b4c5")` | Clave primaria única distribuida de $12\text{ bytes}$. |
| **`String`** | `"Laptop Pro 14"` | Cadenas codificadas en UTF-8. |
| **`Int32` / `Int64`** | `NumberInt(25)` / `NumberLong(9007199254740991)` | Contadores, cantidades enteras e identificadores secuenciales. |
| **`Double`** | `1299.99` | Punto flotante de $64\text{ bits}$ (aproximado, estándar en JavaScript). |
| **`Decimal128`** | `NumberDecimal("1299.99")` | Punto flotante decimal IEEE 754-2008 de $128\text{ bits}$ para **valores monetarios y financieros exactos**. |
| **`Date` (`ISODate`)** | `ISODate("2026-11-07T15:00:00Z")` | Entero de $64\text{ bits}$ que representa milisegundos desde el *Unix Epoch* (1 de enero de 1970 UTC). |
| **`Boolean`** | `true` / `false` | Banderas lógicas de $1\text{ byte}$. |
| **`Array`** | `["electronica", "portatil", "oferta"]` | Listas ordenadas de valores primitivos o subdocumentos. |
| **`Object`** | `{ panel: "IPS", resolucion: "4K" }` | Subdocumentos embebidos anidados. |
| **`BinData` / `RegExp`** | `BinData(...)` / `/^prod/i` | Flujos de bytes binarios y expresiones regulares compiladas. |

### 2.2. Anatomía Matemática del `_id` y el Tipo `ObjectId`

Todo documento en MongoDB **debe** poseer un campo inmutable llamado `_id` que actúa como clave primaria de la colección (con un índice único `B-Tree` creado automáticamente). Si la aplicación no suministra un `_id` explícito al insertar el documento, el driver genera un **`ObjectId` de $12\text{ bytes}$ ($96\text{ bits}$)** representado habitualmente como una cadena hexadecimal de $24$ caracteres.

La estructura de los $12\text{ bytes}$ se divide en tres segmentos contiguos:

$$\underbrace{\text{4 bytes (32 bits)}}_{\text{Timestamp UNIX (segundos)}} \;\Vert\; \underbrace{\text{5 bytes (40 bits)}}_{\text{Valor aleatorio por proceso/nodo}} \;\Vert\; \underbrace{\text{3 bytes (24 bits)}}_{\text{Contador incremental}}$$

1. **Timestamp ($4\text{ bytes}$)**: Segundos transcurridos desde el *Unix Epoch* en formato *big-endian*. Como se ubica en los bytes más significativos, los `ObjectId` quedan **aproximadamente ordenados de forma cronológica** por instante de creación, y permiten extraer la fecha exacta de inserción mediante `.getTimestamp()` sin necesidad de almacenar un campo `created_at` adicional.
2. **Identificador Aleatorio ($5\text{ bytes}$)**: Valor pseudoaleatorio generado una vez por proceso e instancia de cliente/servidor.
3. **Contador Incremental ($3\text{ bytes}$)**: Contador inicializado con un valor aleatorio que se incrementa monotónicamente ($\bmod\, 2^{24}$) ante múltiples documentos creados dentro del mismo segundo en el mismo proceso, permitiendo hasta $2^{24} = 16{,}777{,}216$ identificadores únicos por segundo y por proceso sin colisión.

> **Ventaja en Sistemas Distribuidos**: A diferencia del clásico `SERIAL` o `AUTO_INCREMENT` de SQL, que requiere coordinación centralizada y bloqueos (*locks*) en el servidor de base de datos para asignar el siguiente entero, el `ObjectId` puede generarse de forma **totalmente descentralizada en los nodos cliente (drivers)** con probabilidad de colisión virtualmente nula.

### 2.3. Equivalencias Terminológicas: SQL Relacional vs. MongoDB

| Concepto en SGBD Relacional (SQL) | Equivalente en MongoDB | Notas Técnicas |
| :--- | :--- | :--- |
| **Base de Datos (`Database`)** | **Base de Datos (`Database`)** | Contenedor lógico de aislamiento y permisos. |
| **Tabla (`Table` / Relación)** | **Colección (`Collection`)** | Agrupación de documentos sin esquema rígido obligatorio por defecto. |
| **Fila / Tupla (`Row` / `Tuple`)** | **Documento (`Document`)** | Objeto BSON de hasta $16\text{ MB}$. |
| **Columna / Atributo (`Column`)** | **Campo / Llave (`Field`)** | Par `clave: valor` que puede contener primitivos, arreglos o subdocumentos. |
| **Clave Primaria (`Primary Key`)** | **Campo `_id`** | Obligatorio, único e inmutable dentro de la colección. |
| **Índice (`B-Tree Index`)** | **Índice (`Index`)** | Estructuras `B-Tree`, texto, `2dsphere`, TTL, *hashed* y vectoriales. |
| **`JOIN` entre tablas** | **Documentos Embebidos** o etapa **`$lookup`** | Se prioriza embeber datos que se consultan juntos; `$lookup` realiza un *Left Outer Join* entre colecciones. |
| **`GROUP BY` / Funciones de agregación** | **`Aggregation Pipeline`** (`$group`, `$match`, `$project`) | Tubería de procesamiento multi-etapa. |

---

## 3. Modelado de Datos en MongoDB: Embeber (*Embedding*) vs. Referenciar (*Referencing*) y Antipatrones

El principio fundamental del modelado documental establece que **"los datos que se consultan juntos deben almacenarse juntos"**. A diferencia del diseño relacional —que se guía exclusivamente por la estructura intrínseca de los datos para evitar redundancia—, el diseño en MongoDB se guía por los **patrones de consulta y actualización de la aplicación**.

### 3.1. Estrategia 1: Documentos Embebidos (*Desnormalización Controlada*)

Consiste en anidar objetos o arreglos de subdocumentos directamente dentro del documento padre. Es la estrategia óptima para relaciones **$1:1$** y **$1:N$ acotadas** (donde $N$ es pequeño y tiene una cota superior predecible), cuyos subdocumentos no necesitan consultarse ni modificarse de manera aislada fuera del contexto del padre.

```javascript
// Colección: posts (Relación 1:N acotada con comentarios embebidos)
{
  _id: ObjectId("672c9a10f4a12b0015d8e001"),
  titulo: "Arquitectura de Motores NoSQL",
  autor: {
    nombre: "Ana Gómez",
    rol: "Arquitecta de Datos"
  },
  fecha_publicacion: ISODate("2026-11-07T14:30:00Z"),
  tags: ["nosql", "mongodb", "bson"],
  comentarios: [
    {
      usuario: "carlos_dev",
      texto: "Excelente explicación sobre la estructura de 12 bytes del ObjectId.",
      fecha: ISODate("2026-11-07T15:10:00Z")
    },
    {
      usuario: "laura_data",
      texto: "Muy claro el contraste frente a esquemas en 3FN.",
      fecha: ISODate("2026-11-07T16:05:00Z")
    }
  ]
}
```

- **Ventajas**: Lectura de la entidad completa en una única búsqueda en el índice `_id` (alta localidad espacial en disco) y **atomicidad garantizada a nivel de documento único** sin necesidad de transacciones multi-documento.
- **Limitación**: Si el arreglo anidado crece indefinidamente (por ejemplo, millones de eventos de log o miles de comentarios en una publicación viral), el documento se aproximará al límite de $16\text{ MB}$ y provocará costosas reubicaciones en disco y memoria.

### 3.2. Estrategia 2: Referencias Normalizadas y Modelo Híbrido

Cuando existe una relación **$1:N$ de alta cardinalidad** o **$M:N$**, o cuando la entidad relacionada cambia con frecuencia y es compartida por múltiples documentos (por ejemplo, la información de un cliente en múltiples pedidos de compra), duplicar todo el objeto del cliente en cada pedido generaría **anomalías de actualización** idénticas a las estudiadas en la teoría de normalización.

En estos escenarios se aplica un **modelo híbrido**:
1. Se **referencia** al cliente almacenando únicamente su `customer_id` (`ObjectId`) en la colección `orders`, manteniendo los datos maestros del cliente en la colección `customers`.
2. Se **embebe** el arreglo de `items` comprados dentro de cada documento de `orders`, capturando una instantánea inmutable (*snapshot*) del nombre del producto, cantidad y precio unitario vigente en el momento de la compra.

```mermaid
flowchart TD
    subgraph Customers["Colección: customers"]
        C1["{_id: ObjectId('...c01'),<br/>name: 'Carlos Ruiz',<br/>email: 'carlos@correo.co',<br/>city: 'Medellín'}"]
    end

    subgraph Orders["Colección: orders (Modelo Híbrido)"]
        O1["{_id: ObjectId('...f99'),<br/>customer_id: ObjectId('...c01'),<br/>order_date: ISODate('2026-11-07'),<br/>status: 'completed',<br/>items: [<br/>  {product_id: ObjectId('...p01'), name: 'Monitor 27', qty: 1, price: 320},<br/>  {product_id: ObjectId('...p02'), name: 'Teclado Mecánico', qty: 2, price: 85}<br/>]}"]
    end

    subgraph Products["Colección: products"]
        P1["{_id: ObjectId('...p01'),<br/>name: 'Monitor 27',<br/>stock: 15,<br/>specs: {panel: 'IPS', hz: 144}}"]
        P2["{_id: ObjectId('...p02'),<br/>name: 'Teclado Mecánico',<br/>stock: 40,<br/>specs: {switch: 'Brown', rgb: true}}"]
    end

    O1 -->|"Referencia (ObjectId)<br/>Unión bajo demanda con $lookup"| C1
    O1 -.->|"Snapshot embebido en items[]<br/>+ Referencia product_id"| P1
    O1 -.->|"Snapshot embebido en items[]<br/>+ Referencia product_id"| P2

    style Customers fill:#e3f2fd,stroke:#1565c0
    style Orders fill:#fff8e1,stroke:#f57f17
    style Products fill:#e8f5e9,stroke:#2e7d32
```

### 3.3. Antipatrones Críticos de Diseño en MongoDB

1. **Arreglos sin límite (*Unbounded Arrays*)**: Agregar elementos indefinidamente con `$push` a un arreglo dentro de un mismo documento (ej. historial completo de transacciones o lecturas de un sensor IoT). Con el tiempo supera los $16\text{ MB}$ y degrada el rendimiento de escritura e indexación. *Solución*: Aplicar el **Bucket Pattern** (agrupar lecturas por ventana temporal de 1 hora o 1 día por documento) o separar el detalle en una colección referenciada.
2. **Documentos monolíticos gigantes**: Concentrar información que rara vez se consulta junta en un solo documento enorme, desperdiciando memoria RAM en el caché de WiredTiger y ancho de banda de red.
3. **Hiper-normalización relacional en MongoDB**: Diseñar colecciones aplicando ciegamente 3FN/BCNF como si fuera SQL y depender de 5 o 6 etapas `$lookup` en cada lectura de la aplicación. Si un dominio requiere uniones relacionales constantes sobre todas sus entidades, debe evaluarse un motor SQL o rediseñar los agregados documentales.
4. **Exceso de índices (*Over-indexing*)**: Cada índice secundario en una colección acelera las lecturas pero penaliza directamente las operaciones de escritura (`insert`, `update`, `delete`), ya que el motor debe reequilibrar múltiples árboles B en cada mutación.
5. **Ausencia de validación de esquema en producción**: Que el esquema sea *flexible* no significa que carezca de gobernanza. MongoDB permite definir **JSON Schema Validators** nativos a nivel de colección (`$jsonSchema`) para exigir tipos de datos, rangos numéricos y campos obligatorios antes de aceptar una escritura.

---

## 4. Operaciones CRUD, Operadores de Consulta y Proyecciones en `mongosh`

El despliegue de MongoDB puede realizarse localmente mediante **MongoDB Community Server** (escuchando por defecto en `mongodb://localhost:27017` e inspeccionado con **MongoDB Compass** o la consola interactiva **`mongosh`**) o en la nube mediante **MongoDB Atlas** sobre proveedores como AWS, GCP o Azure.

> **Seguridad Arquitectónica en Conexiones de Base de Datos**: La cadena de conexión (`mongodb+srv://usuario:password@cluster...`) **jamás debe exponerse ni invocarse desde el código Frontend** (navegador o aplicación móvil), ya que cualquier usuario podría extraer las credenciales e inyectar operaciones destructivas. Toda interacción con MongoDB debe ejecutarse exclusivamente en la capa de **Backend / API**, gestionando las credenciales mediante variables de entorno seguras.

### 4.1. Creación e Inserción de Documentos (`insertOne`, `insertMany`)

En MongoDB no es necesario ejecutar sentencias `CREATE DATABASE` ni `CREATE TABLE` previas; la base de datos y las colecciones se materializan dinámicamente al insertar el primer documento:

```javascript
// Seleccionar o crear la base de datos de comercio electrónico
use ecommerce_db;

// 1. Insertar un único producto con insertOne()
db.products.insertOne({
  name: "Monitor UltraWide 27",
  category: "Perifericos",
  price: 320.00,
  stock: 15,
  tags: ["monitor", "ips", "gaming", "2k"],
  specs: {
    panel: "IPS",
    refresh_rate_hz: 144,
    ports: ["HDMI 2.1", "DisplayPort 1.4"]
  },
  created_at: new Date()
});

// 2. Insertar múltiples productos con esquemas polimórficos en specs usando insertMany()
db.products.insertMany([
  {
    name: "Teclado Mecanico RGB",
    category: "Perifericos",
    price: 85.50,
    stock: 40,
    tags: ["teclado", "mecanico", "rgb"],
    specs: {
      switch_type: "Tactile Brown",
      wireless: true,
      layout: "ANSI"
    },
    created_at: new Date()
  },
  {
    name: "Workstation Pro 16",
    category: "Computadores",
    price: 1850.00,
    stock: 8,
    tags: ["laptop", "workstation", "nvme"],
    specs: {
      cpu: "12-Core ARM",
      ram_gb: 32,
      storage_tb: 1,
      panel: "Mini-LED"
    },
    created_at: new Date()
  },
  {
    name: "Mouse Ergonomico Vertical",
    category: "Perifericos",
    price: 45.00,
    stock: 60,
    tags: ["mouse", "ergonomico", "wireless"],
    specs: {
      dpi: 4000,
      wireless: true
    },
    created_at: new Date()
  }
]);
```

### 4.2. Lectura, Operadores de Filtrado y Proyección (`find`, `findOne`, `countDocuments`)

El método `db.coleccion.find(filtro, proyeccion)` equivale a la combinación de las operaciones de selección ($\sigma_{\text{filtro}}$) y proyección ($\pi_{\text{proyeccion}}$) del álgebra relacional:

$$\pi_{\text{campos}}\big(\sigma_{\text{predicado}}(\text{Coleccion})\big) \;\Longleftrightarrow\; \texttt{db.coleccion.find(predicado, campos)}$$

#### Tabla de Operadores de Consulta Frecuentes

| Categoría | Operador | Significado Técnico | Ejemplo en `mongosh` |
| :--- | :--- | :--- | :--- |
| **Comparación** | `$eq`, `$ne` | Igual a ($=$) / Diferente de ($\neq$) | `{ category: { $ne: "Computadores" } }` |
| **Comparación** | `$gt`, `$gte`, `$lt`, `$lte` | $>$, $\ge$, $<$, $\le$ | `{ price: { $gte: 50, $lte: 400 } }` |
| **Conjuntos** | `$in`, `$nin` | Pertenece ($\in$) / No pertenece ($\notin$) a una lista | `{ category: { $in: ["Perifericos", "Audio"] } }` |
| **Lógicos** | `$and`, `$or`, `$not`, `$nor` | Conjunción, disyunción y negación lógica | `{ $or: [ { price: { $lt: 50 } }, { stock: { $gt: 30 } } ] }` |
| **Arreglos** | `$all`, `$elemMatch`, `$size` | Contiene todos los elementos / Cumple condición compuesta | `{ tags: { $all: ["wireless", "ergonomico"] } }` |
| **Texto / Patrones** | `$regex` | Evaluación de expresión regular | `{ name: { $regex: /^monitor/i } }` |

```javascript
// 1. Consulta con rango numérico ($gte, $lte) y proyección explícita
// En la proyección: 1 incluye el campo, 0 excluye el campo (_id se incluye siempre salvo que se indique _id: 0)
db.products.find(
  { price: { $gte: 50, $lte: 500 } },
  { _id: 0, name: 1, price: 1, category: 1 }
);

// 2. Consulta sobre campos de subdocumentos anidados usando notación de punto (Dot Notation)
db.products.find(
  { "specs.panel": "IPS" },
  { _id: 0, name: 1, "specs.panel": 1, "specs.refresh_rate_hz": 1 }
);

// 3. Consulta sobre arreglos: productos que contengan TANTO el tag "teclado" COMO "rgb"
db.products.find({
  tags: { $all: ["teclado", "rgb"] }
});

// 4. Ordenamiento descendente (-1) por precio y límite de los 3 productos más costosos
db.products.find(
  {},
  { _id: 0, name: 1, price: 1 }
).sort({ price: -1 }).limit(3);

// 5. Conteo exacto de documentos que cumplen un filtro
db.products.countDocuments({ category: "Perifericos" });
```

### 4.3. Actualización Atómica (`updateOne`, `updateMany`) y Eliminación (`deleteOne`, `deleteMany`)

En MongoDB, los métodos `updateOne(filtro, modificador, opciones)` y `updateMany(filtro, modificador, opciones)` utilizan **operadores de actualización atómica** para mutar únicamente los campos deseados sin sobrescribir el resto del documento:

- **`$set` / `$unset`**: Asigna el valor de un campo (creándolo si no existe) o elimina la llave del documento.
- **`$inc` / `$mul`**: Incrementa (o decrementa con números negativos) o multiplica un valor numérico de forma atómica (ideal para descontar inventario o aplicar descuentos porcentuales).
- **`$push` / `$addToSet` / `$pull`**: Agrega un elemento al final de un arreglo (`$push`), lo inserta solo si no existe aún garantizando unicidad como conjunto (`$addToSet`), o elimina todas las ocurrencias que coincidan (`$pull`).
- **Opción `{ upsert: true }`**: Si ningún documento coincide con el filtro de búsqueda, inserta un nuevo documento combinando los criterios del filtro y las mutaciones.

```javascript
// 1. Actualizar el precio y descontar 2 unidades del stock de forma atómica en una sola operación
db.products.updateOne(
  { name: "Monitor UltraWide 27" },
  {
    $set: { price: 299.99 },
    $inc: { stock: -2 },
    $addToSet: { tags: "oferta" }
  }
);

// 2. Incrementar en un 5% (multiplicar por 1.05) el precio de todos los periféricos con $mul
db.products.updateMany(
  { category: "Perifericos" },
  { $mul: { price: 1.05 } }
);

// 3. Eliminar un elemento específico de un arreglo con $pull
db.products.updateOne(
  { name: "Monitor UltraWide 27" },
  { $pull: { tags: "oferta" } }
);

// 4. Eliminación controlada de documentos con deleteOne() y deleteMany()
db.products.deleteOne({ name: "Mouse Ergonomico Vertical" });

// PRECAUCIÓN: db.products.deleteMany({}) con filtro vacío elimina TODOS los documentos de la colección
db.products.deleteMany({ stock: { $lte: 0 } });
```

---

## 5. Tuberías de Agregación (*Aggregation Pipelines*) e Indexación Especializada

### 5.1. Arquitectura del *Aggregation Pipeline*

Para consultas analíticas complejas, reportes gerenciales y cruces entre colecciones, MongoDB implementa el **Aggregation Framework** a través del método `db.coleccion.aggregate([...])`. Su diseño sigue el patrón arquitectónico de **Tuberías y Filtros (*Pipes and Filters*)**: los documentos de la colección entran a una secuencia ordenada de etapas $E_1 \to E_2 \to \dots \to E_k$, donde la salida transformada de la etapa $E_i$ constituye la entrada de la etapa $E_{i+1}$.

```mermaid
flowchart LR
    C[("Colección<br/>orders")] --> S1["1. $match<br/>(Filtra pedidos<br/>status: 'completed')"]
    S1 --> S2["2. $unwind<br/>(Desglosa arreglo<br/>'$items' en N docs)"]
    S2 --> S3["3. $group<br/>(Agrupa por cliente<br/>$sum ingresos y uds)"]
    S3 --> S4["4. $lookup<br/>(Une con 'customers'<br/>por customer_id)"]
    S4 --> S5["5. $sort & $project<br/>(Ordena por total<br/>y formatea salida)"]

    style C fill:#e3f2fd,stroke:#1565c0
    style S1 fill:#fff3e0,stroke:#e65100
    style S2 fill:#fce4ec,stroke:#ad1457
    style S3 fill:#e8f5e9,stroke:#2e7d32
    style S4 fill:#f3e5f5,stroke:#6a1b9a
    style S5 fill:#e0f7fa,stroke:#00838f
```

> **Heurística de Optimización en el Pipeline**: Al igual que el empuje de selecciones ($\sigma$) hacia las hojas del árbol en el optimizador de álgebra relacional, en un *Aggregation Pipeline* la etapa **`$match` debe ubicarse siempre al principio de la tubería**. Esto permite al motor aprovechar los índices de la colección y descartar documentos irrelevantes antes de ejecutar etapas costosas en CPU y memoria como `$unwind`, `$group` o `$lookup`.

#### Caso Práctico: Desglose de Ítems (`$unwind`), Agrupación (`$group`) y Unión (`$lookup`)

Supongamos que la colección `orders` almacena pedidos completados donde cada pedido contiene un arreglo `items: [{ product_id, name, qty, price }, ...]`. Para calcular el gasto total y el número de unidades adquiridas por cada cliente junto con su información personal de la colección `customers`, construimos el siguiente pipeline:

```javascript
db.orders.aggregate([
  // Etapa 1: Filtrar tempranamente solo pedidos completados (aprovecha índices si existen)
  {
    $match: { status: "completed" }
  },
  // Etapa 2: Desestructurar el arreglo 'items'; genera 1 documento por cada ítem del pedido
  {
    $unwind: "$items"
  },
  // Etapa 3: Agrupar por customer_id y calcular métricas agregadas
  {
    $group: {
      _id: "$customer_id",
      total_gastado: {
        $sum: { $multiply: ["$items.qty", "$items.price"] }
      },
      total_unidades: { $sum: "$items.qty" },
      ticket_promedio_item: { $avg: "$items.price" }
    }
  },
  // Etapa 4: Left Outer Join con la colección 'customers' para enriquecer los datos del cliente
  {
    $lookup: {
      from: "customers",
      localField: "_id",
      foreignField: "_id",
      as: "datos_cliente"
    }
  },
  // Etapa 5: Convertir el arreglo 'datos_cliente' de 1 elemento en un subdocumento plano
  {
    $unwind: "$datos_cliente"
  },
  // Etapa 6: Proyectar el formato final del reporte y ordenar de mayor a menor gasto
  {
    $project: {
      _id: 0,
      cliente_id: "$_id",
      nombre_cliente: "$datos_cliente.name",
      ciudad: "$datos_cliente.city",
      total_gastado: { $round: ["$total_gastado", 2] },
      total_unidades: 1
    }
  },
  {
    $sort: { total_gastado: -1 }
  }
]);
```

### 5.2. Estrategias de Indexación Especializada (`createIndex`)

Sin un índice adecuado, MongoDB debe ejecutar un **`COLLSCAN` (*Collection Scan*)**, leyendo en disco cada documento de la colección con complejidad $\mathcal{O}(N)$. Al definir un índice mediante `createIndex()`, el motor construye una estructura auxiliar (con complejidad de búsqueda logarítmica $\mathcal{O}(\log N)$ en índices B-Tree) que transforma la operación en un **`IXSCAN` (*Index Scan*)**.

```javascript
// 1. Índice simple ascendente (1) sobre categoría y compuesto sobre categoría + precio descendente (-1)
db.products.createIndex({ category: 1, price: -1 });

// 2. Índice de Búsqueda de Texto (Text Index) para búsquedas léxicas de lenguaje natural
db.products.createIndex({ name: "text", "specs.panel": "text" });

// Consulta utilizando el índice de texto recién creado
db.products.find({ $text: { $search: "monitor ips" } });

// 3. Índice Geoespacial Esférico (2dsphere) sobre coordenadas GeoJSON [longitud, latitud]
db.stores.createIndex({ location: "2dsphere" });

// Consulta de tiendas físicas en un radio máximo de 2,000 metros (2 km) alrededor de un punto
db.stores.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-75.5636, 6.2518] },
      $maxDistance: 2000
    }
  }
});
```

#### Índices Vectoriales (`$vectorSearch`) y Arquitecturas RAG para Inteligencia Artificial

Uno de los casos de uso contemporáneos más relevantes de MongoDB es su papel como **base de datos operacional y vectorial unificada** en arquitecturas **RAG (*Retrieval-Augmented Generation*)** para Grandes Modelos de Lenguaje (LLMs).

En lugar de mantener dos infraestructuras separadas (una base de datos para los documentos del negocio y otra base vectorial externa para los *embeddings*), cada documento en MongoDB puede almacenar tanto el texto original y sus metadatos como el arreglo numérico de punto flotante (`embedding: [0.014, -0.082, ..., 0.049]`) generado por un modelo de *embeddings*. Mediante un índice vectorial (basado en el algoritmo **HNSW — *Hierarchical Navigable Small World***), la etapa `$vectorSearch` del pipeline calcula la similitud del coseno o producto punto entre el vector de la pregunta del usuario $\vec{q}$ y los vectores almacenados $\vec{v}$:

$$\text{sim}_{\cos}(\vec{q}, \vec{v}) = \frac{\vec{q} \cdot \vec{v}}{\|\vec{q}\| \, \|\vec{v}\|}$$

recuperando en milisegundos los $k$ fragmentos semánticamente más relevantes junto con sus metadatos filtrados para inyectarlos en el contexto del LLM.

---

## 6. Reflexión: Poliglotismo de Persistencia y Convergencia Arquitectónica del Curso

Al culminar el recorrido técnico del curso, resulta evidente que la ingeniería de datos moderna no consiste en elegir dogmáticamente una única tecnología, sino en comprender cómo interactúan las distintas capas físicas, lógicas y distribuidas bajo el paradigma de **Persistencia Políglota (*Polyglot Persistence*)**:

1. **Capa de Almacenamiento Físico y Resiliencia en Disco**: Ya sea que utilicemos PostgreSQL, Apache Spark o el motor WiredTiger de MongoDB, todo dato persiste finalmente en bloques físicos de tamaño $B$ con un factor de bloqueo $\text{bfr} = \lfloor B / R \rfloor$ sobre arreglos de discos protegidos por esquemas como **RAID 5** (paridad simple distribuida con $\text{XOR}$ que tolera la caída de $1$ disco) o **RAID 6** (paridad dual independiente $P$ y $Q$ que permite reconstruir secuencialmente hasta $2$ discos caídos mediante álgebra de campos finitos y operaciones $\text{XOR}$).
2. **Capa de Optimización Lógica y Algebraica**: La regla de oro del rendimiento se mantiene invariante en todos los paradigmas vistos en el semestre: **filtrar y proyectar lo más temprano posible**. Empujar las selecciones ($\sigma$) y proyecciones ($\pi$) hacia las hojas del árbol en el optimizador relacional es conceptualmente idéntico a filtrar y usar `reduceByKey` antes del *Shuffle* en **PySpark MapReduce**, y equivale exactamente a colocar `$match` y `$project` al inicio de un **Aggregation Pipeline en MongoDB**.
3. **Coexistencia Relacional, Analítica y Documental**: En una arquitectura empresarial real, el módulo de facturación y contabilidad reside en un **SGBD Relacional (SQL)** normalizado en 3FN/BCNF con transacciones ACID estrictas; el catálogo polimórfico de productos, las sesiones de usuario y los *embeddings* para asistentes de IA operan sobre **MongoDB**; y el procesamiento analítico masivo histórico (*Batch / Lakehouse*) cruza ambas fuentes a escala de terabytes mediante **Apache Spark DataFrames y Spark SQL**. Dominar los fundamentos matemáticos y algorítmicos detrás de cada motor es lo que faculta al ingeniero para diseñar sistemas escalables, resilientes y eficientes.

---

## 7. Referencias y Enlaces de Interés

- **Documentación Oficial de MongoDB — Especificación BSON y Tipos de Datos**: [https://www.mongodb.com/docs/manual/reference/bson-types/](https://www.mongodb.com/docs/manual/reference/bson-types/)
- **Documentación Oficial de MongoDB — Patrones de Modelado de Datos (Embedding vs. Referencing)**: [https://www.mongodb.com/docs/manual/core/data-modeling-introduction/](https://www.mongodb.com/docs/manual/core/data-modeling-introduction/)
- **Documentación Oficial de MongoDB — Aggregation Pipeline y Operadores de Etapas**: [https://www.mongodb.com/docs/manual/core/aggregation-pipeline/](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/)
- **Documentación Oficial de MongoDB Atlas — Búsqueda Vectorial (`$vectorSearch`) para RAG**: [https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/)
- **Silberschatz, A., Korth, H. F., & Sudarshan, S. (2019)**: *Database System Concepts* (7th Edition). McGraw-Hill Education. [https://www.db-book.com/](https://www.db-book.com/)

---

*Publicado por [Mateo Vargas Tirado](https://github.com/araigumadev) como parte del registro de aprendizaje en el curso Bases de Datos 2026/2 - Universidad de Antioquia.*
