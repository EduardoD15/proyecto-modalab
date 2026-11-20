# Modalab — prototipo de autenticación y roles
- Nota El informe se puso en otro md llamado INFORME.MD o se puede encontrar tambien en este readme a la final del documento 

Prototipo académico del módulo administrativo acordado en los talleres: **F01 inicio de sesión, F02 autenticación y F03 control de acceso por roles**. No implementa los módulos de productos, ventas, entregas o reportes y no está preparado para desplegarse en producción.

## Requisitos

- Windows 10/11 y PowerShell.
- Node.js 22.18 o superior; npm se instala con Node.js.
- Chromium de Playwright para las pruebas de navegador A05.
- k6 solo si se ejecutará la prueba de carga A06.

SQLite se guarda en archivos locales del proyecto; no requiere una instalación aparte.

### Instalar k6 (solo para A06)

En Windows, instala k6 desde PowerShell:

```powershell
winget install k6 --source winget
```

Al terminar, cierra y vuelve a abrir la terminal y verifica la instalación:

```powershell
k6 version
```

Las demás pruebas no necesitan k6.

## Preparación inicial

Ejecuta estos comandos desde PowerShell, en la carpeta `proyecto-modalab`:

```powershell
node --version
npm --version
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
npm ci
```

El proyecto requiere Node.js 22.18 o superior. Si acabas de crear `.env`, revisa que las variables `MODALAB_ADMIN_PASSWORD` y `MODALAB_SELLER_PASSWORD` tengan contraseñas sintéticas para pruebas. No publiques `.env` ni uses estas cuentas en un despliegue accesible desde internet.

## Ejecutar la aplicación

Modo desarrollo:

```powershell
npm run dev
```

Abre `http://localhost:3000`. Las cuentas sintéticas son:

| Rol | Usuario |
|---|---|
| Administrador | `admin@modalab.test` |
| Vendedor | `vendedor@modalab.test` |
| Cuenta inactiva para pruebas negativas | `inactivo@modalab.test` |

Las contraseñas del Administrador y Vendedor se configuran en `.env`. La base de datos local se crea en `data/modalab.sqlite`.

Para comprobar la versión compilada en local, detén el modo desarrollo con `Ctrl+C` y ejecuta:

```powershell
npm run build
npm start
```

Abre `http://localhost:3000`. `npm start` requiere que `npm run build` haya terminado correctamente.

## Ejecutar las pruebas automatizadas

Los resultados aparecen en la terminal. `✔` significa que un caso pasó; el resumen indica cuántos pasaron y fallaron.

### Tecnologías utilizadas en las pruebas y por qué

- **Node.js con `node:test` y `tsx`:** permite ejecutar las pruebas unitarias y de integración directamente con el entorno del proyecto, sin necesidad de configurar otro lenguaje o ejecutor. `tsx` permite usar TypeScript en las pruebas y es una opción viable porque el proyecto ya utiliza TypeScript y Node.js.
- **SQLite temporal:** se usa en las pruebas unitarias y de integración para aislar los datos de prueba y verificar la lógica sin modificar la base de datos principal. Es viable porque la aplicación ya utiliza SQLite y no requiere instalar ni configurar un servidor de base de datos adicional.
- **Playwright:** se usa para las pruebas de navegador A05. Permite automatizar Chromium y comprobar el inicio de sesión, la interfaz y la autorización por rol desde la perspectiva de un usuario real. Fue una opción viable porque incluye una API automatizada para navegadores, permite preparar automáticamente la aplicación y conserva trazas cuando una prueba falla.
- **k6:** se usa para la prueba de carga A06 y permite enviar varias solicitudes concurrentes al inicio de sesión. Fue una opción viable porque es fácil de ejecutar desde PowerShell y Windows, permite definir umbrales de rendimiento claros y funciona bien para pruebas locales sin ocupar infraestructura adicional.

### A01 — Pruebas unitarias

```powershell
npm run test:unit
```

Comprueba validaciones de entrada y reglas de acceso por rol.

### A02–A04 — Pruebas de integración

```powershell
npm run test:integration
```

Comprueba bloqueo de cuentas, inicio de sesión, cuentas inactivas y ciclo de vida de sesiones usando una base SQLite temporal.

Para ejecutar A01–A04 en una sola orden:

```powershell
npm test
```

### A05 — Pruebas de navegador

Instala Chromium una vez:

```powershell
npx playwright install chromium
```

Después ejecuta:

```powershell
npm run test:e2e
```

Playwright prepara una base de datos separada y arranca la aplicación en `http://localhost:3100` durante la prueba. No necesitas iniciar `npm run dev` por separado. Los resultados se muestran en la terminal; las trazas se conservan cuando una prueba falla.

### A06 — Prueba de carga local con k6

Primero inicia la aplicación en una terminal:

```powershell
npm run dev
```

En otra terminal, configura el usuario de prueba. `LOGIN_PASSWORD` debe tener el mismo valor que `MODALAB_ADMIN_PASSWORD` en `.env`; k6 no lee `.env` automáticamente:

```powershell
$env:LOGIN_EMAIL = "admin@modalab.test"
$env:LOGIN_PASSWORD = "reemplaza por la contraseña de prueba del administrador"
npm run test:load
si falla npm usar el siguiente comando
& "C:\Program Files\k6\k6.exe" run tests/load/login-concurrency.k6.js
```

La prueba envía 50 solicitudes al endpoint local `http://127.0.0.1:3000/api/auth/login`. Solo acepta destinos `localhost` o `127.0.0.1`; no la ejecutes contra producción ni contra sistemas de terceros.

El resumen de k6 aparece en la terminal. Para aprobar, cada respuesta debe ser HTTP 200, los errores HTTP deben ser menores al 1% y el tiempo promedio debe ser menor a 2 segundos. Si las solicitudes responden correctamente pero el umbral de tiempo falla, la autenticación funcionó, pero no se alcanzó el objetivo de rendimiento en ese equipo.

### Ejecutar todas las pruebas automatizadas juntas

Después de instalar Chromium y k6, ejecuta desde la carpeta del proyecto:

```powershell
npm run test:all
```

Este comando ejecuta lint, A01–A04, A05, compila la aplicación y ejecuta A06 sobre una instancia temporal de producción. Usa bases de datos y resultados temporales para no reemplazar los archivos locales habituales. Si no encuentra k6 en el `PATH` ni en su ubicación usual de Windows, indica la ruta con `$env:K6_PATH = "C:\Program Files\k6\k6.exe"` antes de ejecutarlo.

El proceso termina con error si una prueba falla o si A06 no cumple el umbral de rendimiento. Las 17 pruebas manuales no se pueden ejecutar automáticamente: están descritas en [tests/manual/casos-de-prueba.md](tests/manual/casos-de-prueba.md) y requieren registrar resultados y evidencia. `npm test` sigue disponible para ejecutar solamente A01–A04.

## Orden recomendado

1. Instala dependencias y configura `.env` siguiendo la preparación inicial.
2. Ejecuta las pruebas automatizadas juntas con `npm run test:all`, o usa los comandos individuales de A01–A06 para revisar cada grupo.
3. Ejecuta los casos M01–M17 de forma manual y registra su evidencia.

## Casos automatizados

El diseño contempla 17 casos manuales y 6 automatizados (23 en total):

- **A01:** particiones y límites de entradas / roles — `tests/unit/auth-policy.test.ts`.
- **A02:** bloqueo al tercer intento y concurrencia de fallos — `tests/integration/auth-service.test.ts`; verificación E2E adicional en `tests/e2e/auth.spec.ts`.
- **A03:** cuenta inactiva — `tests/integration/auth-service.test.ts`.
- **A04:** sesión, expiración y logout — `tests/integration/auth-service.test.ts`.
- **A05:** login y autorización por rol en UI/API — `tests/e2e/auth.spec.ts`.
- **A06:** 50 solicitudes válidas concurrentes — `tests/load/login-concurrency.k6.js`.

Los 17 casos manuales, con técnicas, datos, pasos, resultados y trazabilidad, están en [tests/manual/casos-de-prueba.md](tests/manual/casos-de-prueba.md). Están diseñados y pendientes de ejecución manual y captura de evidencia.

## Rutas principales

| Ruta | Uso |
|---|---|
| `POST /api/auth/login` | Valida credenciales, aplica bloqueo temporal y crea cookie de sesión |
| `GET /api/auth/me` | Devuelve el usuario de la sesión activa |
| `POST /api/auth/logout` | Revoca la sesión y elimina la cookie |
| `GET /api/admin/users` | Lista datos básicos sintéticos; solo Administrador |

Las contraseñas se almacenan con scrypt; el identificador de sesión es aleatorio, se entrega en cookie `HttpOnly` y solo se conserva su hash en SQLite. El costo de scrypt está reducido para alcanzar el umbral académico en hardware local de cuatro núcleos; es una decisión exclusiva del prototipo y **no debe copiarse a producción**. El control de rol se verifica en la API, no solamente ocultando enlaces en la interfaz.

## Estructura

```text
src/app/                 Páginas Next.js y rutas API
src/lib/                 Persistencia SQLite, autenticación y política de roles
tests/unit/              Pruebas unitarias
tests/integration/       Pruebas de servicio con base temporal
tests/e2e/               Pruebas Playwright de navegador
tests/manual/            Catálogo de 17 casos manuales
tests/load/              Prueba k6 local de concurrencia
scripts/                 Preparación aislada de datos E2E
```

El plan completo y el documento Word están en la carpeta `../Solucion`.
La guía para distribuir nuevas contribuciones y commits por integrante está en `../Solucion/Guia_Colaboracion_Git.md`.

# Plan de pruebas de software — Modalab (INFORME)

**Proyecto:** Aplicativo administrativo de una tienda de ropa  
**Módulo evaluado:** Inicio de sesión, autenticación y control de acceso  
**Norma de referencia:** ISO/IEC/IEEE 29119-3:2013  
**Versión del documento:** 1.1  
**Fecha:** 18 de noviembre de 2026  
**Autores identificados en los antecedentes:** Eduardo Deossa, Estefanía Montoya y Santiago Castaño  
**Estado de aprobación:** Borrador académico pendiente de revisión del equipo y aprobación docente  

---

## 1. Información específica del documento 

### 1.1 Identificador
MOD-PT-001 — Plan de pruebas del módulo administrativo de Modalab.

### 1.2 Organización emisora y autores
Trabajo académico del equipo de Modalab, desarrollado para la asignatura de Ingeniería de Software / Pruebas de Software.

| Integrante | Responsabilidad propuesta |
| :--- | :--- |
| **Eduardo Deossa** | Aplicación de inicio de sesión y configuración básica del proyecto |
| **Estefanía Montoya** | Alcance, riesgos, casos manuales y documentos del plan |
| **Santiago Castaño** | Pruebas automatizadas, ejecución y guía de comandos |

### 1.4 Historial de cambios

| Versión | Fecha | Cambio | Autoría |
| :--- | :--- | :--- | :--- |
| **0.1** | 09/11/2026 | Inicio del periodo del proyecto propio de inicio de sesión y revisión del prototipo existente del equipo | Equipo Modalab |
| **1.0** | 17/11/2026 | Cierre del periodo de trabajo; consolidación de alcance, riesgos, casos, ejecución y estimaciones | Equipo Modalab |
| **1.1** | 18/11/2026 | Actualización de autores, cronograma, estado de A06 y propuesta para separar aportes en una copia nueva | Equipo Modalab |

---

## 2. Introducción y declaración de adaptación

### 2.1 Alcance del documento
El documento habla sobre las las pruebas realizadas al modulo administrativo del aplicativo:
* **F01 — Inicio de sesión:** presentación y validación del formulario de acceso.
* **F02 — Autenticación:** validación de credenciales, estado de cuenta, manejo de intentos fallidos y sesión.
* **F03 — Control de acceso:** autorización de funciones según el rol Administrador o Vendedor.

Las pruebas anteriores se realizaran en el modulo de inicio de sesion, ninguna de estas utilizara los modulos de productos o gestion de estos.

### 2.2 Referencias
1. `../Actividad/contenidoPlan.txt` — estructura requerida para el informe.
2. `../Actividad/Plan_de_Pruebas_de_Software_ISO_29119-3.pdf` — guía de la actividad final.
3. `../ContextoTrabajosEchoAntes/trabajo.md` — alcance, talleres, riesgos y casos de prueba ya trabajados por el equipo.
4. `../proyecto-modalab/README.md` — proyecto de inicio de sesión usado como sistema bajo prueba.
5. Material de clase sobre fundamentos, diseño y documentación de pruebas.
6. ISO/IEC/IEEE 29119-3:2013, *Software and systems engineering — Software testing — Part 3: Test documentation*.
7. ISO/IEC/IEEE 29119-4, como referencia para técnicas de diseño de pruebas.
8. `Guia_Colaboracion_Git.md` — archivos y pasos para organizar los aportes en una copia nueva del proyecto.

### 2.3 Conformidad y adaptación
La conformidad se basa en una matriz de riesgos y condiciones de 23 casos de pruebas, procedimientos, datos/entorno, criterios de salida y estimaciones en la ultima version utilizable de la aplicacion (`2f3b054`) en una copia nueva del proyecto para evitar cambios en ese historial.

Las pruebas fueron ejecutadas en un entorno local con datos sinteticos, estas solo se realizaron para comprobar que los datos y pruebas funcionaran correctamente mas no para ver la capacidad del aplicativo en el entorno de produccion.

### 2.4 Glosario

| Término | Definición usada en este plan |
| :--- | :--- |
| **Caso de prueba (TC)** | Condiciones, datos, acciones y resultados esperados para una verificación reproducible |
| **Condición de prueba (CP)** | Regla o comportamiento verificable derivado de una característica y un riesgo |
| **RBAC** | Control de acceso basado en roles |
| **P1 / P2 / P3** | Prioridad alta / media / baja de ejecución |
| **Severidad** | Impacto de un defecto sobre seguridad, operación o comprensión |
| **Exposición** | Producto de probabilidad por impacto, en una escala de 1 a 9 |
| **Retest** | Repetición del caso que reveló un defecto, después de su corrección |
| **Regresión** | Ejecución de pruebas existentes para detectar efectos colaterales de cambios |

---

## 3. Contexto de las pruebas

### 3.1 Elementos de prueba

| ID | Elemento | Actor principal | En alcance | Justificación |
| :--- | :--- | :--- | :--- | :--- |
| **F01** | Inicio de sesión | Administrador o Vendedor | Sí | Punto de entrada del módulo y origen de las sesiones |
| **F02** | Autenticación y estado de cuenta | Servicio de autenticación | Sí | Impide el acceso con credenciales inválidas o cuentas inactivas/bloqueadas |
| **F03** | Autorización por roles | Administrador y Vendedor | Sí | Evita que un rol acceda a funciones no autorizadas |
| **F04** | Gestión de productos | Administrador | No | Está fuera del módulo de autenticación priorizado |
| **F05** | Ventas y créditos | Vendedor / Administrador | No | Flujo operativo no implementado en el prototipo |
| **F06** | Entregas y envíos | Personal de tienda | No | Fuera del alcance académico definido |
| **F07** | Reportes y analítica | Administrador | No | Fuera del alcance académico definido |

### 3.2 Alcance y exclusiones de las pruebas
* **Incluido:** validación de entradas; credenciales correctas e incorrectas; cuentas activas e inactivas; bloqueo al tercer intento fallido consecutivo; persistencia y cierre de sesión; rutas y API protegidas por rol; casos negativos y expiración de sesión; pruebas unitarias, de integración/API, de interfaz y carga local acotada.
* **Excluido:** altas y recuperación de usuarios, cambio de contraseña, segundo factor, federación/SSO, integración con sistemas reales, gestión funcional de tienda, auditoría formal, compatibilidad exhaustiva de dispositivos, pruebas de penetración y cualquier carga contra un entorno no local.

### 3.3 Suposiciones y restricciones
* Se cuenta con una versión del prototipo y un entorno local controlado con Node.js 22, Next.js, SQLite y navegador Chromium.
* Las identidades, roles y contraseñas de ensayo son ficticios; no se usan datos personales ni credenciales reales.
* Los roles del alcance son Administrador y Vendedor. La matriz final de permisos de negocio debe ser confirmada por el equipo.
* El equipo reportado para este proyecto está compuesto por Eduardo Deossa, Estefanía Montoya y Santiago Castaño; la distribución de aportes está descrita en 1.2 y 7.
* Regla de bloqueo adoptada del taller: el tercer intento fallido consecutivo bloquea temporalmente la cuenta; los intentos primero y segundo no la bloquean. La duración exacta del bloqueo es una configuración del prototipo y debe confirmarse antes de fijar la prueba temporal.
* Los resultados de rendimiento dependen del equipo, sistema operativo y carga concurrente; deben registrarse junto con la medición y no generalizarse a producción.
* Las estimaciones son esfuerzo-persona, no duración calendario garantizada.

### 3.4 Partes interesadas

| Parte interesada | Interés / responsabilidad |
| :--- | :--- |
| **Docente** | Revisa la alineación con la actividad y aprueba el entregable |
| **Equipo Modalab** | Define alcance y reglas, prepara datos, ejecuta y reporta pruebas |
| **Administrador de tienda (actor)** | Accede a las funciones administrativas autorizadas |
| **Vendedor (actor)** | Usa funciones asignadas a su rol y no debe elevar privilegios |
| **Responsable de desarrollo** | Corrige defectos y entrega una versión identificable para retest |

---

## 4. Gestión de riesgos y matriz de exposición

### 4.1 Escala y método
Probabilidad (P) e impacto (I) se califican de 1 a 3. La exposición se calcula como $E = P \times I$: 1–2 baja, 3–4 baja, 6 media y 9 alta. El equipo tratará obligatoriamente los riesgos de exposición alta con pruebas priorizadas. La evaluación es cualitativa y deberá revisarse si cambian alcance, diseño o entorno.

### 4.2 Matriz de riesgos

| ID | Tipo | Riesgo | P | I | E / nivel | Tratamiento y pruebas | Riesgo residual |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **RP-01** | Producto | El endpoint de autenticación se degrada bajo concurrencia y produce errores o demoras | 3 | 3 | 9 / Alto | CP-04 / A06, carga local de 50 VU; medir latencia y errores | Escenarios, hardware y volúmenes no ensayados |
| **RP-02** | Producto | Fallos en los flujos principales de ingreso, sesión o salida | 2 | 3 | 6 / Medio | CP-02/03/06; M07–M12 y A02–A04 | Casos no contemplados y cambios posteriores |
| **RP-03** | Producto | Una cuenta sin autorización obtiene acceso a funciones de otro rol | 2 | 3 | 6 / Medio | CP-05; M13–M14 y A05, UI y API | Errores de configuración o amenazas no simuladas |
| **RP-04** | Producto | Entradas inválidas o sesiones expiradas generan errores no controlados | 3 | 2 | 6 / Medio | CP-01/06; M03–M07, M15–M16 y A01/A04 | Combinaciones de entrada no incluidas |
| **RY-01** | Proyecto | El entorno de prueba difiere del entorno objetivo | 3 | 3 | 9 / Alto | CP-07 / M01, preparación local reproducible con Node.js/npm y comprobación de datos/versiones | Diferencias con infraestructura productiva inexistente en el prototipo |
| **RY-02** | Proyecto | Retrasos de desarrollo reducen cobertura y tiempo de retest | 3 | 3 | 9 / Alto | CP-08 / M02, hitos de entrada/salida, dependencias WBS y priorización | Retraso extremo puede limitar alcance |
| **RY-03** | Proyecto | Conocimiento técnico desigual retrasa diseño o ejecución | 2 | 2 | 4 / Bajo | CP-09 / M17, glosario, instrucciones y checklist | Curva de aprendizaje y errores humanos |
| **RY-04** | Proyecto | Cambios no controlados amplían el alcance y obligan a rediseñar pruebas | 3 | 2 | 6 / Medio | CP-08 / M02, control de versión y actualización de trazabilidad | Cambios tardíos o no comunicados |

### 4.3 Tratamiento
Se tratan como prioritarios los Casos RP-01, RY-01 y RY-02, el caso A06 ayuda a medir la concurrencia en localhost y no debe ser tomado en cuenta a gran escala para producción los casos M01 y M02 comprueban el entorno reproducible y verifica el control de versión, mientras el el RY-03 se acepta con mitigación documental de bajo costo.

---

## 5. Estrategia de pruebas y derivación de casos

### 5.1 Condiciones de prueba derivadas

| CP | Característica / riesgo | Condición verificable |
| :--- | :--- | :--- |
| **CP-01** | F01/F02 — RP-04 | Las clases válidas e inválidas de correo/contraseña y sus límites se validan en cliente y API |
| **CP-02** | F02 — RP-02 | Credenciales correctas/incorrectas y estado activo/inactivo producen acceso o denegación esperados |
| **CP-03** | F02 — RP-02 | Los fallos consecutivos producen transición Activo → Bloqueado al intento 3 |
| **CP-04** | F01/F02 — RP-01 | Cincuenta solicitudes válidas concurrentes al endpoint local terminan con 100% HTTP 200 y promedio $\le 2\text{ s}$ |
| **CP-05** | F03 — RP-03 | Solo los roles autorizados pueden consultar cada ruta y endpoint |
| **CP-06** | F02 — RP-02/RP-04 | La sesión se crea, persiste, expira y revoca conforme a sus estados |
| **CP-07** | AMB-01 — RY-01 | La versión, el entorno y los datos sintéticos pueden prepararse y repetirse de forma segura |
| **CP-08** | Proyecto — RY-02/RY-04 | Versión, cambios, hitos y criterios de entrada/salida quedan identificados antes de ejecutar |
| **CP-09** | F01/F02/F03 — RY-03 | Mensajes, instrucciones y permisos visibles son comprensibles y coherentes con la guía del equipo |

### 5.2 Técnicas de diseño
* **Valores límite:** intentos fallidos 2/3/4; volumen de 50 solicitudes concurrentes; límites de longitud y campos vacíos.
* **Tabla de decisión:** combinación de credenciales correctas/incorrectas, cuenta activa/inactiva y rol Administrador/Vendedor.
* **Transición de estados:** no autenticado → autenticado; no autenticado → bloqueado; autenticado → sesión cerrada/expirada.
* **Partición de equivalencia:** correo válido/ inválido, contraseña presente/ausente, rol permitido/no permitido.
* **Checklist y exploración guiada:** coherencia de mensajes, navegación y permisos visibles.
* En el catálogo de 23 casos se indica por caso la técnica, la regla de generación de datos y el valor esperado; no se usan combinaciones intuitivas sin fundamento.

### 5.3 Casos de prueba y trazabilidad bidireccional
Se diseñaron 23 casos de pruebas, siendo 23 manuales (M01-M17) y 6 automáticas (A01-06).

| Riesgo | Condición | Casos que lo cubren |
| :--- | :--- | :--- |
| **RP-01** | CP-04 | A06 |
| **RP-02** | CP-02, CP-03, CP-06 | M07–M12; A02–A04 |
| **RP-03** | CP-05 | M13–M14; A05 |
| **RP-04** | CP-01, CP-06 | M03–M07, M15–M16; A01, A04 |
| **RY-01** | CP-07 | M01 |
| **RY-02** | CP-08 | M02 |
| **RY-03** | CP-09 | M17 |
| **RY-04** | CP-08 | M02 |

Trazabilidad directa e inversa: cada riesgo de la matriz conduce a condiciones y casos; cada caso incluye sus IDs de condición y riesgo. RP-01, RY-01 y RY-02 tienen casos de mitigación. A01–A04 se ejecutan con `npm test`, A05 con Playwright, A06 con k6 y `npm run test:all` reúne las pruebas automatizadas. M01–M17 requieren ejecución y registro de evidencia por separado.

### 5.4 Estado de validación técnica del prototipo
La validación local realizada el 18/11/2026 pasaron por 2 pruebas unitarias y 4 de integración, se completaron todas las pruebas correctamente respondiendo el servidor de forma efectiva en tiempos promedios de 4 a 6 segundos, 2 segundos por encima del criterio propuesto, por lo cual se debe realizar una optimización del caso A06 y realizar las pruebas nuevamente.

### 5.5 Requisitos de entorno y datos
Se realizo en un entorno con Node.js 22.18, npm, Next.js SQLite asilada de produccion y Chromium con Playwright en un entorno local con datos sinteticos y no se conecta con los datos de produccion, los datos fueron cuentas de vendedores, inactivas, administradores y contraseñas ficticias. Antes de TC-01 y TC-05 se reinicia el contador de fallos y se limpia la sesión; al terminar se eliminan sesiones y resultados temporales. No se almacenan contraseñas en texto claro ni se introducen datos reales.

### 5.6 Criterios de suspensión y finalización
* **Suspender si:** el entorno apunta fuera de localhost, hay riesgo de borrar datos no sintéticos, la versión bajo prueba no puede identificarse, la base de datos no se puede restaurar, aparecen errores sistémicos que invalidan resultados, o una prueba de carga afecta recursos ajenos.
* **Finalizar cuando:** 100% de casos P1 diseñados se ejecutaron y sus resultados están registrados; 100% de riesgos críticos/altos está mitigado con casos y evidencia; 100% de reglas de acceso incluidas tiene resultado; para flujos secundarios se ejecutó al menos 80% de los casos diseñados; hay 0 defectos críticos/altos y como máximo 2 defectos medios y 5 bajos abiertos (solo si el equipo acepta documentarlos); A06 conserva evidencia de 50 solicitudes, status, latencia y hardware; existe retest de correcciones; y se entrega el informe de resultados. La cobertura de casos ejecutados no equivale a cobertura total del código.

---

## 6. Actividades de prueba y estimaciones

| ID WBS | Actividad | Esfuerzo | Dependencia | Resultado |
| :--- | :--- | :--- | :--- | :--- |
| **1.0** | Revisar el proyecto propio de inicio de sesión existente, preparar entorno y datos | 8 h | — | AMB-01 y DAT-01 repetibles |
| **2.0** | Revisar talleres, alcance y riesgos; diseñar los 23 casos y condiciones | 22 h | 1.0 | M01–M17, A01–A06 y trazabilidad |
| **3.0** | Implementar y preparar automatización A01–A06 | 34 h | 2.0 | Unitarias, integración, E2E y carga local |
| **4.0** | Ejecutar casos manuales y registrar hallazgos | 15 h | 2.0 | Evidencia M01–M17 y defectos |
| **5.0** | Retest, regresión y consolidación del informe | 8 h | 3.0 y 4.0 | Informe de cierre y evidencia consolidada |
| **Total** | | **87 h** | | |

La estimación de 87 horas es una propuesta para las actividades de prueba; no incluye volver a desarrollar el prototipo de inicio de sesión. Entre tres integrantes equivale a un promedio de 29 horas por integrante durante el periodo.

---

## 7. Personal

| Rol | Responsabilidades |
| :--- | :--- |
| **Coordinación de pruebas** | Mantener alcance, prioridades, riesgos, trazabilidad e informe |
| **Diseño de pruebas** | Especificar casos, datos, resultados y criterios de aceptación |
| **Ejecución QA** | Ejecutar casos, conservar evidencia, registrar defectos y hacer retest |
| **Automatización** | Programar, mantener y ejecutar los cinco/seis casos estables y scripts de carga local |
| **Desarrollo** | Entregar versión identificada, corregir defectos y describir cambios |
| **Docente / revisor** | Evaluar el cumplimiento académico y aprobar o solicitar ajustes |

---

## 8. Cronograma

El periodo reportado fue del 09/11/2026 al 17/11/2026. No se realizaron pruebas ni horas los días 14 y 15 de noviembre.

| Fecha | Actividades y responsables propuestos | Esfuerzo / salida prevista |
| :--- | :--- | :--- |
| **09/11/2026** | Revisar el prototipo propio existente y preparar entorno/datos. Eduardo y Estefanía. | 8 h — AMB-01, DAT-01 y alcance |
| **10/11/2026** | Revisar talleres, alcance y reglas; analizar riesgos. Eduardo y Estefanía. | 10 h — riesgos y CP-01–CP-09 |
| **11/11/2026** | Diseñar los 17 casos manuales, datos, pasos, resultados y trazabilidad. Estefanía. | 12 h — catálogo manual |
| **12/11/2026** | Preparar y ejecutar A01–A04. Santiago. | 18 h — resultados y defectos |
| **13/11/2026** | Preparar A05 y A06; ejecutar pruebas automatizadas. Santiago y Eduardo. | 16 h — evidencia y mediciones |
| **14–15/11/2026** | Fin de semana; sin actividades planificadas. | 0 h |
| **16/11/2026** | Ejecutar casos manuales y registrar resultados/evidencia. Estefanía y Santiago. | 15 h — estado M01–M17 y defectos |
| **17/11/2026** | Retest, regresión, revisar trazabilidad y cerrar informe. Los tres integrantes. | 8 h — cierre y entregables |
| **Total estimado** | Periodo: 09/11/2026–17/11/2026 | **87 h** |

---

## Anexo A. Registro mínimo de defectos
Cada defecto debe incluir: identificador, versión/commit, caso asociado, entorno, pasos para reproducir, resultado esperado/obtenido, evidencia sin secretos, severidad, prioridad, responsable, estado, corrección y resultado del retest.

## Anexo B. Límites de interpretación
Este plan y el prototipo son recursos académicos. El cumplimiento de los casos no certifica el sistema, no demuestra que esté libre de vulnerabilidades y no sustituye revisión profesional, pruebas de seguridad autorizadas ni controles de operación. Los umbrales de aceptación de rendimiento y la duración del bloqueo deben ser ratificados por el equipo antes de presentarse como requisitos definitivos.