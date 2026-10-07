# Reto ENSEÑA · Plataforma de Hackathons Open-Source

> Proyecto Fin de Grado · Grado en Ingeniería Informática · Universidad Europea de Madrid · Curso 2025-2026
> Desarrollado en colaboración con **Oracle Iberia** en el marco del programa Reto ENSEÑA.

**Autores:** Jorge García Pedraza y Costin Dragos Dogaru
**Director académico:** Jorge Saldaña Alegre
**Codirectores (Oracle):** Óscar Robles Pereira y Carolina Díaz Serna

---

## Qué es

Reto ENSEÑA es una plataforma web integral para gestionar hackathons universitarios de principio a fin, construida íntegramente sobre Oracle Cloud Infrastructure con **Oracle APEX 24.2** como capa de presentación low-code y **Oracle Autonomous Database** como capa de datos. Sustituye el conjunto habitual de formularios, hojas de cálculo y correos por una única aplicación con trazabilidad completa.

Cubre el ciclo completo de un hackathon:

- Registro y autenticación con tres roles diferenciados: **participante**, **tutor** y **organizador**.
- Formación de equipos con invitación por enlace, código QR o token.
- Retos divididos en fases, cada una con enunciado, puntuación máxima y tipo de corrección (manual, automática por token tipo CTF, o mixta).
- Entregas con texto y archivos adjuntos.
- Evaluación por rúbrica de criterios con feedback del tutor.
- Clasificación automática mediante un modelo de puntuación top-down (reto = 100 puntos, repartidos por peso entre fases y criterios, mantenido por triggers).
- Gamificación con sistema de pistas que aplican penalización configurable.
- Recuperación de contraseña por correo sobre infraestructura SMTP propia (OCI Email Delivery).
- Perfil de usuario con avatar, biografía, enlaces y modo claro/oscuro persistido en base de datos.
- Panel de administración y exportación CSV.

### Inteligencia artificial integrada

- **OCI Language (NLP):** cada entrega se analiza automáticamente para extraer resumen, frases clave, entidades y sentimiento, como apoyo al tutor en la evaluación.
- **Select AI + OCI Generative AI (Cohere Command-R-Plus):** asistente conversacional que traduce preguntas en español a SQL ejecutable sobre el esquema. Existen dos funciones según el rol:
  - `RETO_AI_ASK` (tutor/organizador): asistente analítico, genera y ejecuta SQL con acceso restringido mediante `object_list`.
  - `RETO_AI_ASK_PARTICIPANTE` (participante): asistente de ayuda en modo `chat`, no genera ni ejecuta SQL, solo orienta sobre el uso de la plataforma. Código en [`RETO_AI_ASK_PARTICIPANTE.sql`](RETO_AI_ASK_PARTICIPANTE.sql).

## Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│  Oracle APEX 24.2  (36 páginas, tema Reto ENSEÑA Pro)       │  Presentación
├─────────────────────────────────────────────────────────────┤
│  PL/SQL: pkg_auth · pkg_email · pkg_password_reset          │  Lógica
│          analyze_phase_submission · RETO_AI_ASK · triggers  │
├─────────────────────────────────────────────────────────────┤
│  Oracle Autonomous Database 26ai  (22 tablas)               │  Datos
├─────────────────────────────────────────────────────────────┤
│  OCI Language  ·  OCI Generative AI (Select AI)  ·  OCI     │  Servicios
│  Email Delivery  ·  Resource Principal                      │  cloud
└─────────────────────────────────────────────────────────────┘
```

| Componente | Valor |
|---|---|
| Región OCI | `uk-london-1` (necesaria para Generative AI en Always Free) |
| Workspace APEX | `WSK_RETOENSENA` |
| Esquema | `WKSP_WSKRETOENSENA` |
| ID aplicación APEX | `290419` |
| Perfil Select AI | `RETO_ENSENA_AI` |
| Modelo LLM | `cohere.command-r-plus-08-2024` |
| Coste | Diseñado para funcionar dentro de los límites Always Free |

La arquitectura completa (cinco vistas), el modelo de datos, el catálogo de tablas y las decisiones de diseño están en la memoria, capítulo 4.2 y Anexo B.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| [`deploy_retoensena.sql`](deploy_retoensena.sql) | Script de despliegue todo-en-uno: crea las 22 tablas, el paquete `pkg_auth` completo, las especificaciones de `pkg_email` y `pkg_password_reset`, los catálogos básicos y dos usuarios por defecto. |
| [`RETO_AI_ASK_PARTICIPANTE.sql`](RETO_AI_ASK_PARTICIPANTE.sql) | Función PL/SQL del asistente de ayuda para participantes (Select AI en modo `chat`). |
| [`Memoria_TFG_RetoENSENA_Entrega_Final.pdf`](Memoria_TFG_RetoENSENA_Entrega_Final.pdf) | Memoria completa del TFG (112 páginas). Contiene el manual de despliegue, el manual de usuario, los casos de uso y toda la documentación técnica. |

> **Nota:** el export de la aplicación APEX (`f290419.sql`) y los cuerpos completos de `pkg_email` y `pkg_password_reset` forman parte del paquete de entregables del TFG y no están incluidos en este repositorio de momento. El script `deploy_retoensena.sql` deja la base de datos lista; la aplicación APEX debe importarse por separado.

## Despliegue

Guía resumida. El procedimiento detallado, con los bloques PL/SQL completos para cada paso, está en el **Anexo A de la memoria (páginas 79 a 83)**.

### Requisitos previos

- Cuenta de Oracle Cloud Infrastructure con Always Free disponible (o suscripción activa).
- Permisos en la tenencia para crear compartimentos, políticas IAM, usuarios y grupos.
- Export de la aplicación APEX `f290419.sql`.
- Opcional: un dominio propio con acceso a DNS, solo si se quiere habilitar el envío de correo.

### Pasos

**1. Provisionar la Autonomous Database** (Anexo A.2)
Desde la consola OCI, en `Oracle Database > Autonomous Database`, crear una base en la región `uk-london-1` con workload *Data Warehouse*, infraestructura compartida, Always Free activado y versión 26ai o superior.

**2. Crear el workspace APEX** (Anexo A.3)
Desde `Tool configuration > APEX > Open APEX` (Internal Workspace), crear el workspace `WSK_RETOENSENA` asociado a un esquema nuevo `WKSP_WSKRETOENSENA`.

**3. Desplegar el esquema** (Anexo A.4)
Abrir SQL Developer Web conectado como `WKSP_WSKRETOENSENA` y ejecutar [`deploy_retoensena.sql`](deploy_retoensena.sql) completo. Tarda entre 1 y 2 minutos y deja creados:

- Las 22 tablas del esquema con sus índices y constraints.
- El paquete `pkg_auth` (registro, hash con sal por usuario, autenticación).
- Los catálogos de roles, universidades y categorías de reto.
- Dos usuarios por defecto:

  | Rol | Email | Contraseña |
  |---|---|---|
  | Organizador | `organizador@retoensena.local` | `Demo1234!` |
  | Tutor | `tutor@retoensena.local` | `Demo1234!` |

  Cambiar estas contraseñas en el primer acceso desde *Mi Perfil*.

A continuación, ejecutar [`RETO_AI_ASK_PARTICIPANTE.sql`](RETO_AI_ASK_PARTICIPANTE.sql) en la misma conexión.

**4. Importar la aplicación APEX** (Anexo A.5)
Desde el workspace, `App Builder > Import`, importar `f290419.sql` manteniendo el ID 290419 si está libre. Revisar después los esquemas de autorización y las credenciales referenciadas en los procedimientos PL/SQL.

**5. Habilitar Resource Principal** (Anexo A.6 y Anexo C)
Necesario para que la base de datos invoque OCI Generative AI sin credenciales explícitas. Como `ADMIN`:

```sql
BEGIN DBMS_CLOUD_ADMIN.ENABLE_RESOURCE_PRINCIPAL(); END;
/
GRANT EXECUTE ON DBMS_CLOUD_AI TO WKSP_WSKRETOENSENA;
```

Añadir la ACL de red para `inference.generativeai.uk-london-1.oci.oraclecloud.com` y crear en IAM la política que permite al recurso usar `generative-ai-family` (bloques completos en el Anexo A.6).

**6. Crear el perfil Select AI** (Anexo A.7)
Conectado como `WKSP_WSKRETOENSENA`, crear el perfil `RETO_ENSENA_AI` con proveedor `oci`, credencial `OCI$RESOURCE_PRINCIPAL`, modelo `cohere.command-r-plus-08-2024` y un `object_list` limitado a `USERS`, `TEAMS`, `CHALLENGES`, `CHALLENGE_PHASES` y `PHASE_SUBMISSIONS`.

**7. Credencial de OCI Language** (Anexo A.8)
Crear la credencial `OCI_LANG_CRED` con `DBMS_CLOUD.CREATE_CREDENTIAL` (user OCID, tenancy OCID, clave privada y fingerprint) y abrir la ACL para `language.aiservice.uk-london-1.oci.oraclecloud.com`.

**8. OCI Email Delivery** (Anexo A.9, opcional)
Solo si se quiere recuperación de contraseña por correo: configurar el dominio con DKIM y SPF, aprobar el remitente `noreply@<dominio>`, crear el grupo `email-senders` con un usuario técnico y sus credenciales SMTP, y apuntar APEX (`Workspace Administration > Manage Instance > Email`) al endpoint `smtp.email.uk-london-1.oci.oraclecloud.com:587`.

**9. Validar** (Anexo A.10)
Acceder a la URL de la aplicación, registrar un usuario, crear un equipo e invitar por token, crear un reto con dos fases, enviar una entrega y comprobar que aparece su análisis en `AI_PHASE_ANALYSES`, hacer una pregunta al asistente IA y verificar el registro en `AI_QUERIES`, y probar la recuperación de contraseña si se configuró el correo.

## Cómo funciona (visión rápida por rol)

**Participante**
Se registra, crea o se une a un equipo, se inscribe en un reto, consulta las fases, entrega (texto + adjuntos), puede revelar pistas asumiendo la penalización, y recibe feedback y puntuación. Tiene un asistente de ayuda que explica cómo usar la plataforma sin acceder a datos.

**Tutor**
Ve los retos a los que está asignado, revisa las entregas de sus equipos con el análisis NLP ya generado (resumen, frases clave, entidades, sentimiento), puntúa por criterios y deja feedback. Puede consultar datos en lenguaje natural mediante el asistente analítico.

**Organizador**
Crea y configura retos, fases, criterios de evaluación y pistas; asigna tutores; gestiona usuarios y equipos; consulta estadísticas y la clasificación; exporta a CSV.

## Manual de usuario

El manual de usuario completo, organizado por rol y con capturas de pantalla, está en la memoria:

| Sección | Páginas |
|---|---|
| Anexo I · Manual de usuario | 91 a 112 |
| I.1 Manual para participantes | 91 |
| I.2 Manual para tutores | 101 |
| I.3 Manual para organizadores | 105 |
| I.4 Flujos transversales (recuperación de contraseña, perfil, tema) | 109 |

Los siete casos de uso principales con sus flujos paso a paso (registro, equipos, inscripción, entrega con NLP, evaluación con IA, asistente, recuperación de contraseña) están en el **Anexo G, páginas 86 a 89**.

## Dónde encontrar cada cosa en la memoria

| Tema | Capítulo / Anexo |
|---|---|
| Objetivos y alcance | Cap. 3 (pág. 23) |
| Metodología, sprints y reparto de trabajo | Cap. 4.1 (pág. 28) |
| Arquitectura en tres capas y vistas detalladas | Cap. 4.2.2 y 4.2.3 (pág. 34) |
| Modelo de datos y catálogo de las 22 tablas | Cap. 4.2.5 a 4.2.7 (pág. 41) |
| Implementación funcional | Cap. 4.2.8 (pág. 45) |
| Seguridad y RGPD | Cap. 4.2.9 (pág. 51) |
| Retos técnicos resueltos | Cap. 4.2.10 (pág. 52) |
| Presupuesto y coste real en OCI | Cap. 4.4 (pág. 55) |
| Manual de despliegue paso a paso | Anexo A (pág. 79) |
| Reglas de decisión del modelo de datos | Anexo B (pág. 83) |
| Resource Principal (referencia rápida) | Anexo C (pág. 84) |
| Glosario | Anexo D (pág. 84) |
| Casos de uso detallados | Anexo G (pág. 86) |
| Manual de usuario | Anexo I (pág. 91) |

## Licencia

Proyecto académico de código abierto desarrollado como Trabajo de Fin de Grado. Los nombres Oracle, APEX, OCI y Reto ENSEÑA son marcas de sus respectivos titulares.
