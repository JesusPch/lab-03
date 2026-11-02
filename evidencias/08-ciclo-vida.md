# Actividad 8 — Derecho de Cancelación y Ciclo de Vida de los Datos

## 1. Análisis Previo a la Cancelación

Antes de ejecutar una instrucción de eliminación (`DELETE`) sobre la tabla principal `clientes`, se debe auditar la presencia del dato en todo el ecosistema de la organización:

* **¿Existen otras tablas relacionadas?**  
  Sí. El titular puede estar registrado en tablas de soporte (`clientes_soporte`), transacciones de compra, registros de facturación o tickets de atención.
* **¿Existen respaldos?**  
  Sí. Los respaldos completos de la base de datos (*dumps*, imágenes snapshot, *WAL logs*) generados previamente contienen la información del cliente.
* **¿Existen réplicas?**  
  Sí. Si la arquitectura cuenta con réplicas de lectura en *standby* o servidores secundarios, el dato reside o se ha sincronizado en ellos.
* **¿Existen registros de auditoría?**  
  Sí. Las bitácoras del sistema (como `pg_audit` o logs de PostgreSQL) guardan el historial de operaciones `INSERT` y `UPDATE` realizadas previamente sobre el registro.
* **¿Existen copias utilizadas para analítica?**  
  Sí. Pueden existir vistas materializadas, *Data Warehouses* o extractos de datos históricos consumidos por el área de analítica.
* **¿Existen archivos exportados?**  
  Sí. Es común encontrar reportes descargados previamente en formatos CSV, Excel o PDF por parte de usuarios autorizados.
* **¿Existen sistemas externos?**  
  Sí. Integraciones mediante API a plataformas de correo masivo, CRM externos o proveedores de pago.

---

## 2. Mapa del Ecosistema de Datos de Clientes
```text
clientes (Tabla Principal / Base de Datos Operativa)
    │
    ├── tablas_relacionadas (clientes_soporte, facturación, transacciones)
    ├── respaldos (archivos .dump, snapshots, copias de seguridad pasivas)
    ├── réplicas (nodos secundarios / réplicas de lectura)
    ├── auditoría (logs de transacciones, bitácoras pg_audit)
    ├── analítica (vistas materializadas, Data Warehouse, Data Lakes)
    ├── reportes / exportaciones (archivos CSV, Excel, PDFs generados)
    └── sistemas externos (CRM, herramientas de marketing, pasarelas de pago).
```

### Pregunta: ¿Por qué eliminar un registro de una tabla no necesariamente significa que toda la información relacionada con esa persona haya desaparecido de la organización?
   
LABORATORIO 03 Alumnos (Privacidad Cumplimiento y Gobierno de Datos)
Respuesta:

Eliminar un registro mediante DELETE FROM clientes WHERE id = X; únicamente borra la fila en la tabla productiva activa de la base de datos PostgreSQL. No garantiza la desaparición total de la información de la organización debido a los siguientes factores técnicos y procedimentales
LABORATORIO 03 Alumnos (Privacidad Cumplimiento y Gobierno de Datos)

Dispersión en la Arquitectura: Los datos personales frecuentemente flu yen hacia otros sistemas (réplicas, Data Warehouses, CRM y herramientas de correo) mediante procesos ETL o sincronizaciones automáticas El DELETE en la tabla principal no elimina automáticamente las copias en esos destinos secundarios

Medios Pasivos y Respaldos (Backups): Las copias de seguridad creadas en fechas anteriores mantienen la información tal como existía en ese momento Borrar el dato operativo no altera retroactivamente los archivos de respaldo existentes

Logs y Bitácoras de Auditoría: Para fines de trazabilidad y seguridad, las bitácoras registran las actividades ejecutadas Los datos personales pueden quedar grabados en texto plano o estructurado dentro de los archivos de registro del servidor

Archivos Locales Exportados: Diversos departamentos pueden haber generado descargas operativas (archivos CSV o Excel) almacenadas en discos locales o carpetas compartidas

Periodo de Bloqueo y Obligaciones Legales: legalmente (conforme a la LFPDPPP), el ejercicio del derecho de cancelación no implica la destrucción inmediata de toda huella si existen obligaciones fiscales o contractuales. En su lugar, debe realizarse un periodo de bloqueo donde el dato se retira de la operación diaria pero se conserva aislado hasta que expiren las prescripciones legales aplicables.   
LABORATORIO 03 Alumnos (Privacidad Cumplimiento y Gobierno de Datos)

# Actividad 9 — Derecho de Oposición

## 1. Análisis Inicial
**¿Es necesario eliminar inmediatamente todos sus datos de la base?**  
**No**. El **Derecho de Oposición (O)** no implica la eliminación o cancelación de la información personal del sistema, sino la **cesación del tratamiento de los datos para una finalidad específica** (en este caso, la finalidad secundaria de promociones o marketing). El cliente puede continuar siendo un usuario activo para la finalidad primaria (compras, facturación, soporte).

---

## 2. Desarrollo del Análisis

### 1. ¿Qué información podría necesitar mantenerse?
Para garantizar la continuidad del servicio y el cumplimiento de obligaciones legales y contractuales, la organización debe conservar:
* **Información Operativa:** `nombre`, `correo` y `telefono` para la atención de solicitudes, soporte técnico y autenticación en la plataforma.
* **Información Fiscal y Contable:** `rfc`, `codigo_postal` e historial de facturación para cumplir con los periodos de retención exigidos por las leyes tributarias.
* **Registro de la Solicitud de Oposición (Lista de Exclusión / Suppression List):** Es necesario mantener un registro de la negativa del titular para garantizar técnicamente que sus datos no vuelvan a ser incluidos por error en futuras campañas publicitarias[cite: 1].

---

### 2. ¿Qué información podría dejar de utilizarse para determinada finalidad?
* **Datos de Contacto en Campañas Promocionales:** El `correo` y `telefono` deben excluirse de los envíos de boletines, ofertas, llamadas telefónicas o correos masivos de mercadotecnia[cite: 1].
* **Datos Analíticos de Perfilamiento:** La `edad`, `ciudad` y el historial de comportamiento de compra del titular deben dejar de ser utilizados para segmentación publicitaria, perfiles de consumo o algoritmos de recomendación personalizable[cite: 1].

---

### 3. ¿Qué mecanismos técnicos podrían separar finalidades?
Para aislar la finalidad primaria de la secundaria dentro de la base de datos PostgreSQL, se pueden implementar los siguientes controles[cite: 1]:

1. **Campos Banderas de Consentimiento (`Opt-Out Flags`):**  
   Agregar columnas booleanas en la tabla de clientes para registrar el estado de los permisos:
   ```sql
   ALTER TABLE clientes 
   ADD COLUMN permite_marketing BOOLEAN DEFAULT TRUE,
   ADD COLUMN fecha_oposicion TIMESTAMP;
   ```
### 4. ¿Por qué la decisión final no debe ser tomada únicamente por el DBA?
El DBA es responsable de la implementación técnica del control, pero no de la evaluación jurídica ni de negocio La decisión debe ser multidisciplinaria por las siguientes razones

Análisis de Validez Legal (Área Jurídica / Privacidad): Determina si la oposición procede jurídicamente o si existen excepciones legales (ej. cuando el tratamiento sea necesario para el cumplimiento de una obligación legal o un contrato activo)

Impacto Operativo y de Negocio (Propietario de Datos / Dirección): Evalúa el impacto comercial y contractual de restringir el tratamiento del dato

Riesgo de Incumplimiento: Si un DBA toma de forma unilateral la decisión de borrar datos sin consultar, podría destruir registros exigidos por la ley fiscal; o por el contrario, si ignora la solicitud por falta de un mecanismo técnico, expondrá a la empresa a sanciones por parte de la autoridad reguladora