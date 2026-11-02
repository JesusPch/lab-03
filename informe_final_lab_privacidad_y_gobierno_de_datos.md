# INFORME FINAL: PRIVACIDAD, CUMPLIMIENTO Y GOBIERNO DE DATOS EN POSTGRESQL

## 1. Introducción
El manejo de bases de datos relacionales en entornos corporativos enfrenta el reto fundamental de equilibrar la eficiencia operativa con la protección estricta de los datos personales. Tradicionalmente, la administración de bases de datos se centró de forma exclusiva en la disponibilidad, integridad y rendimiento, relegando la privacidad a políticas en papel o parches de seguridad reactivos. Esta desconexión suele originar prácticas desproporcionadas como el acaparamiento de datos (*data hoarding*), la conservación indefinida de registros obsoletos, el acceso irrestricto de usuarios a tablas de producción y la exposición de datos reales en entornos de pruebas y analítica.

El presente informe sintetiza la aplicación de los principios de protección de datos (con énfasis en la LFPDPPP y el marco de derechos ARCO) a la administración técnica en PostgreSQL 16. A través del análisis de escenarios reales, se demuestra cómo la privacidad no puede depender de la buena voluntad, sino que debe traducirse en arquitecturas con **Privacidad por Diseño y por Defecto**, donde los controles técnicos (vistas, roles, *triggers*, cifrado y bitácoras) ejecuten de forma inquebrantable las directrices administrativas fijadas por la gobernanza de datos.

---

## 2. Inventario de Datos
El primer paso en la gobernanza de datos es la identificación y clasificación sistemática de los activos de información para determinar su nivel de sensibilidad y el grado de protección requerido.

| Dato | Clasificación | Nivel de Sensibilidad | Razón de Clasificación |
| :--- | :--- | :---: | :--- |
| `nombre` | PII Directo (Identificación) | **Alto** | Identifica de forma explícita y directa al titular de la información. |
| `correo` | PII Directo (Contacto / Login) | **Alto** | Permite la localización del titular y actúa como credencial en plataformas digitales. |
| `telefono` | PII Directo (Contacto) | **Medio-Alto** | Permite la comunicación directa y puede vincularse a factores de autenticación (2FA). |
| `rfc` | PII Directo / Fiscal | **Muy Alto** | Identificador fiscal vinculante con implicaciones legales y patrimoniales. |
| `fecha_nacimiento` | PII Indirecto | **Medio** | Permite verificar la mayoría de edad; combinado con otros datos puede re-identificar al titular. |
| `edad` | PII Indirecto / Derivado | **Medio** | Atributo dinámico derivado de la fecha de nacimiento; innecesario como campo persistente. |
| `ciudad` | PII Indirecto / Ubicación | **Medio** | Dato geográfico de nivel macro utilizado para logística y segmentación. |
| `codigo_postal` | PII Indirecto / Ubicación | **Medio** | Permite la micro-segmentación territorial y el cálculo de impuestos regionales. |
| `salario` | Dato Financiero / Sensible | **Muy Alto** | Revela la capacidad económica del titular; su filtración genera alto riesgo de extorsión o discriminación. |
| `estado_civil` | Dato Personal Sensible | **Alto** | Pertenece a la esfera íntima del individuo; irrelevante para la mayoría de las operaciones comerciales. |
| `ocupacion` | Dato Personal / Socioeconómico | **Medio** | Utilizado para perfilamiento; requiere justificación explícita de negocio. |

---

## 3. Finalidades
El **Principio de Limitación de la Finalidad** exige que todo dato recolectado tenga un propósito determinado, explícito y legítimo, prohibiendo su tratamiento posterior para fines incompatibles con los informados al titular.

* **Finalidades Primarias (Indispensables):**
  * **Gestión de la relación contractual y prestación del servicio:** Uso de `nombre`, `correo` y `telefono` para autenticación, soporte técnico y notificaciones operativas.
  * **Cumplimiento de obligaciones fiscales y legales:** Uso de `rfc`, `nombre` y dirección fiscal para la emisión de comprobantes fiscales digitales y registros contables exigidos por ley.
* **Finalidades Secundarias (Optativas / Requieren Consentimiento Explícito):**
  * **Mercadotecnia y prospección comercial:** Uso del `correo` para el envío de boletines o promociones.
  * **Análisis de mercado y perfilamiento:** Uso de `ciudad`, `codigo_postal` y rango de `edad` para estadísticas de consumo y optimización de campañas regionales.

---

## 4. Minimización
El **Principio de Minimización** establece que una organización solo debe recolectar y conservar los datos estrictamente necesarios, pertinentes y adecuados para la finalidad autorizada.

### Datos a Eliminar del Registro Estándar
1. **`salario` y `estado_civil`:** Deben eliminarse completamente del formulario de registro de clientes. Representan un riesgo desproporcionado y carecen de utilidad para operaciones comerciales no crediticias.
2. **`edad`:** Debe eliminarse como columna persistente en la base de datos. Representa redundancia informativa y genera inconsistencia con el tiempo.

### Datos a Limitar o Condicionar
1. **`fecha_nacimiento`:** En lugar de almacenar la fecha exacta de nacimiento, si la aplicación solo requiere validar la mayoría de edad, la interfaz debe procesar una bandera booleana (`es_mayor_edad BOOLEAN`) en tiempo de registro, evitando guardar la fecha real.
2. **`rfc`:** No debe solicitarse en el alta inicial del cliente. Su recolección debe posponerse al módulo de facturación y aplicarse únicamente cuando el cliente requiera un comprobante fiscal.
3. **`ocupacion`:** Mover a un perfil opcional posterior al registro, bajo un mecanismo de consentimiento *Opt-In*.

---

## 5. Control de Acceso
El control de acceso en la base de datos PostgreSQL debe estructurarse mediante el **Principio del Menor Privilegio** y un esquema de **Control de Acceso Basado en Roles (RBAC)**, eliminando el acceso directo de los usuarios finales a las tablas productivas base.

```text
                                [ PostgreSQL 16 ]
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           ▼                            ▼                            ▼
   [ Rol: Atención ]            [ Rol: Facturación ]         [ Rol: Analítica ]
           │                            │                            │
           ▼                            ▼                            ▼
  Vista Operativa            Vista Datos Fiscales         Vista Anonimizada / DW
(nombre, correo, tel)          (rfc, dirección)             (datos agregados)
```

* **Atención al Cliente:** Rol restringido a la consulta de `nombre`, `correo` y `telefono` mediante vistas específicas para soporte. No posee permisos sobre datos fiscales ni financieros.
* **Facturación:** Rol con acceso exclusivo a la vista de emisión de facturas (`rfc`, dirección fiscal, monto). No accede a preferencias de marketing ni a datos de perfilamiento.
* **Analítica:** Se revoca todo acceso a la tabla base `clientes`. El área de analítica consulta únicamente vistas materializadas anonimizadas o repositorios *Data Warehouse* donde los identificadores directos (nombres, correos, RFC) han sido suprimidos o agrupados.
* **Marketing:** Acceso condicionado a vistas filtradas que incluyen únicamente a usuarios con la bandera `permite_marketing = TRUE`.
* **Desarrollo:** Prohibición absoluta de utilizar datos reales en entornos de prueba. Los desarrolladores reciben bases de datos pobladas con datos sintéticos o sometidas a un proceso de enmascaramiento estático (*Data Masking*).
* **Administrador de la Base de Datos (DBA):** El DBA posee privilegios para administrar la infraestructura y la seguridad técnica, pero no debe acceder al contenido de los datos de negocio sin una orden de auditoría registrada.

---

## 6. Derechos ARCO
La atención de los derechos de Acceso, Rectificación, Cancelación y Oposición (ARCO) requiere controles técnicos específicos en el motor PostgreSQL para garantizar el cumplimiento legal y la trazabilidad.

```text
Flujo Técnico de Atención ARCO:
[Solicitud Titular] ──> [Portal ARCO / Ticket] ──> [Validación OTP/Firma] ──> [Ejecución SQL + Audit Log]
```

* **Acceso y Rectificación:**
  * **Implicación Técnica:** Todo cambio sobre un dato personal (ej. actualización de correo) debe vincularse al ID de ticket formal otorgado por el área de privacidad.
  * **Bitácora de Auditoría:** La modificación ejecuta un *trigger* que registra en la bitácora (`pg_audit`) el usuario que realizó el cambio, la dirección IP, el *timestamp* exacto y el histórico de valores anterior y nuevo (*Delta*).
* **Cancelación:**
  * **Análisis Previo:** Ejecutar un `DELETE` en la tabla principal no destruye el dato de la organización. Se deben identificar y procesar las copias en tablas relacionadas, réplicas de lectura, *dumps* de respaldo, *logs* de transacciones, vistas materializadas, extractos de analítica y sistemas externos (CRM/APIs).
  * **Periodo de Bloqueo:** Conforme a la ley, la cancelación no elimina el dato de inmediato si existen obligaciones fiscales. Se aplica un **bloqueo técnico** (marcar el registro como inactivo y restringir su acceso exclusivamente a autoridades) durante el plazo de prescripción legal antes de su purga definitiva.
* **Oposición:**
  * **Mecanismo Técnico:** No implica borrar al cliente, sino cesar el tratamiento para finalidades secundarias. Se implementan columnas booleanas (`permite_marketing = FALSE`), vistas filtradas para el área comercial y sincronización automática con la lista de exclusión (*Suppression List*).

---

## 7. Retención
Establecer una política de retención evita el almacenamiento indefinido de información obsoleta y reduce la superficie de ataque. El periodo de conservación debe determinarse bajo los siguientes criterios:

1. **Vigencia Operativa:** Conservación del dato mientras la relación contractual o la cuenta del usuario permanezca activa.
2. **Plazos Legales y Fiscales:** Retención obligatoria de datos contables y de facturación por el periodo fijado por la legislación tributaria (ej. 5 a 10 años posteriores a la transacción).
3. **Periodo de Bloqueo:** Fases aisladas de conservación pasiva únicamente para la atención de posibles responsabilidades jurídicas o requerimientos judiciales.
4. **Disposición Final:** Definición clara del momento en que el registro debe ser sometido a una **purga segura** (borrado físico no recuperable) o a un proceso de **anonimización irreversible** para conservación estadística.

---

## 8. Gobierno de Datos
La matriz de gobierno de datos vincula cada atributo con su responsabilidad funcional, nivel de protección, ciclo de vida y acción al finalizar su uso.

| Dato | Finalidad | Responsable | Usuarios Autorizados | Protección | Retención | Acción al Finalizar |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`nombre`** | Identificación y prestación del servicio. | Atención al Cliente | Atención al Cliente, Facturación, Dirección. | Alta. Cifrado en reposo y tránsito. | 5 años tras la última transacción. | Purga segura. |
| **`correo`** | Comunicación operativa y marketing consentido. | Marketing / Atención al Cliente | Marketing, Atención al Cliente, Facturación. | Alta. Cifrado y control de acceso. | 3 años de inactividad o revocación. | Eliminación o anonimización. |
| **`telefono`** | Soporte, verificación de identidad y urgencias. | Atención al Cliente | Atención al Cliente. | Media-Alta. Cifrado parcial. | Relación activa, o 1 año post-servicio. | Eliminación. |
| **`rfc`** | Emisión obligatoria de facturas fiscales. | Facturación | Facturación, Contabilidad. | Muy Alta. Cifrado fuerte y acceso limitado. | 5-10 años (requisito fiscal). | Purga segura. |
| **`edad`** | Análisis demográfico y de mercado. | Marketing / Producto | Analítica, Marketing. | Media. Anonimización para análisis. | Relación activa o actualización. | Anonimización / Eliminación. |
| **`ciudad`** | Análisis geográfico de ventas y logística. | Logística | Logística, Analítica, Marketing. | Media. Datos agregados para análisis. | Relación activa. | Anonimización / Eliminación. |
| **`codigo_postal`** | Logística precisa e impuestos regionales. | Logística / Facturación | Logística, Facturación. | Media. Datos agregados para análisis. | Relación activa. | Anonimización / Eliminación. |

---

## 9. Evaluación Integral
A partir de la evaluación de la infraestructura y prácticas de la organización, se identificaron 10 riesgos críticos de cumplimiento y seguridad, acompañados de sus respectivos controles de mitigación.

| N.º | Problema Identificado | Principio Afectado | Riesgo de Negocio / Seguridad | Control Propuesto |
| :---: | :--- | :--- | :--- | :--- |
| **1** | Acceso directo de analítica a la tabla productiva. | Confidencialidad / Mínimo Privilegio | Fuga de información por empleados (*insider threat*). | Implementar RBAC y mover analítica a una réplica o *Data Warehouse*. |
| **2** | Analítica consulta PII identificable (`nombre`, `correo`) para estadísticas. | Minimización | Exposición innecesaria de PII ante filtraciones. | Proporcionar vistas anonimizadas con datos agregados. |
| **3** | Conservación de datos de clientes de hace 12 años sin justificación. | Limitación del Plazo de Conservación | Incremento masivo de la superficie de ataque con datos "zombi". | Ejecutar política de borrado de datos pasados los plazos legales. |
| **4** | Ausencia de una política documentada de retención. | Responsabilidad Proactiva (*Accountability*) | Sanciones en auditorías por retención indefinida no justificada. | Formalizar la Política de Retención y Disposición de Datos. |
| **5** | Entrega de copias completas de la base productiva a desarrolladores. | Integridad y Confidencialidad | Extracción de datos reales a entornos de desarrollo desprotegidos. | Prohibir PII real en pruebas; implementar generación de datos sintéticos. |
| **6** | Uso de datos reales en entornos de pruebas de software. | Seguridad | Acceso no autorizado de desarrolladores a datos sensibles de clientes. | Aplicar herramientas de enmascaramiento estático de datos (*Data Masking*). |
| **7** | Uso de correos para marketing sin registro de consentimiento. | Licitud y Consentimiento | Sanciones legales por envíos de spam e impacto reputacional. | Auditar el consentimiento e implementar un portal de preferencias (*Opt-In/Opt-Out*). |
| **8** | Inexistencia de documentación sobre la finalidad de cada dato. | Transparencia y Finalidad | Tratamiento ilegal de datos por falta de aviso explícito. | Mapear datos y actualizar el Aviso de Privacidad institucional. |
| **9** | Coexistencia de datos sensibles (`rfc`, `telefono`) en tabla plana sin cifrar. | Confidencialidad / Seguridad | Robo de identidad masivo y fraude fiscal ante una intrusión. | Aplicar cifrado a nivel de columna/tabla y segregación de datos. |
| **10** | Ausencia de gobernanza y trazabilidad técnica en PostgreSQL. | Responsabilidad Proactiva / Privacidad por Diseño | Incapacidad de auditar cambios o responder a solicitudes ARCO. | Implementar `pg_audit`, *triggers* de auditoría y un comité de gobierno de datos. |

---

## 10. Conclusiones
La realización de este laboratorio permite extraer lecciones fundamentales sobre la intersección entre la privacidad, la seguridad y la administración de bases de datos:

1. **La seguridad es condición necesaria, pero no suficiente para la privacidad:** Un sistema puede estar fuertemente cifrado y protegido contra ataques externos (Seguridad), pero si recolecta datos excesivos o los utiliza para fines no autorizados, violará la ley (Falta de Privacidad). La seguridad protege el dato; la gobernanza protege al titular.
2. **La Privacidad por Diseño debe ser arquitectónica:** Intentar corregir fallas de privacidad en sistemas ya desplegados mediante parches manuales es costoso e ineficiente. La minimización, la segregación de roles y la retención deben integrarse desde la fase de diseño del esquema DDL en PostgreSQL.
3. **El rol del Administrador de Bases de Datos (DBA) ha evolucionado:** El DBA ya no es un mero gestor de espacio en disco y rendimiento de consultas. Hoy actúa como el custodio técnico de la gobernanza de datos, responsable de implementar mediante código SQL, vistas y políticas de auditoría las decisiones legales fijadas por la organización.
4. **La dualidad de controles es indispensable:** Los controles administrativos (políticas, avisos de privacidad) establecen las reglas de negocio, mientras que los controles técnicos (PostgreSQL RBAC, `pg_audit`, cifrado) garantizan su cumplimiento automático inquebrantable. Un control sin el otro resulta inoperante en la práctica corporativa.

# Preguntas de Reflexión: Privacidad, Seguridad y Gobierno de Datos

### 1. ¿Una base de datos técnicamente segura necesariamente cumple con todos los requisitos de privacidad?
No. Existe una diferencia conceptual y práctica fundamental entre seguridad de la información y privacidad de datos:

Seguridad (Protección del Contenedor): Se enfoca en la confidencialidad, integridad y disponibilidad del sistema (tríada CIA). Garantiza que la base de datos esté blindada contra accesos no autorizados mediante firewalls, cifrado de disco (TDE) y contraseñas robustas.

Privacidad (Protección del Titular): Se enfoca en la licitud, finalidad, proporcionalidad y consentimiento en el tratamiento del dato personal.

Una base de datos puede contar con cifrado AES-256 de nivel militar y estar aislada de internet (100% segura), pero si almacena datos irrelevantes sin consentimiento del usuario, los conserva indefinidamente sin justificación o los utiliza para fines no informados en el Aviso de Privacidad, la organización estará en incumplimiento grave de la ley de privacidad. La seguridad protege al activo de información; la privacidad protege los derechos fundamentales de la persona.

### 2. ¿Por qué el principio de mínimo privilegio también contribuye a la privacidad?
El Principio del Menor Privilegio (Least Privilege) establece que cada usuario, rol o aplicación debe tener únicamente los permisos estrictamente indispensables para realizar su función operativa.

Contribuye directamente a la privacidad porque:

Limita la propagación y exposición innecesaria de PII: Impide que personal no autorizado (como desarrolladores, analistas o soporte) consulte columnas sensibles (rfc, salario, direccion).

Mitiga la amenaza interna (Insider Threat): Reduce drásticamente el impacto ante credenciales comprometidas o empleados malintencionados.

Garantiza la separación de finalidades: Asegura que el área de Facturación solo vea datos fiscales y el área de Atención solo datos de soporte, evitando que un mismo rol cruce información para fines no autorizados.

### 3. ¿Por qué el DBA no debería decidir por sí solo la finalidad del tratamiento de los datos?
El Administrador de Base de Datos (DBA) es el custodio técnico de la infraestructura, no el propietario del dato (Data Owner) ni el evaluador legal de la organización.

Permitir que el DBA decida unilateralmente la finalidad genera graves riesgos porque:

Falta de contexto normativo y legal: El DBA no cuenta con la atribución ni la formación jurídica para interpretar leyes de privacidad (como LFPDPPP o GDPR) ni para estructurar el Avisos de Privacidad.

Falta de alineación con el negocio: El DBA desconoce los acuerdos contractuales, estrategias comerciales o compromisos asumidos con los clientes.

Conflicto de interés operativo: Un DBA priorizará la practicidad técnica o el rendimiento sobre las restricciones de privacidad, lo que puede derivar en la creación de tablas o vistas que vulneren la ley por "eficiencia".

### 4. ¿Por qué los desarrolladores no deberían recibir automáticamente copias completas de bases de datos productivas?
Entregar volcados (dumps) completos de producción a entornos de desarrollo viola los principios fundamentales de seguridad y privacidad por las siguientes razones:

Diferencial de seguridad en entornos: Los entornos de desarrollo y pruebas (testing) carecen de los mismos niveles de monitoreo, controles de acceso, cifrado y bitácoras que el entorno de producción.

Dispersión irrestricta de PII: Copiar la base de datos duplica la superficie de ataque. Si un desarrollador almacena la copia en su computadora portátil o en un servidor local sin cifrar, un robo físico o malware expondrá la totalidad de los datos reales de los clientes.

Falta de necesidad funcional: Para escribir código, depurar o realizar pruebas unitarias, los desarrolladores no necesitan conocer el nombre o RFC real de una persona; requieren únicamente la estructura DDL y tipos de datos válidos.

### 5. ¿Por qué una copia utilizada para desarrollo también puede representar un riesgo de privacidad?
Una copia en desarrollo representa exactamente el mismo riesgo legal y reputacional que una filtración en el entorno productivo principal:

Sanciones legales equivalentes: Ante una autoridad reguladora, la exposición no autorizada de PII es una violación de confidencialidad, independientemente de si los datos se filtraron desde un servidor de producción o desde la laptop de un programador.

Riesgo de fuga en repositorios de código: Es común que durante las pruebas los desarrolladores incluyan respaldos o extractos de datos en scripts que por accidente se suben a repositorios públicos (GitHub/GitLab), exponiendo credenciales y PII a internet.

Visibilidad por terceros y proveedores: En el desarrollo suelen participar consultores externos o contratistas temporalmente autorizados que no deberían tener acceso a la información confidencial de la empresa.
### 6. ¿Por qué la retención debe formar parte del ciclo de vida de los datos?
El ciclo de vida del dato comprende desde su recolección hasta su destrucción o anonimización definitiva. La retención es una fase crítica por los siguientes motivos:

Principio de Limitación del Plazo de Conservación: Los datos solo deben guardarse mientras exista una relación contractual activa o un mandato legal que lo exija (ej. leyes fiscales que exigen conservar registros por 5 años).

Eliminación de "Datos Zombi": Mantener información de clientes inactivos de hace 10 o 15 años sin justificación incrementa innecesariamente los costos de almacenamiento y el impacto de una brecha de seguridad.

Cumplimiento de Cancelación (Derechos ARCO): La política de retención establece el "periodo de bloqueo" previo a la purga segura, garantizando que el dato no esté disponible operativamente pero se conserve aislado para responder ante posibles requerimientos judiciales.
### 7. ¿Qué relación existe entre privacidad por diseño y diseño de bases de datos?
Privacidad por Diseño (Privacy by Design) exige que los principios de privacidad se integren proactivamente en el diseño de los sistemas desde la fase conceptual, no como un parche posterior.

En el diseño de bases de datos, esto se traduce directamente en la estructura DDL y la arquitectura:

Normalización y Segregación: Separar datos operativos básicos de datos fiscales y sensibles en tablas o esquemas independientes.

Modelado Mínimo: Evitar la inclusión de columnas redundantes o innecesarias (salario, estado_civil) durante la creación de tablas.

Definición de Vistas y RBAC desde el inicio: Crear el esquema garantizando que las aplicaciones y roles se conecten a vistas restringidas y no a las tablas base.

Automatización de Auditoría y Purga: Diseñar triggers y tareas programadas (cron jobs) para el registro de cambios (pg_audit) y la supresión automática de datos vencidos.

### 8. ¿Qué controles técnicos implementarías para reducir el riesgo de exposición de datos personales?
Para mitigar la exposición de PII en una base de datos PostgreSQL 16, se deben aplicar los siguientes controles en capas (Defensa en Profundidad):

Control de Acceso Basado en Roles (RBAC): Revocar privilegios directos (REVOKE ALL ON clientes FROM PUBLIC) y canalizar el acceso mediante roles restringidos.

Cifrado en Reposo y en Tránsito: Implementar TLS/SSL para conexiones de red y utilizar funciones criptográficas (pgcrypto) para cifrar columnas con datos altamente sensibles (rfc).

Enmascaramiento de Datos (Data Masking): Utilizar vistas con enmascaramiento dinámico para presentar datos parcialmente ocultos (ej. ****-****-1234 o j***@email.com) a roles de soporte y atención.

Seguridad a Nivel de Fila (RLS - Row Level Security): Aplicar políticas RLS (CREATE POLICY) para que un usuario comercial solo pueda consultar los registros pertenecientes a su jurisdicción o catálogo autorizado.

Bitácoras de Auditoría Extensivas (pg_audit): Activar extensiones de auditoría para registrar cada intento de consulta, modificación o eliminación sobre tablas con PII.

Datos Sintéticos en Desarrollo: Implementar herramientas de generación de datos falsos/sintéticos para que los entornos de pruebas no contengan ningún registro de producción.
### 9. ¿Qué información debería conocer un DBA para responder técnicamente a una solicitud ARCO?
Para ejecutar de forma precisa una solicitud de derechos ARCO en la base de datos, el DBA debe recibir formalmente del Oficial de Privacidad la siguiente información:

Acceso: ID único e inconfundible del cliente (id_cliente) para extraer un reporte estandarizado de todos los registros asociados en tablas operativas y de soporte.

Rectificación: ID único e inconfundible del cliente (id_cliente) para extraer un reporte estandarizado de todos los registros asociados en tablas operativas y de soporte.

Rectificación: Atributo exacto a modificar, valor nuevo validado y el identificador del ticket administrativo que autoriza la corrección.

Cancelación: ID del usuario, fecha de vencimiento de la relación contractual, verificación de obligaciones fiscales pendientes y el parámetro para aplicar bloqueo técnico (marcado de inactividad) o purga física.

Oposición: La finalidad específica a la que el cliente se opone (ej. marketing), para cambiar la bandera booleana (permite_marketing = FALSE) o actualizar las vistas comerciales.

### 10. ¿Qué responsabilidades deberían corresponder al DBA, al responsable de privacidad, al área jurídica y al propietario de los datos?

La gobernanza eficiente requiere una delimitación clara de responsabilidades bajo una matriz de asignación de funciones (RACI):
```text
[ Área Jurídica ] ──> Define licitud, contratos y Avisos de Privacidad.
        │
[ Resp. de Privacidad ] ──> Gestiona solicitudes ARCO, auditorías y políticas.
        │
[ Propietario de Datos ] ──> Define valor del dato, permisos y necesidades de negocio.
        │
[ DBA (Custodio Técnico) ] ──> Executa controles SQL, RBAC, cifrado y purga en PostgreSQL.
```

1. Área Jurídica
Interpretación de las leyes y normativas de protección de datos aplicables.

Redacción y actualización de los Avisos de Privacidad institucionales y términos contractuales.

Evaluación de riesgos legales ante transferencias de datos o incidentes de seguridad.

2. Responsable de Privacidad (Oficial de Privacidad / DPO)
Definición del Programa Institucional de Privacidad y atención del canal oficial de solicitudes ARCO.

Supervisión del cumplimiento de los periodos de retención y consentimiento.

Enlace entre los titulares de los datos, la alta dirección y la autoridad reguladora (INAI).

3. Propietario de los Datos (Data Owner / Área de Negocio)
Definición de la finalidad comercial y operativa de los datos que recolecta su área.

Clasificación inicial del nivel de sensibilidad de la información de negocio.

Autorización formal de qué roles o departamentos requieren acceso a qué atributos.

4. Administrador de la Base de Datos (DBA / Custodio Técnico)
Implementación técnica de los controles: Configuración del esquema DDL, vistas, extensiones (pg_audit, pgcrypto) y políticas de roles (RBAC) en el motor PostgreSQL.

Execution de las operaciones físicas de bloqueo, purga o anonimización ordenadas por el Responsable de Privacidad.

Mantenimiento de la seguridad de la infraestructura, respaldos seguros, parches de software y monitoreo de la disponibilidad del sistema.