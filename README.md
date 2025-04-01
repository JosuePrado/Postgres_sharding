# Postgres Sharding

Esta guía te muestra cómo configurar un clúster Citus simple usando Docker Compose, crear una tabla, distribuirla (sharding) y analizar cómo se ejecutan las consultas.

**Prerrequisitos:**

*   Docker instalado.
*   Docker Compose instalado.
*   El archivo `docker-compose.yml` del repositorio

## 1. Iniciar los Servicios Citus

Abre una terminal en el directorio donde guardaste el archivo `docker-compose.yml` y ejecuta:

```bash
docker-compose up
```
Ahora en otra terminar verificamos que los contenedores se hayan creado
```bash
docker ps
```
## 2. Conectarse al Coordinador

```bash
docker exec -it postgres_sharding-coordinator-1 psql -U postgres
```

## 3. Verificar los Nodos Worker

Dentro de `psql`, ejecutamos el siguiente comando para asegur de que el coordinador ve a los nodos worker que se añadieron automáticamente por el script de inicio:

```sql
SELECT * FROM citus_get_active_worker_nodes();
```

## 4. Crear la Tabla

Creamos la tabla que vamos a distribuir. Nota la clave primaria compuesta, es necesaria para las tablas distribuidas en Citus:

```sql
CREATE TABLE activity_log (
    log_id BIGSERIAL,
    user_id INT NOT NULL,
    action_description VARCHAR(255),
    log_timestamp TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, log_id)
);
```
## 5. Distribuir la Tabla (Sharding)

Ahora, le decimos a Citus que distribuya la tabla `activity_log` basándose en los valores de la columna `user_id`:

```sql
SELECT create_distributed_table('activity_log', 'user_id');
```

## 6. Insertar Datos de Ejemplo

Insertamos algunas `filas en la tabla distribuida. Nos conectamos al coordinador, pero Citus enrutará cada fila al worker apropiado según el `user_id`:

```sql
INSERT INTO activity_log (user_id, action_description) VALUES
(1, 'Logged in'),
(2, 'Viewed dashboard'),
(1, 'Updated profile'),
(3, 'Posted a comment'),
(2, 'Logged out');
```

## 7. Analizar el Plan de Ejecución de Consultas

Usamos `EXPLAIN ANALYZE` para ver cómo Citus planifica y ejecuta consultas que filtran por la columna de distribución (`user_id`). Esto nos muestra la eficiencia del sharding y con que worker esta trabajando.

Primero, analizamos una consulta para `user_id = 1`:

```sql
EXPLAIN ANALYZE SELECT * FROM activity_log WHERE user_id = 1;
```

Ahora, hacemos lo mismo para `user_id = 2`:

```sql
EXPLAIN ANALYZE SELECT * FROM activity_log WHERE user_id = 2;
```