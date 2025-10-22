# Estructura de Base de Datos por Dominios FlightHub

## Índice

1. [Visión General](#visión-general)
2. [Principios de Diseño](#principios-de-diseño)
3. [Flight Orchestrator Domain](#flight-orchestrator-domain)
4. [Resources Domain](#resources-domain)
5. [Timeline Domain](#timeline-domain)
6. [Delays Domain](#delays-domain)
7. [Crew Domain](#crew-domain)
8. [Alerts Domain](#alerts-domain)
9. [Passengers Domain](#passengers-domain)
10. [Baggage Domain](#baggage-domain)
11. [Fuel Domain](#fuel-domain)
12. [Aircraft Domain](#aircraft-domain)
13. [Schedules Domain](#schedules-domain)
14. [Onward Flights Domain](#onward-flights-domain)
15. [Codeshare Domain](#codeshare-domain)
16. [Resumen de Dominios](#resumen-de-dominios)
17. [Estrategia de Migración](#estrategia-de-migración)

---

## Visión General

Este documento describe la arquitectura de base de datos de FlightHub basada en **Domain-Driven Design (DDD)**, donde cada dominio de negocio tiene sus propias tablas con responsabilidades claramente definidas.

### Principios Fundamentales

1. **Prefijo por Dominio**: Todas las tablas llevan prefijo `fh_` seguido del nombre del dominio
2. **Identificador Único**: Cada vuelo tiene un `fuid` único tipo ULID (26 caracteres)
3. **Campos de Identificación**: Los 6 campos clave se replican en cada tabla de dominio
4. **Sin JOINs**: Cada dominio es independiente y puede consultarse sin JOINs
5. **Auditoría**: Todos los campos llevan created_at, created_by, updated_at, updated_by

### Los 6 Campos de Identificación

Estos campos aparecen en todas las tablas de todos los dominios:

```sql
operation_date          DATE NOT NULL           -- Día de Operación: 2025-07-01
flight_designator       VARCHAR(10) NOT NULL    -- Número de vuelo: "999"
operational_suffix      VARCHAR(3) NOT NULL     -- Sufijo operacional: "A", "B", ""
airline_designator      VARCHAR(3) NOT NULL     -- Código IATA: "IB"
departure_airport       VARCHAR(3) NOT NULL     -- Aeropuerto salida: "MAD"
departure_number        INTEGER NOT NULL        -- Número de intento: 1
```

### Dominios del Sistema

```
FlightHub Database Architecture
│
├── 🎯 Flight Orchestrator (Orquestación central)
├── 📍 Resources (Recursos aeroportuarios)
├── ⏰ Timeline (Tiempos operacionales)
├── ⚠️ Delays (Retrasos)
├── 👨‍✈️ Crew (Tripulación)
├── 🚨 Alerts (Alarmas y desvíos)
├── 👥 Passengers (Pasajeros)
├── 🎒 Baggage (Equipaje y carga)
├── ⛽ Fuel (Combustible)
├── ✈️ Aircraft (Aeronaves)
├── 📅 Schedules (Programación)
├── 🔄 Onward Flights (Vuelos de continuación)
└── 🤝 Codeshare (Vuelos compartidos)
```

---

## Principios de Diseño

### 1. Autonomía de Dominios

Cada dominio:
- ✅ Tiene sus propias tablas
- ✅ Puede ser consultado independientemente
- ✅ Puede escalar de forma independiente
- ✅ Puede desplegarse independientemente
- ✅ Tiene ownership claro

### 2. Identificación Única

**fuid**: `VARCHAR(26)` - ULID (Universally Unique Lexicographically Sortable Identifier)

Ejemplo: `"01HQZ8X9Y1K2M3N4P5Q6R7S8T9"`

Los 6 campos de identificación externa:
- operation_date: 2025-07-01
- flight_designator: "999"
- operational_suffix: "A" (o "B", "")
- airline_designator: "IB"
- departure_airport: "MAD"
- departure_number: 1

### 3. Campos Comunes en Todas las Tablas

```sql
-- Identificación del vuelo (PK o FK)
fuid                    VARCHAR(26) NOT NULL

-- 6 Campos de Identificación (replicados)
operation_date          DATE NOT NULL
flight_designator       VARCHAR(10) NOT NULL
operational_suffix      VARCHAR(3) NOT NULL DEFAULT ''
airline_designator      VARCHAR(3) NOT NULL
departure_airport       VARCHAR(3) NOT NULL
departure_number        INTEGER NOT NULL DEFAULT 1

-- Auditoría
created_at              TIMESTAMP NOT NULL
created_by              VARCHAR NOT NULL
updated_at              TIMESTAMP NOT NULL
updated_by              VARCHAR NOT NULL
```

---

## Flight Orchestrator Domain

**Responsabilidad**: Identificación única de vuelos, gestión del ciclo de vida, trazabilidad de mensajes.

**Prefijo**: `fh_flight`

### Tabla: `fh_flight`

**Descripción**: Tabla principal de vuelos con información de identificación única.

```sql
CREATE TABLE fh_flight (
  -- Identificador único permanente (ULID)
  fuid                          VARCHAR(26) PRIMARY KEY,

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Códigos ICAO adicionales
  airline_designator_icao       VARCHAR(4) NOT NULL,
  flight_designator_atc         VARCHAR(10) NOT NULL,
  departure_airport_icao        VARCHAR(4) NOT NULL,
  departure_airport_orig        VARCHAR(3) NOT NULL,
  departure_airport_orig_icao   VARCHAR(4) NOT NULL,
  arrival_airport               VARCHAR(3) NOT NULL,
  arrival_airport_icao          VARCHAR(4) NOT NULL,
  arrival_airport_orig          VARCHAR(3) NOT NULL,
  arrival_airport_orig_icao     VARCHAR(4) NOT NULL,

  -- Control de estado
  active                        BOOLEAN NOT NULL DEFAULT true,
  principal                     BOOLEAN NOT NULL DEFAULT true,
  fuid_new_flight               VARCHAR(26) NULL,
  fuid_flight_principal         VARCHAR(26) NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_operation_date (operation_date),
  INDEX idx_airline (airline_designator),
  INDEX idx_flight_designator (flight_designator),
  INDEX idx_departure (departure_airport),
  INDEX idx_active (active),
  INDEX idx_principal (principal)
);
```

**Campos clave**:
- `fuid`: Identificador único permanente (ULID 26 caracteres)
- `active`: Indica si el vuelo está activo (no ha sido modificado por otro)
- `principal`: Indica si es vuelo principal (no es secundario de otro)
- `fuid_new_flight`: Referencia al nuevo vuelo que desactiva este registro
- `fuid_flight_principal`: Referencia al vuelo principal si este es secundario

**Fuente**: [dominios.md](dominios.md) - `fh_flight`

---

### Tabla: `flight_record`

**Descripción**: Registros de procesamiento de vuelos - control de procesamiento de mensajes.

```sql
CREATE TABLE flight_record (
  -- Identificador
  id                            BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,

  -- Identificación del mensaje
  arinc633message_id            VARCHAR(255) NULL,
  flight_identifier             VARCHAR(255) NULL,

  -- Control de procesamiento
  attempts                      INTEGER NOT NULL,
  status                        INTEGER NOT NULL,
  execution_date                DATE NULL,
  comments                      VARCHAR(255) NULL,

  -- Índices
  INDEX idx_flight_identifier (flight_identifier),
  INDEX idx_status (status),
  INDEX idx_execution_date (execution_date)
);
```

**Fuente**: [dominios.md](dominios.md) - `flight_record`

---

### Tabla: `fh_message_log`

**Descripción**: Log completo de todos los mensajes recibidos y procesados.

```sql
CREATE TABLE fh_message_log (
  -- Identificador
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información del mensaje
  source                        VARCHAR(50) NOT NULL,     -- 'TELEX', 'AENA', 'CKI'
  message_type                  VARCHAR(50) NOT NULL,     -- 'MVT', 'ASM', 'CDM'
  message_subtype               VARCHAR(50) NULL,         -- 'NEW', 'AA', 'AD'

  -- Contenido
  raw_message                   JSONB NOT NULL,           -- Mensaje original
  parsed_message                JSONB NULL,               -- Mensaje parseado

  -- Estado de procesamiento
  processing_status             VARCHAR(20) NOT NULL,     -- PENDING, PROCESSED, FAILED
  processed_at                  TIMESTAMP NULL,
  error_message                 TEXT NULL,

  -- Auditoría
  received_at                   TIMESTAMP NOT NULL DEFAULT NOW(),

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_source_type (source, message_type),
  INDEX idx_status (processing_status),
  INDEX idx_received (received_at)
);
```

---

## Resources Domain

**Responsabilidad**: Gestión de recursos aeroportuarios (terminales, puertas, stands, pistas, cintas, mostradores).

**Prefijo**: `fh_resource_`

### Tabla: `fh_resource_airport`

**Descripción**: Recursos aeroportuarios asignados al vuelo (consolidación de terminales, puertas, stands, pistas, cintas).

```sql
CREATE TABLE fh_resource_airport (
  -- Identificador único
  fuid                          VARCHAR(26) PRIMARY KEY REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Recursos de Salida
  departure_terminal_zone       VARCHAR NULL,
  departure_terminal            VARCHAR NULL,
  departure_stand               VARCHAR NULL,
  departure_runway              VARCHAR NULL,
  departure_checkin_counter_first  VARCHAR NULL,
  departure_checkin_counter_last   VARCHAR NULL,
  departure_checkin_counter_type   VARCHAR NULL,
  departure_boarding_zone       VARCHAR NULL,
  departure_boarding_gate       VARCHAR NULL,
  departure_boarding_gate_2     VARCHAR NULL,
  departure_bag_belt            VARCHAR NULL,
  departure_bag_belt_status     VARCHAR NULL,

  -- Recursos de Llegada
  arrival_terminal_zone         VARCHAR NULL,
  arrival_terminal              VARCHAR NULL,
  arrival_stand                 VARCHAR NULL,
  arrival_runway                VARCHAR NULL,
  arrival_hall                  VARCHAR NULL,
  arrival_gate                  VARCHAR NULL,
  arrival_bag_belt              VARCHAR NULL,
  arrival_bag_belt_2            VARCHAR NULL,
  arrival_bag_belt_status       VARCHAR NULL,
  arrival_bag_claim_unit_code   VARCHAR NULL,
  arrival_bag_claim_unit_status VARCHAR NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_departure_gate (departure_boarding_gate),
  INDEX idx_departure_stand (departure_stand),
  INDEX idx_arrival_gate (arrival_gate),
  INDEX idx_arrival_stand (arrival_stand)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_airport_resource`

**Nota**: Esta tabla consolida todos los recursos aeroportuarios. Si se requiere separar por tipo de recurso en el futuro, se pueden crear tablas individuales:
- `fh_resource_gate` (puertas)
- `fh_resource_stand` (stands)
- `fh_resource_runway` (pistas)
- `fh_resource_belt` (cintas)
- `fh_resource_counter` (mostradores)

---

## Timeline Domain

**Responsabilidad**: Todos los tiempos operacionales del vuelo (salida, llegada, CDM, embarque, puertas).

**Prefijo**: `fh_timeline_`

### Tabla: `fh_timeline_departure`

**Descripción**: Tiempos de salida del vuelo.

```sql
CREATE TABLE fh_timeline_departure (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Variación horaria
  departure_time_variation        VARCHAR NOT NULL,
  departure_time_variation_orig   VARCHAR NOT NULL,

  -- Tiempos programados
  departure_time_scheduled        TIMESTAMP NOT NULL,

  -- Tiempos estimados
  departure_time_estimated        TIMESTAMP NOT NULL,
  off_blocks_time_estimated       TIMESTAMP NOT NULL,
  taxi_out_time_estimated         NUMERIC NULL,
  takeoff_time_estimated          TIMESTAMP NOT NULL,

  -- Tiempos reales
  departure_time_actual           TIMESTAMP NULL,
  off_blocks_time_actual          TIMESTAMP NULL,
  taxi_out_time_actual            NUMERIC NULL,
  takeoff_time_actual             TIMESTAMP NULL,

  -- Tiempos de puertas y cabina
  cabin_door_close_estimated      TIMESTAMP NOT NULL,
  cabin_door_close_actual         TIMESTAMP NULL,
  cargo_door_close_actual         TIMESTAMP NULL,

  -- Tiempos de embarque
  departure_boarding_gate_start_time    TIMESTAMP NULL,
  departure_boarding_gate_end_time      TIMESTAMP NULL,
  departure_boarding_gate2_start_time   TIMESTAMP NULL,
  departure_boarding_gate2_end_time     TIMESTAMP NULL,

  -- Tiempos de equipaje
  departure_bag_belt_start_time   TIMESTAMP NULL,
  departure_bag_belt_end_time     TIMESTAMP NULL,

  -- Tiempos de preparación
  ready_cabin_time_estimated      TIMESTAMP NULL,
  ready_cabin_time_actual         TIMESTAMP NULL,
  ready_maintenance_time_actual   TIMESTAMP NULL,
  ready_parking_time_actual       TIMESTAMP NULL,
  next_info_departure_time_actual TIMESTAMP NULL,

  -- Block time
  block_time_min_estimated        NUMERIC NOT NULL,
  block_time_min_actual           NUMERIC NULL,
  block_time_min_estimated_orig   NUMERIC NOT NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  UNIQUE(fuid),
  INDEX idx_scheduled (departure_time_scheduled),
  INDEX idx_actual (departure_time_actual)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_timeline` (campos departure_*)

---

### Tabla: `fh_timeline_arrival`

**Descripción**: Tiempos de llegada del vuelo.

```sql
CREATE TABLE fh_timeline_arrival (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Variación horaria
  arrival_time_variation          VARCHAR NOT NULL,
  arrival_time_variation_orig     VARCHAR NOT NULL,

  -- Tiempos programados
  arrival_time_scheduled          TIMESTAMP NOT NULL,

  -- Tiempos estimados
  arrival_time_estimated          TIMESTAMP NOT NULL,
  landing_time_estimated          TIMESTAMP NOT NULL,
  on_blocks_time_estimated        TIMESTAMP NOT NULL,
  taxi_in_time_estimated          NUMERIC NULL,

  -- Tiempos reales
  arrival_time_actual             TIMESTAMP NULL,
  landing_time_actual             TIMESTAMP NULL,
  on_blocks_time_actual           TIMESTAMP NULL,
  taxi_in_time_actual             NUMERIC NULL,

  -- Tiempos de puertas
  cabin_door_open_estimated       TIMESTAMP NOT NULL,
  cabin_door_open_actual          TIMESTAMP NULL,
  cargo_door_open_actual          TIMESTAMP NULL,

  -- Tiempos de cintas equipaje
  arrival_bag_belt_start_time     TIMESTAMP NULL,
  arrival_bag_belt_end_time       TIMESTAMP NULL,
  arrival_bag_belt2_start_time    TIMESTAMP NULL,
  arrival_bag_belt2_end_time      TIMESTAMP NULL,
  arrival_bag_claim_unit_start_time  TIMESTAMP NULL,
  arrival_bag_claim_unit_end_time    TIMESTAMP NULL,

  -- Trip time
  trip_time_min_estimated         NUMERIC NOT NULL,
  trip_time_min_actual            NUMERIC NULL,
  trip_time_min_estimated_orig    NUMERIC NOT NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  UNIQUE(fuid),
  INDEX idx_scheduled (arrival_time_scheduled),
  INDEX idx_actual (arrival_time_actual)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_timeline` (campos arrival_*)

---

### Tabla: `fh_timeline_cdm`

**Descripción**: Tiempos CDM (Collaborative Decision Making) específicos.

```sql
CREATE TABLE fh_timeline_cdm (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Off-Blocks Times (CDM)
  off_blocks_time_scheduled_cdm    TIMESTAMP NULL,
  off_blocks_time_estimated_cdm    TIMESTAMP NULL,
  off_blocks_time_target_cdm       TIMESTAMP NULL,
  off_blocks_time_actual_cdm       TIMESTAMP NULL,

  -- Taxi Times (CDM)
  taxi_out_time_estimated_cdm      NUMERIC NULL,
  taxi_out_time_actual_cdm         NUMERIC NULL,
  taxi_in_time_estimated_cdm       NUMERIC NULL,
  taxi_in_time_actual_cdm          NUMERIC NULL,

  -- Take-Off Times (CDM)
  takeoff_time_estimated_cdm       TIMESTAMP NULL,
  takeoff_time_calculated_cdm      TIMESTAMP NULL,
  takeoff_time_target_cdm          TIMESTAMP NULL,
  takeoff_time_actual_cdm          TIMESTAMP NULL,

  -- Landing Times (CDM)
  landing_time_estimated_cdm       TIMESTAMP NULL,
  landing_time_target_cdm          TIMESTAMP NULL,
  landing_time_actual_cdm          TIMESTAMP NULL,

  -- On-Blocks Times (CDM)
  on_blocks_time_scheduled_cdm     TIMESTAMP NULL,
  on_blocks_time_estimated_cdm     TIMESTAMP NULL,
  on_blocks_time_actual_cdm        TIMESTAMP NULL,

  -- Turnaround Times (CDM)
  turnaround_time_scheduled_cdm    TIMESTAMP NULL,
  turnaround_time_estimated_cdm    TIMESTAMP NULL,
  turnaround_time_minimum_cdm      TIMESTAMP NULL,
  turnaround_time_actual_cdm       TIMESTAMP NULL,

  -- Ground Handling Times (CDM)
  ground_handling_time_actual_cdm  TIMESTAMP NULL,
  ground_handling_start_time_cdm   TIMESTAMP NULL,
  ground_handling_end_time_cdm     TIMESTAMP NULL,

  -- Startup Times (CDM)
  startup_request_time_actual_cdm  TIMESTAMP NULL,
  startup_approval_time_target_cdm TIMESTAMP NULL,
  startup_approval_time_actual_cdm TIMESTAMP NULL,

  -- De-icing Times (CDM)
  deicing_time_estimated_cdm       TIMESTAMP NULL,
  deicing_time_actual_cdm          TIMESTAMP NULL,
  deicing_ready_time_estimated_cdm TIMESTAMP NULL,
  deicing_ready_time_actual_cdm    TIMESTAMP NULL,
  deicing_start_time_estimated_cdm TIMESTAMP NULL,
  deicing_start_time_actual_cdm    TIMESTAMP NULL,
  deicing_end_time_estimated_cdm   TIMESTAMP NULL,
  deicing_end_time_actual_cdm      TIMESTAMP NULL,

  -- Ready Departure Time (CDM)
  ready_departure_time_actual_cdm  TIMESTAMP NULL,

  -- Boarding Start Time (CDM)
  start_boarding_time_actual_cdm   TIMESTAMP NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  UNIQUE(fuid),
  INDEX idx_tobt (off_blocks_time_target_cdm),
  INDEX idx_tsat (startup_approval_time_target_cdm)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_timeline` (campos *_cdm)

---

### Tabla: `fh_timeline_checkin`

**Descripción**: Tiempos de check-in (CI, CL, CC) y sus variantes WAB.

```sql
CREATE TABLE fh_timeline_checkin (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Check-in CI (inicial)
  checkin_ci_time_actual        TIMESTAMP NULL,

  -- Check-in CL (intermedio)
  checkin_cl_time_actual        TIMESTAMP NULL,

  -- Check-in CC (final)
  checkin_cc_time_actual        TIMESTAMP NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  UNIQUE(fuid)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_timeline` (campos checkin*timeactual)

---

## Delays Domain

**Responsabilidad**: Gestión de retrasos del vuelo con sus códigos y tiempos.

**Prefijo**: `fh_delay_`

### Tabla: `fh_delay`

**Descripción**: Información de retrasos del vuelo (hasta 4 retrasos diferentes).

```sql
CREATE TABLE fh_delay (
  -- Identificador único
  fuid                          VARCHAR(26) PRIMARY KEY REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Retraso 1
  delay_type_1                  VARCHAR NOT NULL,     -- SCD, DEP
  delay_number_1                VARCHAR NOT NULL,
  delay_code_1                  VARCHAR NOT NULL,
  delay_group_code_1            VARCHAR NOT NULL,
  delay_mins_1                  NUMERIC NOT NULL,
  delay_comments_1              VARCHAR NOT NULL,

  -- Retraso 2
  delay_type_2                  VARCHAR NULL,
  delay_number_2                VARCHAR NULL,
  delay_code_2                  VARCHAR NULL,
  delay_group_code_2            VARCHAR NULL,
  delay_mins_2                  NUMERIC NULL,
  delay_comments_2              VARCHAR NULL,

  -- Retraso 3
  delay_type_3                  VARCHAR NULL,
  delay_number_3                VARCHAR NULL,
  delay_code_3                  VARCHAR NULL,
  delay_group_code_3            VARCHAR NULL,
  delay_mins_3                  NUMERIC NULL,
  delay_comments_3              VARCHAR NULL,

  -- Retraso 4
  delay_type_4                  VARCHAR NULL,
  delay_number_4                VARCHAR NULL,
  delay_code_4                  VARCHAR NULL,
  delay_group_code_4            VARCHAR NULL,
  delay_mins_4                  NUMERIC NULL,
  delay_comments_4              VARCHAR NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_delay_code_1 (delay_code_1),
  INDEX idx_delay_mins (delay_mins_1, delay_mins_2, delay_mins_3, delay_mins_4)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_delay`

**Complementar con old.md**: La tabla `flight_delays` de old.md tiene estructura similar pero normalizada. Si se requiere normalización futura, crear:

### Tabla alternativa: `fh_delay_detail` (normalizada)

```sql
CREATE TABLE fh_delay_detail (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información del retraso
  delay_sequence                INTEGER NOT NULL,         -- 1, 2, 3, 4
  delay_type                    VARCHAR NOT NULL,         -- SCD, DEP
  delay_number                  VARCHAR NOT NULL,
  delay_code                    VARCHAR NOT NULL,
  delay_group_code              VARCHAR NOT NULL,
  delay_minutes                 NUMERIC NOT NULL,
  delay_comments                VARCHAR NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_delay_code (delay_code),
  INDEX idx_sequence (delay_sequence)
);
```

**Fuente**: [old.md](old.md) - `flight_delays` (normalizado)

---

## Crew Domain

**Responsabilidad**: Información de tripulación del vuelo.

**Prefijo**: `fh_crew_`

### Tabla: `fh_crew_assignment`

**Descripción**: Asignación de tripulación técnica y de cabina.

```sql
CREATE TABLE fh_crew_assignment (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información de tripulación técnica (Cockpit)
  cockpit_crew_count            INTEGER NULL,
  cockpit_employer              VARCHAR NULL,

  -- Información de tripulación de cabina
  cabin_crew_count              INTEGER NULL,
  cabin_employer                VARCHAR NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_info` (campos cockpitemployer, cabinemployer)

**Nota**: Esta tabla es básica. Si se requiere información detallada de tripulación de [old.md](old.md) `flight_departure_info` (capitán, primer oficial, etc.), se puede extender la tabla con:

```sql
-- Campos adicionales para detalles de tripulación
captain_id                    VARCHAR NULL,
captain_name                  VARCHAR NULL,
first_officer_id              VARCHAR NULL,
first_officer_name            VARCHAR NULL,
-- ... más campos según necesidad
```

---

## Alerts Domain

**Responsabilidad**: Alertas, alarmas y situaciones especiales (desvíos, retornos).

**Prefijo**: `fh_alert_`

### Tabla: `fh_alert_alarm`

**Descripción**: Alarmas operacionales del vuelo.

```sql
CREATE TABLE fh_alert_alarm (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información de la alarma
  alarm_code                    VARCHAR(20) NULL,
  alarm_text                    TEXT NULL,
  alarm_severity                VARCHAR(20) NULL,        -- INFO, WARNING, CRITICAL

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_alarm_code (alarm_code),
  INDEX idx_severity (alarm_severity)
);
```

**Fuente**: [old.md](old.md) - `flight_departure_info` (campos alarmCode, alarmText)

---

### Tabla: `fh_alert_diversion`

**Descripción**: Información de desvíos, retornos o vuelos frustrados.

```sql
CREATE TABLE fh_alert_diversion (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información del desvío
  diversion_type                VARCHAR(20) NULL,        -- DIVERSION, RETURN, ABORTED
  diversion_airport             VARCHAR(3) NULL,
  diversion_airport_icao        VARCHAR(4) NULL,
  diversion_code                VARCHAR(20) NULL,
  diversion_reason              TEXT NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_diversion_airport (diversion_airport)
);
```

**Fuente**: [old.md](old.md) - `flight_arrival_info` (campos diversionAirport, diversionCode)

---

## Passengers Domain

**Responsabilidad**: Todo lo relacionado con pasajeros, capacidad, reservas, facturación, embarque.

**Prefijo**: `fh_pax_`

### Tabla: `fh_pax_summary`

**Descripción**: Resumen general de pasajeros (totales y desglose por tipo).

```sql
CREATE TABLE fh_pax_summary (
  -- Identificador único
  fuid                          VARCHAR(26) PRIMARY KEY REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Totales generales
  config_total                  NUMERIC NOT NULL DEFAULT 0,
  capacity_total                NUMERIC NOT NULL DEFAULT 0,
  availability_total            NUMERIC NOT NULL DEFAULT 0,
  load_factor_total             NUMERIC NOT NULL DEFAULT 0,
  booked_pax_total              NUMERIC NOT NULL DEFAULT 0,
  checked_pax_total             NUMERIC NOT NULL DEFAULT 0,
  boarded_pax_total             NUMERIC NOT NULL DEFAULT 0,
  forecast_pax_total            NUMERIC NOT NULL DEFAULT 0,

  -- Por conexión
  inbound_booked_pax_total      NUMERIC NOT NULL DEFAULT 0,
  outbound_booked_pax_total     NUMERIC NOT NULL DEFAULT 0,

  -- Por tipo de pasajero
  adult_pax_total               NUMERIC NOT NULL DEFAULT 0,
  male_pax_total                NUMERIC NOT NULL DEFAULT 0,
  female_pax_total              NUMERIC NOT NULL DEFAULT 0,
  no_gender_pax_total           NUMERIC NOT NULL DEFAULT 0,
  child_pax_total               NUMERIC NOT NULL DEFAULT 0,
  infant_pax_total              NUMERIC NOT NULL DEFAULT 0,

  -- Pasajeros especiales
  pad_pax_total                 NUMERIC NOT NULL DEFAULT 0,
  dhc_pax_total                 NUMERIC NOT NULL DEFAULT 0,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_booked (booked_pax_total),
  INDEX idx_checked (checked_pax_total),
  INDEX idx_boarded (boarded_pax_total)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_pax` (campos totales)

---

### Tabla: `fh_pax_cabin`

**Descripción**: Información de pasajeros desglosada por cabina (Business, Premium, Turista).

```sql
CREATE TABLE fh_pax_cabin (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Identificación de cabina
  cabin_code                    VARCHAR(10) NOT NULL,    -- JC, W, Y
  cabin_name                    VARCHAR(50) NULL,        -- Business, Premium, Turista

  -- Configuración y capacidad
  config_cabin                  NUMERIC NOT NULL DEFAULT 0,
  capacity_cabin                NUMERIC NOT NULL DEFAULT 0,
  availability_cabin            NUMERIC NOT NULL DEFAULT 0,
  load_factor_cabin             NUMERIC NOT NULL DEFAULT 0,

  -- Contadores de pasajeros
  booked_pax_cabin              NUMERIC NOT NULL DEFAULT 0,
  checked_pax_cabin             NUMERIC NOT NULL DEFAULT 0,
  boarded_pax_cabin             NUMERIC NOT NULL DEFAULT 0,
  forecast_pax_cabin            NUMERIC NOT NULL DEFAULT 0,

  -- Por conexión
  inbound_booked_pax_cabin      NUMERIC NOT NULL DEFAULT 0,
  outbound_booked_pax_cabin     NUMERIC NOT NULL DEFAULT 0,

  -- Check-in por evento (CI, CL, CC)
  checked_in_pax_total_ci       NUMERIC NOT NULL DEFAULT 0,
  checked_in_pax_total_cl       NUMERIC NOT NULL DEFAULT 0,
  checked_in_pax_total_cc       NUMERIC NOT NULL DEFAULT 0,
  checked_in_infants_total_ci   NUMERIC NOT NULL DEFAULT 0,
  checked_in_infants_total_cl   NUMERIC NOT NULL DEFAULT 0,
  checked_in_infants_total_cc   NUMERIC NOT NULL DEFAULT 0,
  checked_in_pax_cabin_ci       NUMERIC NOT NULL DEFAULT 0,
  checked_in_pax_cabin_cl       NUMERIC NOT NULL DEFAULT 0,
  checked_in_pax_cabin_cc       NUMERIC NOT NULL DEFAULT 0,

  -- Por tipo de pasajero
  adult_pax_cabin               NUMERIC NOT NULL DEFAULT 0,
  male_pax_cabin                NUMERIC NOT NULL DEFAULT 0,
  female_pax_cabin              NUMERIC NOT NULL DEFAULT 0,
  no_gender_pax_cabin           NUMERIC NOT NULL DEFAULT 0,
  child_pax_cabin               NUMERIC NOT NULL DEFAULT 0,
  infant_pax_cabin              NUMERIC NOT NULL DEFAULT 0,
  pad_pax_cabin                 NUMERIC NOT NULL DEFAULT 0,
  dhc_pax_cabin                 NUMERIC NOT NULL DEFAULT 0,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  UNIQUE(fuid, cabin_code),
  INDEX idx_cabin_code (cabin_code)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_pax` (campos por cabina: *cabinjc, *cabinw, *cabiny)

---

### Tabla: `fh_pax_special`

**Descripción**: Pasajeros con necesidades especiales o categorías particulares.

```sql
CREATE TABLE fh_pax_special (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Categoría del pasajero especial
  category_code                 VARCHAR(10) NOT NULL,    -- WCHC, WCHR, WCHS, UM, etc.
  category_name                 VARCHAR(100) NULL,
  cabin_class                   VARCHAR(20) NULL,        -- cabin1, cabin2, cabin3
  quantity                      INTEGER NOT NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_category (category_code)
);
```

**Fuente**: [old.md](old.md) - `flight_passengers`

---

## Baggage Domain

**Responsabilidad**: Equipaje, carga y correo.

**Prefijo**: `fh_bag_`

### Tabla: `fh_bag_summary`

**Descripción**: Resumen de equipaje del vuelo.

```sql
CREATE TABLE fh_bag_summary (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información de equipaje (de LDM u otras fuentes)
  total_bags                    INTEGER NULL,
  total_bag_weight_kg           INTEGER NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid)
);
```

**Fuente**: [old.md](old.md) - `flight_arrival_info` (bagsLDM, bagWeightLDM)

---

### Tabla: `fh_bag_cargo`

**Descripción**: Resumen de carga y correo del vuelo.

```sql
CREATE TABLE fh_bag_cargo (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Pesos de carga (kg)
  cargo_weight_kg               INTEGER NULL,
  additional_cargo_weight_kg    INTEGER NULL,
  total_cargo_weight_kg         INTEGER NULL,

  -- Pesos de correo (kg)
  mail_weight_kg                INTEGER NULL,
  additional_mail_weight_kg     INTEGER NULL,
  total_mail_weight_kg          INTEGER NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid)
);
```

**Fuente**: [old.md](old.md) - `flight_info` (cargoWeight, mailWeight, additionalCargoWeight, additionalMailWeight, totalCargoWeight, totalMailWeight)

---

### Tabla: `fh_bag_cargo_item`

**Descripción**: Ítems individuales de carga especial (AVI, DGR, HUM, etc.).

```sql
CREATE TABLE fh_bag_cargo_item (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información del ítem
  item_name                     VARCHAR(100) NOT NULL,   -- AVI, DGR, HUM, etc.
  item_value                    NUMERIC NULL,
  item_unit                     VARCHAR(20) NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_item_name (item_name)
);
```

**Fuente**: [old.md](old.md) - `flight_cargos`

---

## Fuel Domain

**Responsabilidad**: Combustible, repostaje, planificación de fuel.

**Prefijo**: `fh_fuel_`

### Tabla: `fh_fuel_summary`

**Descripción**: Resumen de combustible del vuelo.

```sql
CREATE TABLE fh_fuel_summary (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Pesos de combustible (kg)
  total_fuel_weight_kg          INTEGER NULL,
  taxi_fuel_weight_kg           INTEGER NULL,
  trip_fuel_weight_kg           INTEGER NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid)
);
```

**Fuente**: [old.md](old.md) - `flight_fuels`

---

### Tabla: `fh_fuel_accept_aircraft`

**Descripción**: Combustible al aceptar aeronave.

```sql
CREATE TABLE fh_fuel_accept_aircraft (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Combustibles (kg)
  arrival_fuel_kg               INTEGER NULL,
  remaining_fuel_kg             INTEGER NULL,
  fuel_tipping_kg               INTEGER NULL,
  depart_fuel_kg                INTEGER NULL,
  calculated_planned_uplift_kg  INTEGER NULL,
  required_block_fuel_kg        INTEGER NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid)
);
```

**Fuente**: [old.md](old.md) - `flight_final_fuel_accept_aircraft`

---

### Tabla: `fh_fuel_close_flight`

**Descripción**: Combustible al cerrar vuelo.

```sql
CREATE TABLE fh_fuel_close_flight (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Combustibles (kg)
  arrival_fuel_kg               INTEGER NULL,
  remaining_fuel_kg             INTEGER NULL,
  fuel_tipping_kg               INTEGER NULL,
  depart_fuel_kg                INTEGER NULL,
  calculated_planned_uplift_kg  INTEGER NULL,
  required_block_fuel_kg        INTEGER NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid)
);
```

**Fuente**: [old.md](old.md) - `flight_final_fuel_close_flight`

---

### Tabla: `fh_fuel_event`

**Descripción**: Eventos de repostaje.

```sql
CREATE TABLE fh_fuel_event (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Tipo de evento
  event_type                    VARCHAR(50) NOT NULL,    -- ACCEPT_AIRCRAFT, CLOSE_FLIGHT

  -- Información del repostaje
  measurement_system            VARCHAR(20) NULL,        -- metric, imperial, us
  supplier                      VARCHAR(100) NULL,
  vendor                        VARCHAR(100) NULL,
  invoice_number                VARCHAR(50) NULL,

  -- Tipo y densidad del combustible
  fuel_type                     VARCHAR(20) NULL,        -- JET A1
  density                       NUMERIC(10,3) NULL,
  density_unit                  VARCHAR(20) NULL,

  -- Cantidad repostada
  actual_uplift                 NUMERIC(10,2) NULL,
  actual_uplift_unit            VARCHAR(20) NULL,

  -- Cambios y razones
  new_fuel_kg                   INTEGER NULL,
  reason                        TEXT NULL,
  reduce_note                   TEXT NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_event_type (event_type)
);
```

**Fuente**: [old.md](old.md) - `flight_final_fuel_fueling_event_accept_aircraft`, `flight_final_fuel_fueling_event_close_flight`

---

## Aircraft Domain

**Responsabilidad**: Información de aeronaves, configuraciones, asignaciones.

**Prefijo**: `fh_aircraft_`

### Tabla: `fh_aircraft_info`

**Descripción**: Información de la aeronave asignada al vuelo.

```sql
CREATE TABLE fh_aircraft_info (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información del propietario y operador
  aircraft_owner                VARCHAR NOT NULL,
  cockpit_employer              VARCHAR NOT NULL,
  cabin_employer                VARCHAR NOT NULL,

  -- Tipo y configuración
  aircraft_type                 VARCHAR NOT NULL,
  aircraft_subtype              VARCHAR NOT NULL,
  aircraft_config               VARCHAR NOT NULL,
  aircraft_version              VARCHAR NULL,
  aircraft_registration         VARCHAR NULL,
  aircraft_registration_first   VARCHAR NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_registration (aircraft_registration),
  INDEX idx_type (aircraft_type)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_info` (campos aircraft*)

---

### Tabla: `fh_aircraft_registry`

**Descripción**: Registro maestro de aeronaves (catálogo).

```sql
CREATE TABLE fh_aircraft_registry (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  tail_number                   VARCHAR(10) UNIQUE NOT NULL,

  -- Tipo de aeronave
  aircraft_type                 VARCHAR(10) NOT NULL,
  aircraft_subtype              VARCHAR(20) NULL,
  manufacturer                  VARCHAR(50) NULL,
  serial_number                 VARCHAR(50) NULL,

  -- Propietario
  airline_owner                 VARCHAR(3) NULL,
  registration_country          VARCHAR(2) NULL,

  -- Estado operacional
  operational_status            VARCHAR(20) NULL,
  in_service_date               DATE NULL,
  out_of_service_date           DATE NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at                    TIMESTAMP NOT NULL DEFAULT NOW(),

  -- Índices
  INDEX idx_aircraft_type (aircraft_type),
  INDEX idx_airline_owner (airline_owner)
);
```

**Fuente**: Datos maestros (no en entidades actuales)

---

## Schedules Domain

**Responsabilidad**: Programación de vuelos, información SSIM.

**Prefijo**: `fh_schedule_`

### Tabla: `fh_schedule_info`

**Descripción**: Información de programación y servicio del vuelo.

```sql
CREATE TABLE fh_schedule_info (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Tipo de servicio
  service_type_code             VARCHAR NOT NULL,
  service_type_desc             VARCHAR NOT NULL,

  -- Estado del vuelo
  status_code                   VARCHAR NOT NULL DEFAULT 'PDEP',
  status_desc                   VARCHAR NOT NULL DEFAULT 'Predeparture',

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_status (status_code)
);
```

**Fuente**: [dominios.md](dominios.md) - `fh_flight_info` (servicetypecode, servicetypedesc, statuscode, statusdesc)

---

## Onward Flights Domain

**Responsabilidad**: Relación entre un vuelo de llegada (inbound) y sus vuelos de continuación (onward), ya sea para pasajeros en conexión o simplemente para mapear continuidad operacional.

**Prefijo**: `fh_onward_`

### Tabla: `fh_onward_flight`

**Descripción**: Información del vuelo siguiente conectado (onward flight).

```sql
CREATE TABLE fh_onward_flight (
  -- Identificador único
  id                            UUID PRIMARY KEY,

  -- Vuelo de llegada (inbound)
  inbound_fuid                  VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación del vuelo inbound
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Vuelo siguiente / de continuación (onward)
  onward_airline_designator     VARCHAR(3) NOT NULL,
  onward_flight_designator      VARCHAR(10) NOT NULL,
  onward_operation_date         DATE NOT NULL,
  onward_operation_day          VARCHAR(2),

  -- Metadatos y clasificaciones
  connection_type               VARCHAR(20),              -- direct, interline, codeshare, etc.
  min_connection_time_minutes   INTEGER,                  -- MCT opcional, para planificación
  turnaround_time_minutes       INTEGER,                  -- opcional si se usa también para tiempos entre vuelos

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_inbound (inbound_fuid),
  INDEX idx_onward (onward_airline_designator, onward_flight_designator, onward_operation_date)
);
```

**Notas**:
- **Nombre**: `fh_onward_flight` comunica claramente que son vuelos de continuación, no operaciones de turnaround
- **Relación**: Cada registro vincula un vuelo entrante (inbound_fuid) con un vuelo "siguiente" definido por su código y fecha de operación
- **Extensible**: Se pueden añadir tablas complementarias como `fh_onward_connection_log`, `fh_mct_rules` o `fh_onward_status` para gestión avanzada
- **Índices**: Optimizados para buscar rápido por vuelo inbound o por designador + fecha del onward

**Fuente**: [dominios.md](dominios.md) - `fh_flight_info` (onwardflightdate, onwardairlinedesignator, onwardflightdesignator)

---

## Codeshare Domain

**Responsabilidad**: Información de vuelos compartidos.

**Prefijo**: `fh_codeshare_`

### Tabla: `fh_codeshare_info`

**Descripción**: Información de vuelos codeshare (compartidos).

```sql
CREATE TABLE fh_codeshare_info (
  -- Identificador único
  id                            UUID PRIMARY KEY,
  fuid                          VARCHAR(26) NOT NULL REFERENCES fh_flight(fuid),

  -- 6 Campos de Identificación
  operation_date                DATE NOT NULL,
  flight_designator             VARCHAR(10) NOT NULL,
  operational_suffix            VARCHAR(3) NOT NULL DEFAULT '',
  airline_designator            VARCHAR(3) NOT NULL,
  departure_airport             VARCHAR(3) NOT NULL,
  departure_number              INTEGER NOT NULL DEFAULT 1,

  -- Información de codeshare
  codeshare_indicator           VARCHAR(1) NULL,         -- P (Principal), S (Secondary)
  codeshare_principal_flight_id VARCHAR NULL,
  codeshare_principal_flight    VARCHAR NULL,
  codeshare_secondary_flights   VARCHAR NULL,

  -- Auditoría
  created_at                    TIMESTAMP NOT NULL,
  created_by                    VARCHAR NOT NULL,
  updated_at                    TIMESTAMP NOT NULL,
  updated_by                    VARCHAR NOT NULL,

  -- Índices
  INDEX idx_fuid (fuid),
  INDEX idx_indicator (codeshare_indicator)
);
```

**Fuente**: [old.md](old.md) - `flight_info` (codeshareIndicator, codesharePrincipalFlightId, etc.)

---

## Event Publisher

### Responsabilidad

El **Event Publisher** es un componente crítico que:

1. **Recibe eventos de todos los dominios** (con FUID + 6 campos)
2. **Elimina el FUID** antes de publicar externamente
3. **Publica SOLO los 6 campos** a EventBridge
4. **Adapta el payload** según el tipo de consumidor

### Principio Fundamental: FUID es INTERNO

```
┌─────────────────────────────────────────────────────────────┐
│  IMPORTANTE: El FUID NO se publica a sistemas externos      │
│                                                              │
│  ✅ Uso INTERNO:  FUID + 6 campos (dominios, orchestrator)  │
│  ✅ Uso EXTERNO:  SOLO 6 campos (EventBridge, consumers)    │
└─────────────────────────────────────────────────────────────┘
```

### Campos de Identificación Externa

Los 6 campos que se publican a sistemas externos:

```typescript
interface ExternalFlightIdentifier {
  operation_date: Date;           // Fecha de operación UTC
  flight_designator: string;      // "347"
  operational_suffix: string;     // "A", "B", ""
  airline_designator: string;     // "IB"
  departure_airport: string;      // "MAD"
  departure_number: number;       // 1, 2, 3... (turnarounds)
}
```

### Flujo de Publicación

```typescript
// 1. Evento INTERNO de dominio (incluye FUID + 6 campos)
const internalEvent = {
  fuid: "01HQZ8X9Y1K2M3N4P5Q6R7S8T9",  // ← FUID interno
  operation_date: "2025-01-14",
  flight_designator: "347",
  operational_suffix: "",
  airline_designator: "IB",
  departure_airport: "MAD",
  departure_number: 1,
  domain: "passengers",
  type: "passengers.checkin.updated",
  data: {
    total_passengers: 180,
    checked_in_passengers: 150,
    boarded_passengers: 0,
  },
  source: "CKI",
  timestamp: "2025-01-14T08:30:00Z",
};

// 2. Event Publisher recibe el evento
// Los 6 campos YA VIENEN en el evento, NO necesita consultar fh_flight

// 3. Event Publisher ELIMINA el FUID y construye payload externo
const externalEvent = {
  flightIdentifier: {
    operation_date: "2025-01-14",
    flight_designator: "347",
    operational_suffix: "",
    airline_designator: "IB",
    departure_airport: "MAD",
    departure_number: 1,
  },
  eventType: "passengers.checkin.updated",
  timestamp: "2025-01-14T08:30:00Z",
  source: "CKI",
  passengers: {
    total: 180,
    checkedIn: 150,
    boarded: 0,
  },
  // ← NOTA: El FUID NO está presente
};

// 4. Publica a EventBridge
await eventBridge.putEvents({
  Entries: [{
    Source: "com.iberia.flighthub",
    DetailType: "passengers.checkin.updated",
    Detail: JSON.stringify(externalEvent),
    EventBusName: "flight-events-bus",
  }],
});
```

### Implementación del Event Publisher

```typescript
class EventPublisher {
  async publishDomainEvent(domainEvent: DomainEvent) {
    // 1. Los 6 campos ya vienen en el evento de dominio
    // NO necesita consultar fh_flight

    // 2. Construir identificador externo (SIN FUID)
    const externalId: ExternalFlightIdentifier = {
      operation_date: domainEvent.operation_date,
      flight_designator: domainEvent.flight_designator,
      operational_suffix: domainEvent.operational_suffix || "",
      airline_designator: domainEvent.airline_designator,
      departure_airport: domainEvent.departure_airport,
      departure_number: domainEvent.departure_number,
    };

    // 3. Construir payload según el tipo de cambio
    const payload = this.buildPayload(domainEvent, externalId);

    // 4. Publicar a EventBridge
    await eventBridge.putEvents({
      Entries: [{
        Source: "com.iberia.flighthub",
        DetailType: domainEvent.type,
        Detail: JSON.stringify(payload),
        EventBusName: "flight-events-bus",
        Resources: [
          `flight:${externalId.airline_designator}:${externalId.flight_designator}`,
          `airport:${externalId.departure_airport}`,
          `date:${externalId.operation_date}`,
        ],
      }],
    });
  }

  private buildPayload(
    event: DomainEvent,
    externalId: ExternalFlightIdentifier
  ) {
    // Payload base con identificadores externos (SIN FUID)
    const basePayload = {
      flightIdentifier: externalId,  // ← Solo 6 campos
      timestamp: event.timestamp,
      source: event.source,
      eventType: event.type,
    };

    // Agregar datos específicos del dominio
    switch (event.domain) {
      case "passengers":
        return {
          ...basePayload,
          passengers: {
            total: event.data.total_passengers,
            checkedIn: event.data.checked_in_passengers,
            boarded: event.data.boarded_passengers,
          },
        };

      case "fuel":
        return {
          ...basePayload,
          fuel: {
            uplift: event.data.fuel_uplift,
            planned: event.data.fuel_planned,
            remaining: event.data.fuel_remaining,
          },
        };

      case "timeline":
        return {
          ...basePayload,
          times: {
            scheduledDeparture: event.data.departure_time_scheduled,
            estimatedDeparture: event.data.departure_time_estimated,
            actualDeparture: event.data.departure_time_actual,
          },
        };

      // ... otros dominios
    }
  }
}
```

### Beneficios del Modelo Dual

1. ✅ **FUID interno**: Simplicidad, inmutabilidad, performance (solo uso interno)
2. ✅ **6 campos externos**: Compatibilidad con estándares aeronáuticos
3. ✅ **Trazabilidad**: `departure_number` mantiene relación entre intentos de despegue
4. ✅ **No consultas adicionales**: Los 6 campos ya vienen en cada evento de dominio
5. ✅ **Separación de concerns**: Dominios trabajan con FUID + 6 campos, externos solo 6 campos
6. ✅ **Sin FUID en EventBridge**: Los sistemas externos no necesitan conocer el FUID interno

### Manejo de Turnarounds

Cuando un vuelo despega múltiples veces (return, diversion):

```typescript
// Primer despegue
{
  fuid: "01HQZ8X9Y1K2M3N4P5Q6R7S8T9",
  operation_date: "2025-01-14",
  flight_designator: "347",
  airline_designator: "IB",
  departure_airport: "MAD",
  departure_number: 1,  // Primer intento
}

// Segundo despegue (return to base)
{
  fuid: "01HQZ8X9Y1K2M3N4P5Q6R7S8T9",  // Mismo FUID
  operation_date: "2025-01-14",
  flight_designator: "347",
  airline_designator: "IB",
  departure_airport: "MAD",
  departure_number: 2,  // Segundo intento
}

// Evento externo publicado (incluye departure_number)
{
  flightIdentifier: {
    operation_date: "2025-01-14",
    flight_designator: "347",
    operational_suffix: "",
    airline_designator: "IB",
    departure_airport: "MAD",
    departure_number: 2,  // ← Identifica el intento
  },
  eventType: "flight.departed",
  // ... datos del vuelo
}
```

---

## Resumen de Dominios

### Tabla Resumen de Dominios y Tablas

| Dominio | Prefijo | Tablas | Responsabilidad Principal |
|---------|---------|--------|---------------------------|
| **🎯 Flight Orchestrator** | `fh_flight` | 3 | Identificación única, control de ciclo de vida, mensajes |
| **📍 Resources** | `fh_resource_` | 1 | Recursos aeroportuarios (gates, stands, runways, belts) |
| **⏰ Timeline** | `fh_timeline_` | 4 | Tiempos operacionales (departure, arrival, CDM, checkin) |
| **⚠️ Delays** | `fh_delay_` | 1-2 | Retrasos y códigos de delay |
| **👨‍✈️ Crew** | `fh_crew_` | 1 | Tripulación técnica y de cabina |
| **🚨 Alerts** | `fh_alert_` | 2 | Alarmas y desvíos |
| **👥 Passengers** | `fh_pax_` | 3 | Pasajeros, capacidad, cabinas |
| **🎒 Baggage** | `fh_bag_` | 3 | Equipaje, carga, correo |
| **⛽ Fuel** | `fh_fuel_` | 4 | Combustible y repostaje |
| **✈️ Aircraft** | `fh_aircraft_` | 2 | Aeronaves y configuraciones |
| **📅 Schedules** | `fh_schedule_` | 1 | Programación y servicio |
| **🔄 Onward Flights** | `fh_onward_` | 1 | Vuelos de continuación |
| **🤝 Codeshare** | `fh_codeshare_` | 1 | Vuelos compartidos |

### Conteo Total de Tablas

```
🎯 Flight Orchestrator:  3 tablas
📍 Resources:            1 tabla
⏰ Timeline:             4 tablas
⚠️ Delays:               1 tabla (+ 1 opcional normalizada)
👨‍✈️ Crew:                1 tabla
🚨 Alerts:               2 tablas
👥 Passengers:           3 tablas
🎒 Baggage:              3 tablas
⛽ Fuel:                 4 tablas
✈️ Aircraft:             2 tablas
📅 Schedules:            1 tabla
🔄 Onward Flights:       1 tabla
🤝 Codeshare:            1 tabla
─────────────────────────────────
TOTAL:                  27 tablas
```

### Comparación: Antes vs Después

**ANTES (Sistema Legacy):**
- ❌ 20 tablas `flight_*` mezclando responsabilidades
- ❌ `flight_departure_info` con 100+ campos mezclados
- ❌ Queries complejas con múltiples JOINs
- ❌ Difícil de escalar y mantener

**DESPUÉS (Arquitectura por Dominios):**
- ✅ 27 tablas organizadas en 13 dominios
- ✅ Cada tabla tiene responsabilidad clara
- ✅ Queries sin JOINs (6 campos replicados)
- ✅ Escalabilidad independiente por dominio
- ✅ Ownership claro
- ✅ Prefijos consistentes (`fh_*`)

---

## Estrategia de Migración

### Fase 1: Preparación

1. **Crear nuevas tablas** con prefijo `fh_*` en paralelo a las existentes
2. **Mapear campos** de tablas legacy a nuevos dominios
3. **Implementar lógica de dual-write** en aplicación

### Fase 2: Dual Write

```typescript
// Escribir en ambas estructuras
async function updateFlightTimeline(fuid: string, data: TimelineData) {
  // 1. Escribir en estructura legacy
  await legacyRepo.updateFlightDepartureInfo(flightId, {
    actualTakeOffTime: data.takeoff_time_actual,
    estimatedDepartureTime: data.departure_time_estimated,
    // ...
  });

  // 2. Escribir en nueva estructura
  await timelineRepo.updateDepartureTimeline(fuid, {
    takeoff_time_actual: data.takeoff_time_actual,
    departure_time_estimated: data.departure_time_estimated,
    // ...
  });
}
```

### Fase 3: Migración de Datos Históricos

1. **Script de migración** para copiar datos existentes a nuevas tablas
2. **Validación** de integridad de datos
3. **Verificación** de consistencia entre ambas estructuras

### Fase 4: Cambio de Lectura

```typescript
// Leer de nueva estructura, seguir escribiendo en ambas
async function getFlightTimeline(fuid: string): Promise<TimelineData> {
  // Leer de nueva estructura
  return await timelineRepo.getDepartureTimeline(fuid);
}
```

### Fase 5: Solo Nueva Estructura

```typescript
// Solo usar nueva estructura
async function updateFlightTimeline(fuid: string, data: TimelineData) {
  return await timelineRepo.updateDepartureTimeline(fuid, data);
}
```

### Fase 6: Deprecación Legacy

1. **Remover código de dual-write**
2. **Archivar tablas legacy**
3. **Documentar cambios**

---

## Beneficios de la Nueva Arquitectura

### 1. Separación Clara de Responsabilidades

```
❌ ANTES:
flight_departure_info (100+ campos mezclados)
├─ Times (20+ campos)
├─ Gates/Stands (10+ campos)
├─ Passengers (30+ campos)
├─ Check-in (10+ campos)
├─ Crew (15+ campos)
└─ Baggage (10+ campos)

✅ DESPUÉS:
⏰ Timeline Domain
├─ fh_timeline_departure (31 campos)
├─ fh_timeline_arrival (28 campos)
├─ fh_timeline_cdm (45 campos)
└─ fh_timeline_checkin (3 campos)

📍 Resources Domain
└─ fh_resource_airport (25 campos)

👥 Passengers Domain
├─ fh_pax_summary (18 campos)
└─ fh_pax_cabin (40 campos)

👨‍✈️ Crew Domain
└─ fh_crew_assignment (6 campos)

🎒 Baggage Domain
└─ fh_bag_summary (3 campos)
```

### 2. Queries Optimizadas Sin JOINs

```sql
-- ❌ ANTES: Múltiples JOINs
SELECT
  f.*,
  fi.*,
  fdi.*,
  fai.*
FROM flights f
LEFT JOIN flight_info fi ON f.flight_info_id = fi.id
LEFT JOIN flight_departure_info fdi ON f.flight_departure_info_id = fdi.id
LEFT JOIN flight_arrival_info fai ON f.flight_arrival_info_id = fai.id
WHERE f.flight_id = '20250701_IB_999_b_MAD_1';

-- ✅ DESPUÉS: Query directa sin JOINs
SELECT * FROM fh_timeline_departure
WHERE fuid = '01HQZ8X9Y1K2M3N4P5Q6R7S8T9';

-- O bien usando los 6 campos de identificación:
SELECT * FROM fh_timeline_departure
WHERE operation_date = '2025-07-01'
  AND airline_designator = 'IB'
  AND flight_designator = '999'
  AND operational_suffix = 'B'
  AND departure_airport = 'MAD'
  AND departure_number = 1;
```

### 3. Escalabilidad Independiente

- ✅ Timeline Domain puede tener más réplicas durante horas pico
- ✅ Passengers Domain escala independientemente durante check-in
- ✅ Fuel Domain escala según necesidad de repostaje

### 4. Deploys Independientes

- ✅ Cambios en Fuel Domain no afectan Timeline Domain
- ✅ Menos riesgo en despliegues
- ✅ Rollbacks por dominio

### 5. Ownership Claro

- ✅ Cada equipo es dueño de su dominio
- ✅ Responsabilidades claras
- ✅ Autonomía de equipos

---

## Conclusión

Esta arquitectura de base de datos por dominios transforma el sistema monolítico legacy en una arquitectura moderna, escalable y mantenible basada en Domain-Driven Design (DDD).

### Logros Principales

1. ✅ **13 dominios independientes** con responsabilidades claras
2. ✅ **27 tablas organizadas** con prefijos consistentes `fh_*`
3. ✅ **Sin JOINs necesarios** gracias a los 6 campos replicados
4. ✅ **Escalabilidad por dominio** según carga específica
5. ✅ **Separación de concerns** clara y lógica
6. ✅ **Facilita microservicios** futuros por dominio
7. ✅ **Ownership claro** para equipos

### Próximos Pasos

1. Revisar y validar estructura de dominios
2. Implementar scripts de migración
3. Establecer estrategia de dual-write
4. Definir ownership por dominio
5. Crear documentación técnica detallada
6. Planificar rollout gradual

---

**Documento generado**: 2025-01-XX
**Versión**: 2.0
**Basado en**: [dominios.md](dominios.md) + [old.md](old.md)
