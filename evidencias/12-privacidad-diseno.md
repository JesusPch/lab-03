# Actividad 12 — Análisis de Minimización y Privacidad por Defecto

## 1. Análisis del Escenario
El desarrollador propone registrar una amplia variedad de datos personales y socioeconómicos (`salario`, `ocupacion`, `estado_civil`, `edad`, `fecha_nacimiento`, etc.) bajo la justificación de *"guardar todos los datos por si posteriormente necesitamos alguno"*. 

Esta premisa vulnera los estándares de gobierno de datos y marcos regulatorios como la LFPDPPP y el GDPR.

---

## 2. Respuestas al Cuestionario

### 1. ¿Qué problemas identificas?
* **Acaparamiento de datos (Data Hoarding):** Recopilar información personal sin una finalidad clara, presente y justificada ("por si acaso").
* **Redundancia e inconsistencia:** Almacenar `edad` y `fecha_nacimiento` simultáneamente duplica información. La edad cambia dinámicamente con el tiempo y calcularla a partir de un campo estático guardado en la base de datos genera datos desactualizados.
* **Aumento drástico de la superficie de ataque:** Guardar datos financieros y personales sensibles (`salario`, `ocupacion`, `estado_civil`) sin necesidad operativa incrementa el riesgo legal y el impacto financiero de la organización ante una eventual filtración o acceso no autorizado.
* **Violación del Aviso de Privacidad:** No se puede informar válidamente al titular sobre el uso de sus datos si la propia empresa desconoce la finalidad futura para la cual los está almacenando.

---

### 2. ¿Qué principio de privacidad se está ignorando?
Se están ignorando tres principios fundamentales:

1. **Principio de Minimización de Datos:** Exige que los datos personales sean adecuados, pertinentes y limitados a lo estrictamente necesario en relación con las finalidades del tratamiento.
2. **Principio de Limitación de la Finalidad:** Establece que los datos deben ser recopilados para fines determinados, explícitos y legítimos, y no ser tratados posteriormente de manera incompatible con dichos fines.
3. **Privacidad por Diseño y por Defecto (*Privacy by Design and by Default*):** Requiere que la arquitectura de sistemas configure, por defecto, la menor recolección posible de datos personales.

---

### 3. ¿Qué preguntas realizarías antes de aprobar el diseño?
Antes de autorizar el script DDL en la base de datos, el DBA/Oficial de Privacidad debe cuestionar:
* ¿Cuál es la finalidad operativa y de negocio de cada uno de los campos propuestos?
* ¿La aplicación es una plataforma financiera o crediticia que justifique legalmente solicitar `salario` y `ocupacion`?
* ¿Está cada uno de estos campos explícitamente contemplado en el Aviso de Privacidad que firmará o aceptará el usuario?
* ¿Por qué se requieren tanto `fecha_nacimiento` como `edad`?
* ¿Cuáles son los tiempos de retención y las políticas de purga previstas para este conjunto de datos?

---

### 4. ¿Qué datos eliminarías?
Salvo que la aplicación pertenezca a un sector regulado específico (como evaluación de crédito o banca), se deben eliminar del formulario de registro inicial:

* **`edad`:** Eliminar por completo. Si se requiere saber la edad, se calcula dinámicamente desde la `fecha_nacimiento`.
* **`salario`:** Innecesario para el registro operativo estándar de un cliente.
* **`estado_civil`:** Innecesario y potencialmente irrelevante para la prestación del servicio.
* **`ocupacion`:** Innecesario salvo que exista un proceso de perfilamiento justificado y consentido.

---

### 5. ¿Qué datos requerirían una justificación adicional?
* **`rfc`:** Solo se justifica si el cliente solicita la emisión de comprobantes fiscales/facturas[cite: 1]. No debe ser un campo obligatorio en el registro inicial de la plataforma.
* **`direccion` y `codigo_postal`:** Se justifican únicamente si la aplicación requiere envíos físicos de productos o el cálculo de impuestos regionales.
* **`telefono`:** Se justifica solo si se utiliza como factor de autenticación (2FA), notificaciones operativas o soporte urgente.
* **`fecha_nacimiento`:** Se justifica únicamente si la plataforma necesita verificar la mayoría de edad para dar acceso al servicio.

---

### 6. ¿Cómo aplicarías privacidad por defecto?

Para implementar privacidad por defecto en el diseño de la base de datos PostgreSQL:

#### A. Reducción de la Tabla Base (`clientes`)
Diseñar la tabla de registro únicamente con los atributos estrictamente indispensables para la autenticación y prestación básica del servicio:

```sql
-- Tabla principal minimizada
CREATE TABLE clientes (
    id BIGSERIAL PRIMARY KEY,
    nombre TEXT NOT NULL,
    correo TEXT NOT NULL UNIQUE,
    fecha_alta TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
# Actividad 13 — Diferenciar Controles Técnicos y Administrativos

## 1. Matriz de Clasificación de Medidas

A continuación se presenta la clasificación de las medidas de seguridad y gobierno de datos según su naturaleza técnica, administrativa o híbrida:

| Medida | Técnico | Administrativo |
| :--- | :---: | :---: |
| **Crear roles de PostgreSQL** | **X** | |
| **Política de retención** | | **X** |
| **Cifrado de datos** | **X** | |
| **Procedimiento para solicitudes ARCO** | | **X** |
| **Vista con información limitada** | **X** | |
| **Clasificación de información** | | **X** |
| **Registro de auditoría** | **X** | **X** |
| **Política de control de acceso** | | **X** |

---

## 2. Justificación por Tipo de Control

### Medidas Estrictamente Técnicas (Lógicas)
* **Crear roles de PostgreSQL:** Es un mecanismo de control de acceso lógico ejecutado mediante comandos SQL (`CREATE ROLE`, `GRANT`) directamente en el motor de base de datos para limitar permisos de usuarios.
* **Cifrado de datos:** Es la implementación tecnológica de algoritmos criptográficos (como AES-256, TLS/SSL o funciones de `pgcrypto`) para proteger la confidencialidad de los datos en reposo y tránsito.
* **Vista con información limitada:** Objeto DDL (`CREATE VIEW`) que aplica transformaciones, filtros o agregaciones para restringir las columnas y filas visibles a ciertos usuarios.

### Medidas Estrictamente Administrativas (Organizacionales)
* **Política de retención:** Documento normativo formal que establece los plazos legales, fiscales y de negocio durante los cuales debe conservarse la información antes de ser destruida o anonimizada.
* **Procedimiento para solicitudes ARCO:** Manual operativo e institucional que define los pasos, validaciones de identidad y tiempos de respuesta que sigue el personal de la empresa para atender derechos de los usuarios.
* **Clasificación de información:** Esquema o estándar de gobernanza que establece los niveles de sensibilidad de los datos (Público, Interno, Confidencial, Sensible) y las reglas para su manejo.
* **Política de control de acceso:** Documento de directrices corporativas que establece la matriz de responsabilidades y la regla del menor privilegio (quién debe tener acceso a qué datos y por qué).

---

## 3. Justificación de Medidas que Involucran Ambos Controles

### Registro de Auditoría (Técnico y Administrativo)
El registro de auditoría es una medida híbrida indispensable en el gobierno de datos:

1. **Perspectiva Técnica:** Requiere la activación y configuración de extensiones en el motor de base de datos (como `pg_audit`, *triggers* de auditoría o capturadores de eventos en el log de PostgreSQL) que registran automáticamente transacciones, comandos `UPDATE`/`DELETE` e inicios de sesión.
2. **Perspectiva Administrativa:** Requiere una directiva formal que especifique qué eventos es obligatorio auditar, con qué frecuencia el área de seguridad debe revisar los registros, cuánto tiempo se archivarán las bitácoras y cómo se actúa legal u operativamente ante un hallazgo o anomalía.

---

### Principio Fundamental de Integración
> **Regla de Gobierno:** La **política administrativa** establece *el qué y el por qué* (la regla de negocio o legal), mientras que el **control técnico** implementa *el cómo* (el mecanismo de sofware que exige su cumplimiento inquebrantable en la base de datos). Un control técnico sin política carece de fundamento normativo, y una política sin control técnico se convierte en una simple declaración de intenciones sin aplicación real.

