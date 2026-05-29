# Plataforma CI/CD Enterprise — GlobalFin Services
## Solución del Examen de GitHub Actions

Este repositorio contiene la solución completa para la plataforma CI/CD enterprise de **GlobalFin Services**. La implementación cumple con políticas corporativas estrictas para entornos híbridos (frontend, backend e infraestructura), asegurando un diseño seguro, reutilizable, escalable, mantenible y optimizado.

---

## Ejercicio 1 — Arquitectura Pipeline Enterprise

### Parte 1 — Estructura Monorepo
El monorepo está organizado en las siguientes carpetas funcionales:
*   [`/frontend`](file:///c:/Users/alvarez/Desktop/proyectos/examen/frontend): Contiene la aplicación web con sus scripts de compilación, linters y tests unitarios.
*   [`/backend`](file:///c:/Users/alvarez/Desktop/proyectos/examen/backend): Contiene el servicio API REST con su suite de pruebas y scripts NPM.
*   [`/infrastructure`](file:///c:/Users/alvarez/Desktop/proyectos/examen/infrastructure): Contiene el código de infraestructura como código (IaC) en Terraform.
*   [`/docs`](file:///c:/Users/alvarez/Desktop/proyectos/examen/docs): Directorio de documentación técnica del proyecto.

### Parte 2 — Ejecución Selectiva (Selective Execution)
Para evitar el gasto innecesario de minutos de computación en runners (especialmente en proyectos monorepo de gran escala), el workflow principal [`orchestrator.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/orchestrator.yml) implementa las siguientes lógicas:
1.  **Detección de Cambios**: Mediante la acción `dorny/paths-filter`, se evalúa qué archivos fueron modificados en el commit o Pull Request.
2.  **Omisión de Documentación**: Si las modificaciones ocurren exclusivamente dentro de `/docs` o archivos `.md` de la raíz del repositorio, se activa la rama de ejecución `docs-notification`. Esta no ejecuta pruebas de código ni compilaciones, completándose en pocos segundos.
3.  **Ejecución Dirigida**: Los pipelines de `/frontend`, `/backend` e `/infrastructure` se disparan de forma independiente y en paralelo gracias a las condiciones `if` evaluadas dinámicamente basadas en los outputs del job detector.

### Parte 3 — Workflows Reutilizables (Reusable Workflows)
Para estandarizar las tareas de CI/CD entre equipos y evitar la duplicación de código YAML, se crearon plantillas parametrizables bajo el directorio [`.github/workflows/reusable/`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable):
*   [`reusable-validate.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable/reusable-validate.yml): Encargado del análisis estático (linting/formatting) con aislamiento de permisos.
*   [`reusable-test.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable/reusable-test.yml): Administra las pruebas unitarias y de integración sobre matrices multiplataforma.
*   [`reusable-build.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable/reusable-build.yml): Ejecuta la compilación de código, empaqueta artefactos y coordina despliegues seguros hacia entornos multi-etapa.

---

## Ejercicio 2 — Seguridad Enterprise

### Hardening de Permisos (Principle of Least Privilege)
En un entorno corporativo como GlobalFin Services, las políticas de seguridad prohíben la asignación de permisos globales de escritura. 
*   **Permiso Global**: Definimos `permissions: {}` en la raíz de cada workflow para revocar el token `GITHUB_TOKEN` por defecto en todos los jobs.
*   **Permiso Específico por Job**: Asignamos únicamente los scopes mínimos necesarios para las tareas del job individual. Por ejemplo:
    *   Para analizar código: `permissions: contents: read`
    *   Para OIDC IAM Federation: `permissions: id-token: write` y `permissions: contents: read`
    *   Para detectar cambios en PRs: `permissions: pull-requests: read` y `permissions: contents: read`

### Acciones Externas (Supply Chain Control)
Para prevenir ataques de inyección en dependencias (como la mutabilidad de etiquetas de versión como `@v4`), aplicamos **Version Pinning** mediante el hash SHA inmutable de Git:
```yaml
uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332 # v4.1.7
```
Esto garantiza que el código ejecutado en el pipeline sea exactamente el revisado y auditado, evitando que un atacante modifique una tag en el repositorio origen de la acción externa para inyectar malware.

### Diseño y Ámbito de Secrets
Para asegurar el principio de separación de funciones (*separation of duties*), dividimos las variables y secretos según su ciclo de vida y nivel de riesgo:
1.  **Secrets de Organización (Org Secrets)**: Secretos transversales de lectura obligatoria en toda la compañía (ej. Tokens de SonarQube, licencias globales). Se configuran políticas de acceso restringido a repositorios privados seleccionados.
2.  **Secrets de Repositorio (Repo Secrets)**: Secretos particulares para este monorepo (ej. Llaves de API de desarrollo o dependencias comerciales).
3.  **Secrets de Entorno (Environment Secrets)**: El máximo estándar de seguridad. Secretos que solo están disponibles cuando el job se ejecuta bajo el contexto de un entorno protegido (ej. `AWS_ROLE_TO_ASSUME` para producción). Un desarrollador con acceso al código no puede leer estos secretos a menos que el deploy sea aprobado por un revisor autorizado.

### Entornos (Environments): Staging y Production
Configuramos dos entornos lógicos en las definiciones de despliegue de GitHub:
*   **Staging**: Permite despliegues rápidos para pruebas integrales, limitados a la rama `main` o ramas de integración (`develop`).
*   **Production**: Protegido estrictamente bajo políticas corporativas:
    *   **Required Reviewers (Aprobación Manual)**: Requiere la firma/aprobación de al menos un miembro del equipo de Operaciones/SecOps antes de liberar el deployment.
    *   **Deployment Branch Limits**: Configurado en la UI de GitHub para que únicamente despliegues originados en la rama `main` puedan ejecutarse. Intentos de desplegar ramas de feature son denegados a nivel de plataforma.

### OIDC (OpenID Connect) en GitHub Actions
OpenID Connect (OIDC) elimina la necesidad de almacenar llaves de acceso estáticas y de larga duración (como llaves AWS Access Key ID / Secret Access Key) dentro de los secretos de GitHub.

#### 1. ¿Qué problema resuelve?
Resuelve el riesgo del compromiso de credenciales estáticas. Si un atacante vulnera el repositorio de GitHub, podría extraer las llaves estáticas y comprometer toda la nube de la organización de manera permanente. Con OIDC, no hay secretos guardados en GitHub.

#### 2. ¿Cómo mejora la seguridad?
*   **Credenciales Efímeras**: GitHub Actions solicita un token web JSON (JWT) de corta duración (válido típicamente por menos de 10 minutos) firmado por GitHub.
*   **Federación de Identidad**: El proveedor de nube (AWS, Azure o GCP) confía en la firma de GitHub y valida las condiciones (*claims*) del JWT (como el nombre del repositorio, la rama o el entorno). Si coincide con la política IAM preestablecida, la nube otorga credenciales temporales específicas.
*   **Auditoría Precisa**: Cada despliegue genera eventos de seguridad únicos en los logs de la nube (CloudTrail en AWS) mapeados al ID específico del workflow run de GitHub Actions.

#### 3. ¿Cuándo utilizarlo?
Debe utilizarse obligatoriamente en todos los despliegues a la nube pública (AWS, GCP, Azure, Kubernetes en la nube) dentro de entornos empresariales, reemplazando cualquier uso de tokens permanentes o llaves de usuario IAM.

---

## Ejercicio 3 — Matrix y Optimización

### Estrategia de Matrices
El workflow [`reusable-test.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable/reusable-test.yml) implementa una matriz de pruebas que cubre múltiples plataformas y runtimes, combinando lógica de inclusión y exclusión:
*   **Multi-Runtime & Multi-Platform**: Ejecuta pruebas en Node.js `18`, `20` y `22` sobre sistemas operativos `ubuntu-latest` y `windows-latest`.
*   **Include**: Permite añadir propiedades especiales como `experimental: true` para Node.js 22 en Ubuntu sin duplicar la lógica de configuración del job.
*   **Exclude**: Excluye Node.js 18 en `windows-latest`. Esto responde a políticas internas para reducir costos y evitar compatibilidades obsoletas en plataformas Windows con runtimes heredados.

### Técnicas de Optimización Aplicadas
1.  **Caching**: Implementamos la propiedad `cache: 'npm'` dentro de `actions/setup-node`. Esto cachea automáticamente el directorio `~/.npm` reduciendo el tiempo de instalación de dependencias hasta en un 70%.
2.  **Paralelización**: Al definir la matriz de jobs con `fail-fast: false`, GitHub Actions paraleliza la ejecución de todas las variantes del sistema operativo y Node.js. Un fallo en una plataforma no cancela inmediatamente las pruebas en las demás, facilitando un diagnóstico integral.
3.  **Concurrency Protection**: En el orquestador principal controlamos la concurrencia:
    ```yaml
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    ```
    Si un desarrollador hace push rápidos y consecutivos a una rama, las ejecuciones previas redundantes se cancelan automáticamente, liberando runners y ahorrando costos computacionales.
4.  **Reutilización de Lógica**: A través de `workflow_call`, el orquestador llama a los pipelines de testing y validación centralizados, eliminando la duplicidad de secuencias complejas.

### Reporting
Para mejorar el monitoreo, los pipelines generan:
*   **Markdown Job Summary**: Al final del testing, se escribe información detallada al archivo `$GITHUB_STEP_SUMMARY`. Esto crea un dashboard de pruebas visual directamente en la pestaña del pipeline en GitHub.
*   **Artifacts**: Guardamos y subimos los reportes de salida en formato `.txt` con una retención estricta de 5 a 10 días para auditorías corporativas.

---

## Ejercicio 4 — Self-hosted Runners

### ¿Cuándo usar Self-hosted Runners?
1.  **Requisitos de Red Privada**: Cuando los despliegues o pruebas necesitan acceder a recursos internos de la red corporativa (bases de datos no expuestas a Internet, registros de contenedores privados, VPCs de la empresa).
2.  **Especificaciones de Hardware**: Si el build requiere aceleración por hardware (GPUs para machine learning, arquitecturas ARM nativas, CPUs con memoria masiva).
3.  **Cumplimiento Normativo (Compliance)**: Políticas de seguridad financiera estrictas que impiden que el código fuente o los datos se procesen en servidores gestionados por terceros (Microsoft/GitHub).
4.  **Optimización de Costos**: Cargas de trabajo de integración continua masivas de 24/7 donde el costo por minuto de GitHub-hosted runners supera el mantenimiento de instancias locales/propias en la nube (AWS EC2, Kubernetes).

### Riesgos de Seguridad y Mitigación
*   **Ataques de Ejecución de Código en Pull Requests**: Si el repositorio es público o tiene permisos abiertos, un atacante podría enviar una PR con modificaciones en el código de tests para ejecutar comandos maliciosos y robar credenciales del servidor self-hosted.
    *   *Mitigación*: Nunca ejecutar runners en repositorios públicos. Usar la política "Require approval for all outside contributors" y deshabilitar ejecuciones automáticas de PRs en runners sensibles.
*   **Persistencia y Envenenamiento de Entorno**: A diferencia de los runners de GitHub que son efímeros, un self-hosted runner tradicional retiene el estado en disco. Si un build escribe malware o archivos infectados, el siguiente build se verá comprometido.
    *   *Mitigación*: Implementar **runners efímeros** (Ephemeral Runners). Se configuran con el flag `--ephemeral` al iniciar el agente. Una vez completado un solo job, el runner se destruye e inicia limpio.

### Estrategia de Organización y Aislamiento
*   **Labels de Segmentación**: Los runners se etiquetan sistemáticamente. Por ejemplo: `runs-on: [self-hosted, prod, linux, aws]`. De este modo, los despliegues de producción solo se ejecutan en runners con alta seguridad dedicados únicamente a producción.
*   **Aislamiento Tecnológico**: Utilizar el **Actions Runner Controller (ARC)** sobre Kubernetes para aprovisionar pods de runners dinámicamente. Cada job corre aislado en su propio pod basado en Docker de forma limpia.
*   **Runner Groups**: En GitHub Enterprise, se estructuran "Runner Groups" asociados a organizaciones específicas, impidiendo que repositorios de menor confianza (ej. sandbox) puedan agotar la cola o utilizar recursos de los runners destinados a infraestructura crítica.

### Ventajas y Desventajas (Comparativa)

| Característica | GitHub-hosted Runners | Self-hosted Runners |
| :--- | :--- | :--- |
| **Seguridad de Entorno** | Excelente. Máquinas virtuales efímeras limpiadas al 100% en cada job. | Requiere configuración activa (ARC / Ephemeral flags) para evitar persistencia. |
| **Integración de Red** | Complejo. Requiere configurar VPNs, proxies o abrir firewalls (IPs públicas dinámicas). | Nativo. Permite residir dentro de la VPC corporativa segura. |
| **Mantenimiento** | Cero. GitHub administra parches del OS, paquetes y actualizaciones de hardware. | Alto. El equipo de infraestructura debe actualizar dependencias, OS y escalar runners. |
| **Personalización** | Limitada a las imágenes de GitHub Virtual Environments. | Total. Libertad de instalar herramientas pesadas pre-instaladas para builds veloces. |
| **Costo** | Pago por minuto de uso. | Costo fijo del servidor/máquina + costo de mantenimiento de infraestructura. |

---

## Ejercicio 5 — Troubleshooting

### Caso A — El pipeline no ejecuta el deploy aunque los tests son correctos
#### 1. Posibles Causas
*   **Condición `if` no evaluada correctamente**: Un filtro en la rama de ejecución (`if: github.ref == 'refs/heads/main'`) que no coincide por diferencias tipográficas o sintaxis de paths.
*   **Estructura de dependencias (`needs`) incorrecta**: Si el job de despliegue depende de un job que falló o que fue omitido, y no se especificó la regla de ejecución correcta (ej. evaluar `always()` o faltar en la lista de `needs`).
*   **Falta de aprobación manual**: El job de deploy está configurado bajo un `environment` (ej. `production`) que tiene un revisor asignado en GitHub, y el pipeline está pausado en espera de la aprobación.
*   **Falta de Permisos OIDC**: El job no tiene el permiso `permissions: id-token: write` para federar credenciales, por lo que la nube rechaza el token y el paso de login falla impidiendo el deployment.

#### 2. Diagnóstico paso a paso
1.  Habilitar logs de depuración del pipeline agregando las variables secretas `ACTIONS_STEP_DEBUG=true` y `ACTIONS_RUNNER_DEBUG=true` a nivel de repositorio.
2.  Visualizar el grafo de dependencias en la pestaña de GitHub Actions para verificar si el job está en estado "Skipped" (omitido), "Waiting" (esperando aprobación) o "Failed" (fallido).
3.  Examinar el payload del evento desencadenador utilizando una expresión para printear el contexto completo: `run: echo "${{ toJSON(github) }}"`. Esto ayuda a verificar si `github.ref` o `github.event_name` tienen los valores que la condición `if` esperaba.

#### 3. Resolución
*   Corregir la lógica condicional en el YAML para que coincida exactamente con la rama destino o trigger deseado.
*   Asegurar que los entornos en GitHub tengan las aprobaciones correspondientes procesadas en tiempo y forma.
*   Configurar los permisos correctos en el job del deploy:
    ```yaml
    permissions:
      contents: read
      id-token: write
    ```

---

### Caso B — Una matrix genera más jobs de los esperados
#### 1. Por qué ocurre
GitHub Actions genera por defecto el **producto cartesiano** (cruzado) de todos los elementos definidos en la sección `matrix`. 
Si se define:
```yaml
matrix:
  os: [ubuntu-latest, windows-latest, macos-latest]  # 3 elementos
  node-version: [18, 20, 22]                          # 3 elementos
```
GitHub Actions generará automáticamente **3 x 3 = 9 jobs** individuales.
Si además el desarrollador desea agregar una configuración especial y utiliza mal la sección `include`, como por ejemplo:
```yaml
matrix:
  os: [ubuntu-latest, windows-latest, macos-latest]
  node-version: [18, 20, 22]
  include:
    - os: ubuntu-latest
      node-version: 16  # Genera un décimo job
    - runtime: python   # Si no se mapea correctamente, puede multiplicar o agregar combinaciones redundantes.
```
Si se especifican listas adicionales en el `include` que no coinciden exactamente con combinaciones existentes, se crearán jobs adicionales en lugar de enriquecer las combinaciones de la matrix base.

#### 2. Identificación de Errores Comunes
*   Intentar usar `include` para agregar variables sin especificar el par clave-valor de la matriz base. Si solo se escribe `- fast-mode: true` en `include`, se asume como una nueva combinación y genera jobs adicionales indeseados.
*   Errores tipográficos en las claves (ej. escribir `node_version` en `include` y `node-version` en la matriz base).

#### 3. Solución
Para resolver la multiplicación descontrolada, se debe estructurar el `include` y `exclude` de manera precisa:
1.  **Exclude**: Eliminar del producto cartesiano las combinaciones inviables financieramente o técnicamente.
2.  **Include**: Usar `include` únicamente para complementar combinaciones que ya existen dentro del producto cartesiano original, o bien declarar explícitamente matrices planas.
Ejemplo optimizado:
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node-version: [20, 22]
    exclude:
      - os: windows-latest
        node-version: 20 # Excluye esta combinación específica
    include:
      - os: ubuntu-latest
        node-version: 22
        experimental: true # Enriquece la combinación ubuntu/node22 sin añadir nuevos jobs
```

---

### Caso C — Un reusable workflow no recibe correctamente outputs
#### 1. Posibles Causas
*   **Falta de Declaración en `workflow_call`**: El reusable workflow tiene un step que genera un output, pero no se ha declarado a nivel del trigger `on: workflow_call: outputs`.
*   **Falta de Mapeo en el Job**: Dentro del reusable workflow, el job que contiene el step del output no propaga el valor a nivel de job-outputs.
*   **Sintaxis de acceso incorrecta en el Caller**: En el workflow principal llamador, se intenta acceder al output del reusable sin referenciar el job correcto o sin utilizar la sintaxis de contexto `needs.<job_id>.outputs.<output_name>`.
*   **Ausencia de dependencias `needs`**: Si el job caller que consume el output no tiene un `needs: [reusable-job-id]`, no tendrá acceso a sus metadatos ni outputs en tiempo de ejecución.

#### 2. Identificación del Problema de Scope
Los outputs tienen tres scopes de visibilidad:
1.  *Step Scope*: `steps.step_id.outputs.output_key` (solo visible en el mismo Job).
2.  *Job Scope*: `jobs.job_id.outputs.output_key` (mapea el step output para que sea visible fuera del Job).
3.  *Workflow Scope (Reusable)*: `on.workflow_call.outputs.output_key` (mapea el job output para que sea visible por el caller workflow).

#### 3. Solución
El mapeo de outputs debe configurarse en los tres niveles de la siguiente forma:

##### Paso 1: Configuración en el Reusable Workflow (`reusable-build.yml`)
```yaml
# 1. Definición del trigger workflow_call
on:
  workflow_call:
    outputs:
      build-status:
        description: "Estado de la compilación"
        value: ${{ jobs.build_job.outputs.status }}  # Propagación a nivel de Workflow

jobs:
  build_job:
    runs-on: ubuntu-latest
    outputs:
      status: ${{ steps.step_build.outputs.result }}  # Propagación a nivel de Job
    steps:
      - id: step_build
        run: echo "result=success" >> $GITHUB_OUTPUT  # Declaración en el Step
```

##### Paso 2: Consumo en el Caller Workflow (`orchestrator.yml`)
```yaml
jobs:
  call-build:
    uses: ./.github/workflows/reusable/reusable-build.yml
    with:
      directory: backend

  deploy:
    needs: call-build # Obligatorio para heredar outputs
    runs-on: ubuntu-latest
    steps:
      - name: Usar Output
        run: echo "El estado del build previo fue: ${{ needs.call-build.outputs.build-status }}"
```

---

## Preguntas Teóricas Cortas

### 1. Diferencia entre Hosted y Self-hosted Runners
*   **GitHub-hosted Runners**: Son máquinas virtuales limpias y seguras administradas en la nube por GitHub (Microsoft). Tienen un costo por minuto, se crean de forma efímera para cada job y se destruyen inmediatamente al terminar, garantizando un entorno aislado y sin mantenimiento.
*   **Self-hosted Runners**: Son servidores físicos o virtuales administrados e instalados por la propia organización. Ofrecen control total del sistema operativo, red local y personalización de herramientas. No tienen cobro de minutos de GitHub, pero la organización debe costear el mantenimiento de infraestructura, parches y garantizar la seguridad de aislamiento.

### 2. Diferencia entre variables (`vars`) y secretos (`secrets`)
*   **Variables (`vars`)**: Almacenan configuraciones en texto plano que no son sensibles (ej. rutas de API, nombres de entornos, puertos, configuraciones de logs). Son legibles directamente en los logs del pipeline y en la configuración del repositorio.
*   **Secretos (`secrets`)**: Almacenan información altamente confidencial (ej. llaves de acceso, contraseñas, certificados, tokens de APIs). GitHub los enmascara automáticamente en los logs de salida (reemplazándolos con `***`) y se almacenan cifrados en repositorios y entornos.

### 3. Cuándo usar Reusable Workflow frente a Composite Action
*   **Reusable Workflow**: Se debe usar cuando se requiere estandarizar un **pipeline completo** o un conjunto de múltiples jobs interconectados. Permite usar diferentes tipos de runners, definir matrices de ejecución, configurar secretos de entornos diferentes y aplicar concurrency a nivel macro.
*   **Composite Action**: Se debe usar cuando se requiere reutilizar una **secuencia específica de pasos** dentro de un mismo job (ej. descargar código, instalar Node, hacer login, configurar dependencias). Solo soporta la ejecución en el mismo runner y contexto del job que lo invoca, sin la capacidad de definir múltiples jobs independientes.

### 4. Qué riesgos tiene usar Actions externas sin pinning
El principal riesgo es el compromiso de la **Cadena de Suministro (Supply Chain Attack)**. Si una acción externa usa tags mutables (ej. `uses: autor/action-name@v1`), un atacante que comprometa la cuenta del autor de la acción puede re-etiquetar un release conteniendo código malicioso. El pipeline empresarial ejecutará automáticamente el código infectado en el siguiente commit, dando acceso a credenciales, servidores y bases de datos internas.

### 5. Qué ventajas aporta OIDC (OpenID Connect)
*   **Cero Credenciales Estáticas**: No requiere almacenar secretos de larga duración en GitHub.
*   **Tokens Dinámicos**: Los tokens otorgados tienen una validez de pocos minutos.
*   **Acceso Basado en Roles (RBAC)**: Permite que el proveedor de la nube otorgue permisos basados exactamente en el contexto del job de GitHub (ej. solo permitir despliegues si el job corre en la rama `main` y en el repositorio específico).
*   **Seguridad de Supply Chain**: Elimina el vector de ataque de exfiltración de credenciales fijas de despliegue.

### 6. Qué contexts suelen provocar más errores
*   **Contexto `env`**: Intentar usar variables del contexto `env` en la definición de parámetros del trigger o en condiciones a nivel de job (como `runs-on` o `concurrency`), donde este contexto aún no se ha evaluado.
*   **Contexto `secrets`**: Tratar de pasar secretos a un reusable workflow mediante `with` en vez de declararlo bajo el bloque específico `secrets`.
*   **Contexto `github`**: Asumir que `github.head_ref` está disponible en triggers tipo `push` (solo existe en eventos `pull_request`).

### 7. Qué diferencia existe entre Parse-time (tiempo de parseo) y Runtime (tiempo de ejecución)
*   **Parse-time**: Ocurre cuando GitHub lee y procesa el archivo YAML del workflow antes de iniciar la cola de ejecución. En esta etapa se validan la estructura sintáctica, se evalúan expresiones que definen el flujo de control (como los `if` de los jobs), las matrices (`strategy.matrix`) y el orquestador de concurrencia. No se tiene acceso a outputs de jobs previos o estados del disco.
*   **Runtime**: Ocurre cuando el job ya se ha asignado a un runner y los pasos (`steps`) se están ejecutando secuencialmente. En esta etapa se evalúan variables de entorno de shell, se generan archivos, se ejecutan scripts y se leen/escriben outputs dinámicos utilizando `$GITHUB_OUTPUT` o `$GITHUB_STEP_SUMMARY`.

### 8. Qué ventajas aporta `concurrency`
*   **Ahorro de Costos**: Cancela ejecuciones obsoletas de forma inmediata si hay commits sucesivos, evitando desperdiciar runner-minutes.
*   **Integridad de Estado (No-Race Conditions)**: Garantiza que despliegues paralelos al mismo entorno no se crucen. Previene, por ejemplo, que dos ejecuciones de Terraform intenten aplicar cambios al mismo tiempo, bloqueando la base de datos de estado (`state lock`) o corrompiendo la infraestructura de la empresa.
*   **Orden de Ejecución**: Mantiene el control del flujo de despliegue, asegurando que la última versión del código en la rama de producción sea la que realmente quede desplegada.

---

## Ejercicio 6 — Refactorización de Robustez Arquitectónica (Enterprise Resilience)

Para alcanzar un estándar verdaderamente Enterprise a prueba de fallos y evitar falsos positivos en el pipeline, se aplicaron mejoras profundas a la estructura original:

### 1. Terraform Completo y Despliegue Real-Simulado
El pipeline de infraestructura no solo se limita a realizar un análisis superficial. El archivo `main.tf` despliega una arquitectura de red completa (VPC, Subredes Públicas/Privadas, Internet Gateway y Tablas de Rutas). 
Para validar este despliegue complejo sin requerir credenciales reales en el entorno de desarrollo, el provider de AWS se configuró con:
*   `skip_credentials_validation = true`
*   `skip_metadata_api_check = true`
*   `skip_requesting_account_id = true`
Esto permite que el step `terraform apply -auto-approve` se ejecute con éxito comprobando toda la lógica de dependencias y outputs, sirviendo como una prueba real de integración. A su vez, la autenticación OIDC tiene configurado `continue-on-error: true` para que el flujo sea ininterrumpido en entornos sin federación.

### 2. Multiplataforma Nativa y Resiliencia en Scripts
Se eliminó la dependencia de Git Bash en los runners de Windows. Las evaluaciones de caché e instalación de NPM (`npm ci` vs `npm install`) se ejecutan ahora utilizando **PowerShell Core (`pwsh`)**, asegurando compatibilidad 100% nativa en `windows-latest` y `ubuntu-latest`. Adicionalmente, el job experimental de Node 22 (`experimental: true`) cuenta con su propio control de fallos (`continue-on-error`) para no comprometer el estado del pipeline de producción.

### 3. Filtros Estrictos para Bypass de Documentación
El detector de cambios (`dorny/paths-filter`) fue calibrado para prevenir bloqueos por omisión. 
*   Se agregó la ruta `.github/workflows/**` como disparador obligatorio de los pipelines de *frontend*, *backend* e *infrastructure*, garantizando que cualquier modificación a las reglas del CI active todas las pruebas.
*   El pipeline de notificación de documentación (`docs-notification`) se disparará **únicamente** cuando se detecten cambios bajo `docs/` o archivos `.md`, y **ninguno** en las áreas principales, evitando que los cambios críticos evadan los chequeos de calidad.
