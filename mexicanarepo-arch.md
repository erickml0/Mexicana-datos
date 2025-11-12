# Arquitectura de Mexicana Repo

## Diagrama de componentes y conexiones

```mermaid
graph TB
    subgraph "APACHE AIRFLOW - Orquestador"
        WEB[Airflow Webserver<br/>Puerto 8080<br/>UI Web]
        SCHED[Airflow Scheduler<br/>Ejecuta DAGs]
        WORKER[Airflow Worker<br/>Celery Executor]
        TRIGGER[Airflow Triggerer<br/>Async Tasks]
    end

    subgraph "BASES DE DATOS"
        PG[(PostgreSQL<br/>Puerto 5432<br/>Metadata Airflow)]
        REDIS[(Redis<br/>Puerto 6379<br/>Message Broker)]
        MONGO[(MongoDB<br/>Puerto 27017<br/>Cache Items)]
    end

    subgraph "INTERFACES DE GESTIÓN"
        MONGOEX[Mongo Express<br/>Puerto 8081<br/>UI MongoDB]
        FLOWER[Flower<br/>Puerto 5555<br/>Monitor Celery]
    end

    subgraph "PIPELINE DE DATOS"
        SOURCE[TainacanSourceOperator<br/>Extracción]
        TRANSFORM[AggregationTransformPipe<br/>Transformación]
        LOAD[LoadAgregationItems<br/>Carga Local]
        DEST[TainacanDestinationOperator<br/>Envío a Destino]
    end

    subgraph "APIs EXTERNAS"
        TAINAPI[Tainacan API Destino<br/>WordPress REST API<br/>wp-json/tainacan/v2]
        SOURCES[Fuentes Tainacan<br/>APIs Remotas<br/>Múltiples Orígenes]
    end

    %% Conexiones Airflow Interno
    WEB -->|Lee/Escribe Estado| PG
    SCHED -->|Lee DAGs| PG
    SCHED -->|Publica Tareas| REDIS
    WORKER -->|Consume Tareas| REDIS
    WORKER -->|Actualiza Estado| PG
    TRIGGER -->|Estado Async| PG

    %% Conexiones Pipeline
    WORKER -->|Ejecuta| SOURCE
    WORKER -->|Ejecuta| TRANSFORM
    WORKER -->|Ejecuta| LOAD
    WORKER -->|Ejecuta| DEST

    %% Flujo de Datos
    SOURCE -->|1. GET Items| SOURCES
    SOURCE -->|2. INSERT Cache| MONGO
    
    LOAD -->|Lee Config| MONGO
    LOAD -->|Crea Collections| MONGO
    
    TRANSFORM -->|3. READ Cache| MONGO
    TRANSFORM -->|4. UPDATE Transformado| MONGO
    
    DEST -->|5. READ Final| MONGO
    DEST -->|6. POST/PATCH Items| TAINAPI
    DEST -->|7. Upload Media| TAINAPI
    DEST -->|8. Gestiona Relaciones| TAINAPI

    %% Interfaces de Gestión
    MONGOEX -->|Visualiza| MONGO
    FLOWER -.->|Monitorea| WORKER
    FLOWER -.->|Conecta| REDIS

    %% Estilos
    classDef airflowClass fill:#4CAF50,stroke:#2E7D32,color:#fff
    classDef dbClass fill:#2196F3,stroke:#1565C0,color:#fff
    classDef pipeClass fill:#FF9800,stroke:#E65100,color:#fff
    classDef apiClass fill:#9C27B0,stroke:#6A1B9A,color:#fff
    classDef uiClass fill:#00BCD4,stroke:#00838F,color:#fff

    class WEB,SCHED,WORKER,TRIGGER airflowClass
    class PG,REDIS,MONGO dbClass
    class SOURCE,TRANSFORM,LOAD,DEST pipeClass
    class TAINAPI,SOURCES apiClass
    class MONGOEX,FLOWER uiClass
```

## Flujo de datos detallado

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'noteBkgColor':'#fff', 'noteTextColor':'#000'}}}%%
sequenceDiagram
    participant Config as Config YAML
    participant Scheduler as Airflow Scheduler
    participant Worker as Airflow Worker
    participant Mongo as MongoDB Cache
    participant Source as API Fuente
    participant Dest as Tainacan Destino

    Config->>Scheduler: 1. Lee configuración DAG
    Scheduler->>Worker: 2. Asigna tarea via Redis
    
    rect rgb(200, 220, 240)
        Note over Worker,Source: FASE 1: EXTRACCIÓN
        Worker->>Source: 3. GET /items?paged=1
        Source-->>Worker: Items JSON
        Worker->>Worker: fn_rename() + fn_add_fields()
        Worker->>Mongo: 4. INSERT items transformados
    end
    
    rect rgb(240, 220, 200)
        Note over Worker,Mongo: FASE 2: AGREGACIÓN
        Worker->>Mongo: 5. READ items por fuente
        Worker->>Worker: Aplica mapeo metadatos
        Worker->>Mongo: 6. UPDATE items agregados
    end
    
    rect rgb(220, 240, 200)
        Note over Worker,Dest: FASE 3: CARGA
        Worker->>Mongo: 7. READ items finales
        loop Por cada item
            Worker->>Dest: 8. POST /collection/items
            Dest-->>Worker: item_id
            Worker->>Dest: 9. PATCH /item/{id}/metadata/{meta_id}
            
            alt Metadato tipo Relationship
                Worker->>Dest: 10a. GET buscar item relacionado
                alt No existe
                    Worker->>Dest: 10b. POST crear item en colección relacionada
                end
            end
            
            alt Tiene thumbnail
                Worker->>Dest: 11. POST /wp/v2/media (upload)
                Dest-->>Worker: attachment_id
                Worker->>Dest: 12. PATCH /items/{id} (_thumbnail_id)
            end
            
            Worker->>Dest: 13. PATCH /items/{id} (status=publish)
        end
    end
```

## Componentes y dependencias

### **Airflow Webserver**
- **Puerto**: 8080
- **Función**: Interfaz web para gestionar DAGs
- **Conexiones**:
  - PostgreSQL: Metadata de usuarios, DAGs, ejecuciones
  - Redis: Indirecto (vía workers)

### **Airflow Scheduler**
- **Función**: Orquestador principal de tareas
- **Conexiones**:
  - PostgreSQL: Lee/escribe estado de DAGs
  - Redis: Publica tareas en cola Celery

### **Airflow Worker (Celery)**
- **Función**: Ejecutor de tareas del pipeline
- **Conexiones**:
  - Redis: Consume tareas de la cola
  - PostgreSQL: Actualiza resultados
  - MongoDB: Cache de datos
  - APIs Tainacan: HTTP REST

### **PostgreSQL**
- **Puerto**: 5432
- **Datos**: Metadata Airflow, estado de ejecuciones, logs

### **Redis**
- **Puerto**: 6379
- **Función**: Message broker para Celery
- **Tipo**: In-memory queue

### **MongoDB**
- **Puerto**: 27017
- **Función**: Cache temporal durante agregación
- **Estructura**:
  - Base de datos por configuración (ej: `mexicana_mongo_cache_db`)
  - Colección por fuente de datos (ej: `idsource_fuente1`)

### **Mongo Express**
- **Puerto**: 8081
- **Función**: UI web para inspeccionar MongoDB
- **Credenciales**: `tainacan`/`tainacan`

### **Flower (Opcional)**
- **Puerto**: 5555
- **Función**: Monitor web de Celery workers
- **Activación**: `docker-compose --profile flower up`

## Operators del Pipeline

```mermaid
graph LR
    A[LoadAgregationItems] -->|Inicializa| B[TainacanSource]
    B -->|Extrae| C[AggregationTransform]
    C -->|Transforma| D[TainacanDestination]
    
    subgraph "Hooks Utilizados"
        H1[TainacanGetItemsHook<br/>Paginación API]
        H2[TainacanAPIHook<br/>CRUD Items]
        H3[MongoHook<br/>Cache Local]
    end
    
    B -.usa.- H1
    B -.usa.- H3
    D -.usa.- H2
    D -.usa.- H3
```

### **1. LoadAgregationItemsDataOperator**
- Lee configuración de agregación
- Crea estructuras en MongoDB
- Prepara colecciones de cache

### **2. TainacanSourceOperator**
- Conecta con API fuente via `TainacanGetItemsHook`
- Descarga items paginados
- Aplica transformaciones:
  - `fn_rename()`: Mapea campos
  - `fn_add_fields()`: Añade metadata fija
- Guarda en MongoDB cache

### **3. AggregationTransformPipeOperator**
- Lee items de MongoDB
- Aplica mapeo de metadatos según configuración
- Normaliza valores
- Actualiza cache con datos transformados

### **4. TainacanDestinationOperator**
- Lee items finales de MongoDB
- Usa `TainacanAPIHook` para:
  - Crear items en colección destino
  - Actualizar metadatos individuales
  - Gestionar relaciones (buscar/crear items relacionados)
  - Subir y asignar thumbnails
  - Publicar items

## Configuración y Dependencias

```mermaid
graph TD
    A[Variable: url_setup] -->|GET| B[Setup YAML]
    B --> C[configs array]
    C -->|Por cada config| D[name]
    C -->|Por cada config| E[config file URL]
    C -->|Por cada config| F[inputs array]
    
    E -->|GET| G[Config YAML<br/>Mapeo metadatos]
    F -->|GET| H[Input YAML<br/>Fuentes datos]
    
    G --> I[create_dag function]
    H --> I
    D --> I
    
    I --> J[DAG generado dinámicamente]
```

## Detalles de componentes

### **Redis - Message Broker**

Redis actúa como intermediario de mensajería entre el Scheduler y los Workers en la arquitectura Celery:

**Flujo de trabajo:**

1. **Airflow Scheduler** detecta que es momento de ejecutar una tarea del DAG
2. **Scheduler** serializa la tarea y la publica en una **cola de Redis** (queue)
3. **Redis** mantiene la tarea en memoria hasta que un Worker la consuma
4. **Airflow Worker** (consumidor Celery) escucha constantemente las colas de Redis
5. **Worker** recibe la tarea, la ejecuta, y reporta el resultado al **PostgreSQL**

**¿Por qué Redis?**
- **Velocidad**: In-memory, ideal para colas de alta velocidad
- **Confiabilidad**: Persistencia opcional para no perder tareas
- **Compatibilidad**: Broker estándar de Celery

**Colas en el sistema:**
```
redis://redis:6379/0
└── default queue
    ├── tainacan-agregador-mexicana.get_items
    ├── tainacan-agregador-mexicana.aggregation_transform_pipe
    └── tainacan-agregador-mexicana.insert_data_on_agregation
```
