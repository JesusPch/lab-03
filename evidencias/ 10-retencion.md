# Actividad 10 — Política de Retención y Conservación de Datos

## 1. Análisis del Escenario
La propuesta de la organización de **"conservar todos los datos de clientes indefinidamente porque podrían ser útiles en el futuro"** contraviene las mejores prácticas de gobierno de datos y los marcos legales de protección de datos personales

---

## 2. Respuestas al Cuestionario

### Pregunta 1: ¿Es compatible con el principio de minimización?
**No, es totalmente incompatible**

El **Principio de Minimización** y el **Principio de Limitación del Plazo de Conservación** establecen que los datos personales únicamente deben ser tratados y conservados durante el tiempo estrictamente necesario para cumplir con la finalidad informada en el Aviso de Privacidad. Almacenar información de forma indefinida ante la mera especulación de que "podría ser útil en el futuro" carece de finalidad legítima, constituye un tratamiento desproporcionado y violenta las normativas de privacidad

---

### Pregunta 2: ¿Qué riesgos genera conservar información innecesaria?

* **Mayor Superficie de Ataque y Severidad en Filtraciones (Data Breach):** En caso de un incidente de seguridad o acceso no autorizado, el volumen de datos expuestos y el número de titulares afectados será innecesariamente alto, incrementando el impacto y los daños a la organización.
* **Sanciones Legal y Multas Regulatorias:** Exposición a infracciones administrativas graves por parte de las autoridades de protección de datos (como el INAI) al carecer de un fundamento legal que justifique el almacenamiento a largo plazo.
* **Costos Operativos y de Almacenamiento Ineficiente:** Gastos innecesarios en infraestructura de base de datos, licencias, sistemas de almacenamiento, respaldos y procesamiento de información obsoleta.
* **Deterioro de la Calidad e Integridad de la Información:** Conservar registros desactualizados ("datos basura") distorsiona los análisis operativos y la toma de decisiones.
* **Carga Administrativa en Solicitudes ARCO:** Dificultad y costo técnico para ubicar, actualizar o cancelar datos dispersos que ya no tienen razón de existir en el sistema.

---

### Pregunta 3: ¿Qué debería determinar la organización antes de establecer un periodo de retención?

Antes de definir un plazo de conservación, la organización debe auditar y determinar:

1. **Vigencia de la Relación Contractual / Operativa:** Tiempo en que la información es indispensable para prestar el servicio o producto al cliente.
2. **Obligaciones Legales, Fiscales y Mercantiles:** Plazos fijados por leyes específicas (ej. Código de Comercio, Código Fiscal) que exigen conservar comprobantes y datos contables durante períodos determinados (ej. 5 a 10 años).
3. **Plazos de Prescripción de Responsabilidades:** Lapso durante el cual el cliente o la autoridad podrían entablar acciones legales, demandas o reclamaciones.
4. **Fase de Bloqueo:** Definición del momento en que los datos dejan de estar disponibles para la operación diaria y pasan a estar aislados exclusivamente a disposición de autoridades judiciales o administrativas.
5. **Método de Destrucción o Anonimización:** Criterio técnico para determinar si, al vencer el plazo, los datos se eliminarán mediante borrado seguro o se someterán a una anonimización irreversible.

---

### Pregunta 4: ¿Qué información debería documentarse para justificar el periodo seleccionado?

La organización debe formalizar una **Política de Retención y Borrado de Datos**, incluyendo

| Elemento Documentado | Descripción |
| :--- | :--- |
| **Categoría de Datos** | Especificar qué tipos de datos se conservan (ej. datos de identificación, transaccionales, de soporte). |
| **Periodo de Conservación Exacto** | Definición del plazo límite (ej. "5 años a partir de la terminación del contrato"). |
| **Fundamento Legal / Base de Legitimidatd** | Referencia explícita a la ley, reglamento o contrato que exige o justifica dicho periodo de retención. |
| **Finalidad del Almacenamiento** | Razón de negocio o legal por la cual se guarda la información durante ese tiempo. |
| **Mecanismo Operativo de Supresión** | Procedimiento técnico mediante el cual el DBA o los sistemas programados destruirán o anonimizarán los registros vencidos. |
| **Roles Responsables** | Identificación de los encargados de supervisar el cumplimiento de la política (Oficial de Privacidad, DBA, Legal). |