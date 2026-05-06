

## 1. Definición del Negocio

**Título:** Sistema de Gestión de Lubricación Industrial (SGLI)

**Problema que resuelve:**
Las plantas industriales de producción de alimentos (como plantas de pastas) deben mantener un programa riguroso de lubricación de maquinaria para garantizar la continuidad operacional y cumplir con normativas de inocuidad alimentaria. Actualmente este proceso es deficiente y no se consigue generar dashboard ni reportes acordes a la realidad, la carga de dato es tediosa para el tecnico y se generan reportes de los cuales el cleinte no entiende la gran cantidad de informacion. El SGLI digitaliza y centraliza la gestión completa del programa de lubricación: planificación, ejecución, stock de insumos, registro de observaciones técnicas y generación de reportes, todo accesible desde una interfaz web, en vez de plataformas separadas. (AUNQUE ESTOY DESARROLLANDO EN PARALELO ESTE PROYECTO EN PYTHON)

---

## 2. Arquitectura de Microservicios

El sistema se compone de **10 microservicios independientes**, cada uno con su propia base de datos (MariaDB):

| # | Microservicio | Responsabilidad |
|---|---|---|
| 1 | `ms-actividades` | Gestión del catálogo de actividades de lubricación (matriz principal) |
| 2 | `ms-equipos` | Gestión de equipos, líneas de producción y plantas |
| 3 | `ms-lubricantes` | Catálogo de lubricantes y marcas |
| 4 | `ms-programacion` | Planificación y programación de actividades por intervalo |
| 5 | `ms-ejecucion` | Registro de ejecuciones realizadas con fecha, responsable y estado |
| 6 | `ms-stock` | Control de inventario de lubricantes en bodega |
| 7 | `ms-observaciones` | Registro de observaciones técnicas e imágenes por actividad |
| 8 | `ms-usuarios` | Autenticación, roles y permisos (técnico, supervisor, admin) |
| 9 | `ms-notificaciones` | Alertas de actividades próximas o vencidas |
| 10 | `ms-reportes` | Generación de reportes estadísticos y dashboard de KPIs |

---

## 3. Desacoplamiento y Comunicación

### Comunicación entre servicios
Todos los microservicios se comunican mediante **API REST** usando JSON sobre HTTP. Cada servicio expone endpoints documentados con los que otros servicios pueden interactuar de forma independiente.

Ejemplo de flujo:
```
ms-programacion → consulta ms-actividades para obtener intervalo
ms-ejecucion    → notifica a ms-notificaciones al registrar una ejecución
ms-stock        → actualiza consumo cuando ms-ejecucion registra una actividad
ms-reportes     → consulta ms-ejecucion y ms-actividades para generar KPIs
```

### Principio Database per Service
Cada microservicio posee su **propia base de datos MariaDB independiente**, garantizando el desacoplamiento total de la capa de datos:

| Microservicio | Base de datos |
|---|---|
| ms-actividades | `db_actividades` |
| ms-equipos | `db_equipos` |
| ms-lubricantes | `db_lubricantes` |
| ms-programacion | `db_programacion` |
| ms-ejecucion | `db_ejecucion` |
| ms-stock | `db_stock` |
| ms-observaciones | `db_observaciones` |
| ms-usuarios | `db_usuarios` |
| ms-notificaciones | `db_notificaciones` |
| ms-reportes | `db_reportes` |

---

## 4. Estructura de Datos por Servicio

### ms-actividades
```
actividad
├── id (PK, BIGINT AUTO_INCREMENT)
├── faena (VARCHAR 100)
├── equipo (VARCHAR 150)
├── codigo_equipo (VARCHAR 100)
├── componente (VARCHAR 150)
├── punto_lubricacion (VARCHAR 100)
├── cantidad (INT)
├── consumo_estimado_g (INT)
├── tiempo_estimado_min (INT)
├── id_equipo_ref (FK lógica → ms-equipos)
├── id_lubricante_ref (FK lógica → ms-lubricantes)
└── id_tipo_actividad_ref (FK lógica → catálogo interno)

tipo_actividad
├── id (PK)
└── nombre (VARCHAR 100)
```

### ms-equipos
```
planta
├── id (PK)
└── nombre (VARCHAR 100)

linea
├── id (PK)
├── nombre (VARCHAR 100)
└── id_planta (FK → planta)

equipo
├── id (PK)
├── nombre (VARCHAR 150)
├── codigo (VARCHAR 100)
└── id_linea (FK → linea)
```

### ms-lubricantes
```
marca_lubricante
├── id (PK)
└── nombre (VARCHAR 100)

lubricante
├── id (PK)
├── nombre (VARCHAR 150)
└── id_marca (FK → marca_lubricante)
```

### ms-programacion
```
programacion
├── id (PK)
├── id_actividad_ref (FK lógica → ms-actividades)
├── intervalo_dias (INT)
├── f_programacion (DATE)
├── f_vencimiento (DATE)
└── estado (ENUM: pendiente, en_curso, completada, vencida)
```

### ms-ejecucion
```
ejecucion
├── id (PK)
├── id_programacion_ref (FK lógica → ms-programacion)
├── id_actividad_ref (FK lógica → ms-actividades)
├── f_ejecucion (DATETIME)
├── responsable (VARCHAR 100)
├── consumo_real_g (INT)
├── tiempo_real_min (INT)
└── estado (ENUM: realizada, no_realizada, parcial)
```

### ms-stock
```
producto_stock
├── id (PK)
├── id_lubricante_ref (FK lógica → ms-lubricantes)
├── cantidad_actual_g (INT)
├── cantidad_minima_g (INT)
└── ubicacion (VARCHAR 100)

movimiento_stock
├── id (PK)
├── id_producto (FK → producto_stock)
├── tipo (ENUM: entrada, salida)
├── cantidad_g (INT)
├── fecha (DATETIME)
└── id_ejecucion_ref (FK lógica → ms-ejecucion)
```

### ms-observaciones
```
observacion
├── id (PK)
├── id_actividad_ref (FK lógica → ms-actividades)
├── id_ejecucion_ref (FK lógica → ms-ejecucion)
├── descripcion (TEXT)
├── fecha (DATETIME)
└── autor (VARCHAR 100)

imagen_observacion
├── id (PK)
├── id_observacion (FK → observacion)
├── url_imagen (VARCHAR 255)
└── fecha_subida (DATETIME)
```

### ms-usuarios
```
usuario
├── id (PK)
├── nombre (VARCHAR 100)
├── email (VARCHAR 150, UNIQUE)
├── password_hash (VARCHAR 255)
├── rol (ENUM: tecnico, supervisor, admin)
└── activo (BOOLEAN)
```

### ms-notificaciones
```
notificacion
├── id (PK)
├── id_programacion_ref (FK lógica → ms-programacion)
├── tipo (ENUM: proximidad, vencimiento, stock_bajo)
├── mensaje (TEXT)
├── leida (BOOLEAN)
├── fecha_generacion (DATETIME)
└── destinatario_ref (FK lógica → ms-usuarios)
```

### ms-reportes
```
reporte
├── id (PK)
├── tipo (ENUM: kpi_mensual, actividades_por_linea, consumo_lubricante)
├── fecha_generacion (DATETIME)
├── generado_por_ref (FK lógica → ms-usuarios)
└── datos_json (JSON)
```

---

## 5. Reglas de Negocio

### ms-actividades
- Una actividad debe tener equipo, componente y lubricante asignados obligatoriamente
- El consumo estimado no puede ser negativo
- No pueden existir dos actividades idénticas (mismo equipo + componente + tipo de actividad)

### ms-programacion
- La fecha de vencimiento se calcula automáticamente: `f_programacion + intervalo_dias`
- No se puede programar una actividad con intervalo = 0 sin justificación, EXCEPTO QUE LA JEFATURA LO INDIQUE
- Una actividad solo puede tener una programación activa a la vez

### ms-ejecucion
- No se puede registrar una ejecución sin una programación asociada activa
- Al registrar ejecución como `realizada`, se descuenta automáticamente el consumo real del stock (notifica a ms-stock)
- Una ejecución `no_realizada` debe incluir motivo obligatorio. DONDE ACTUALMENTE NO SE GUARDA NI SE REALIZA SEGUIMIENTO A ESTE CASO

### ms-stock
- Si `cantidad_actual_g < cantidad_minima_g`, se genera alerta automática en ms-notificaciones. ACTUALMENTE ESTE SERVICIO NO FUNCIONA
- No se permite registrar una salida de stock mayor a la cantidad disponible
- Todo movimiento de stock queda registrado con fecha y referencia de ejecución

### ms-usuarios
- Los roles determinan los permisos: técnico (solo lectura y ejecución), supervisor (CRUD actividades), admin (acceso total)
- No se puede eliminar un usuario con ejecuciones registradas, solo desactivar
- El email debe ser único en el sistema

### ms-observaciones
- Una observación puede tener máximo 5 imágenes adjuntas
- Las imágenes se almacenan en servidor de archivos y solo se guarda la URL en BD
- Solo el autor o un admin puede eliminar una observación

---

## 6. Stack Tecnológico

| Capa | Tecnología |
|---|---|
| Backend | Java 17 + Spring Boot 3 |
| Persistencia | Spring Data JPA + Hibernate |
| Base de datos | MariaDB (XAMPP) |
| Patrón | CSR + DTO + Repository |
| Frontend | React + Vite |
| Comunicación | API REST (JSON) |
| Control de versiones | Git + GitHub |

---

## 7. Distribución del Equipo (3 integrantes) #ESTO ES MUTABLE

| Integrantes | Microservicios asignados |
|---|---|
| Gustavo | ms-usuarios, ms-actividades, ms-equipos | CONOZCO LOS EQUIPOS
| Sebastian | ms-lubricantes, ms-programacion, ms-ejecucion |
| Jordy | ms-stock, ms-observaciones, ms-notificaciones, ms-reportes |

---

## 8. Repositorio

```
sgli/
├── ms-actividades/
├── ms-equipos/
├── ms-lubricantes/
├── ms-programacion/
├── ms-ejecucion/
├── ms-stock/
├── ms-observaciones/
├── ms-usuarios/
├── ms-notificaciones/
├── ms-reportes/
└── README.md
```
