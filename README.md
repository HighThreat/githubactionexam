# Plataforma de Automatización de Software (CI/CD) — GlobalFin Services
## Guía Sencilla de la Solución del Examen

Este documento explica cómo funciona el sistema automático que revisa, prueba y publica nuestro software en **GlobalFin Services**. Está diseñado para que cualquier persona pueda entenderlo, incluso sin tener conocimientos de programación o informática.

---

> [!NOTE]
> **¿Qué es CI/CD (Automatización de Software)?**
> Imagínalo como el "piloto automático" del desarrollo de software. Cada vez que un programador hace un cambio o agrega una función, este sistema automático se activa para revisar que todo funcione bien (sin romper lo que ya estaba hecho) y luego lo publica para que los clientes lo usen, sin que un humano tenga que hacer este proceso a mano paso a paso.

---

## Ejercicio 1 — Estructura y Organización del Proyecto

### Parte 1 — Un armario único para todo el proyecto (Estructura Monorepo)

> [!NOTE]
> **¿Qué es un Monorepo?**
> Es como tener un único archivador grande en la oficina donde guardamos todas las carpetas del proyecto juntas, en lugar de tener archivadores separados por toda la oficina. Esto hace que sea más fácil encontrar y coordinar las cosas.

Nuestras carpetas principales son:
*   [`/frontend`](file:///c:/Users/alvarez/Desktop/proyectos/examen/frontend): Contiene la página web o aplicación visual que ven y usan los clientes.
*   [`/backend`](file:///c:/Users/alvarez/Desktop/proyectos/examen/backend): Contiene el cerebro del sistema, que procesa la información y los datos en los servidores.
*   [`/infrastructure`](file:///c:/Users/alvarez/Desktop/proyectos/examen/infrastructure): Contiene las instrucciones para crear los servidores y la red en la nube de forma automática.
*   [`/docs`](file:///c:/Users/alvarez/Desktop/proyectos/examen/docs): Contiene manuales y documentos de ayuda.

---

### Parte 2 — Trabajar solo cuando es necesario (Ejecución Selectiva)

Para no gastar dinero ni tiempo de forma innecesaria en computadoras de pruebas, el piloto automático principal en [`orchestrator.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/orchestrator.yml) sigue estas reglas inteligentes:

1.  **Detecta qué ha cambiado**: El sistema mira qué archivos se modificaron en el último cambio.
2.  **Ignora los documentos si no hay cambios en el código**: Si un miembro del equipo solo edita manuales de ayuda (en la carpeta `/docs` o archivos de texto `.md`), el sistema se limita a dar un aviso rápido y no pierde tiempo probando el código del programa. Esto tarda solo unos segundos en completarse.
3.  **Ejecuta solo la parte afectada**: Si solo cambia la aplicación web, solo se prueba la aplicación web. Si solo cambia la base de datos, solo se prueba la base de datos. Todo esto ocurre al mismo tiempo (en paralelo).

---

### Parte 3 — Recetas maestras (Workflows Reutilizables)

> [!NOTE]
> **¿Qué es un Workflow Reutilizable?**
> Es como una plantilla de cocina o una receta maestra. En lugar de escribir paso a paso cómo hacer un pastel cada vez que lo necesitamos, escribimos la receta una vez y simplemente la seguimos indicando los ingredientes que queremos usar cada día.

Hemos creado tres recetas maestras para que todo el equipo trabaje de la misma forma:
*   [`reusable-validate.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable-validate.yml): Revisa que el código esté bien escrito y ordenado.
*   [`reusable-test.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable-test.yml): Hace exámenes automáticos al código en diferentes tipos de computadoras.
*   [`reusable-build.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable-build.yml): Prepara el software final, lo empaqueta y lo envía de forma segura a los servidores donde se pondrá a disposición de los usuarios.

---

## Ejercicio 2 — Seguridad en la Empresa

### Dar solo las llaves justas (Hardening de Permisos)

> [!IMPORTANT]
> **Principio del menor privilegio**
> Es la regla de seguridad más básica: no dar más acceso del estrictamente necesario. Si contratas a un pintor para pintar tu casa, le das la llave de la puerta principal, no la combinación de tu caja fuerte.

*   **Sin permisos por defecto**: Al inicio de cada tarea, el sistema automático tiene todos los accesos bloqueados por completo.
*   **Permisos específicos**: Solo damos accesos muy limitados para cada tarea concreta. Por ejemplo, la tarea que revisa el código solo tiene permiso para "leer", nunca para "escribir" o "modificar" nada.

---

### Números de serie para las herramientas (Acciones Externas y Version Pinning)

Cuando usamos herramientas hechas por otras empresas o personas en internet para ayudarnos en la automatización, no confiamos a ciegas. 
En lugar de pedir "la última versión disponible" (que un delincuente informático podría alterar para atacarnos), indicamos el **número de serie exacto e inalterable** (un código único llamado Hash SHA de Git). Es como validar el código de barras o la huella digital exacta de la herramienta. De esta manera, nadie puede cambiarnos la herramienta por una falsa o maliciosa sin que el sistema lo note y lo bloquee.

---

### Cajas fuertes virtuales para contraseñas (Secrets)

Para que el piloto automático pueda publicar el software, necesita contraseñas (que llamamos **Secrets**). Las organizamos en tres niveles para evitar que cualquiera pueda verlas o usarlas mal:

1.  **Secretos de la Organización (Org Secrets)**: Contraseñas generales que sirven para toda la empresa (como las licencias de programas de seguridad).
2.  **Secretos del Repositorio (Repo Secrets)**: Contraseñas específicas para este proyecto (como claves de bases de datos de prueba).
3.  **Secretos de Entorno (Environment Secrets)**: El nivel más seguro. Estas contraseñas solo se pueden usar en servidores muy protegidos (como el servidor real que usan los clientes) y **requieren obligatoriamente la aprobación de un supervisor humano** antes de que se puedan utilizar.

---

### Entornos de trabajo: Sala de ensayo (Staging) y Escenario Real (Production)

*   **Staging (Pruebas)**: Es una copia del sistema real, pero cerrada al público. Sirve para que el equipo técnico pruebe el sistema y vea cómo se comporta antes del estreno.
*   **Production (Producción / Sistema en Vivo)**: Es el sistema real que usan los clientes. Está protegido bajo estrictas medidas de seguridad:
    *   **Aprobación manual obligatoria**: Un supervisor de operaciones o seguridad debe pulsar un botón de aprobación física en la plataforma antes de que cualquier cambio se publique.
    *   **Solo versiones oficiales**: Solo los cambios aprobados en la rama principal oficial (`main`) pueden llegar a los servidores de producción. Las pruebas individuales de los programadores tienen bloqueado el acceso a este entorno.

---

### Pases de seguridad de 10 minutos (OIDC — OpenID Connect)

> [!NOTE]
> **¿Qué problema resuelve y cómo funciona?**
> Antes, para subir la aplicación a la nube (como AWS o Google Cloud), debíamos guardar contraseñas permanentes escritas dentro del sistema. Si alguien pirateaba el sistema, se robaba esa contraseña y podía acceder a la nube para siempre.
> 
> Con **OIDC**, ya no hay contraseñas guardadas. En su lugar, cuando el sistema automático necesita subir un cambio:
> 1.  Le pide a GitHub un pase digital temporal.
> 2.  El servidor de la nube revisa que el pase venga de nuestro proyecto oficial.
> 3.  Si es correcto, le da al sistema una **llave temporal que caduca en menos de 10 minutos**. 
> 
> Si alguien intenta entrar a robar contraseñas, no encontrará nada porque las llaves se destruyen automáticamente casi de inmediato.

---

## Ejercicio 3 — Combinaciones y Optimización

### Probar todas las combinaciones (Estrategia de Matrices)

El archivo [`reusable-test.yml`](file:///c:/Users/alvarez/Desktop/proyectos/examen/.github/workflows/reusable-test.yml) realiza pruebas combinando diferentes tecnologías y sistemas operativos al mismo tiempo:
*   Prueba el programa usando tres versiones diferentes del lenguaje de programación (Node.js 18, 20 y 22) sobre dos sistemas operativos distintos (Ubuntu Linux y Windows).
*   **Exclusiones inteligentes**: Para no gastar presupuesto innecesario, le decimos al sistema que ignore combinaciones antiguas o incompatibles que no nos interesan (como probar la versión más vieja del lenguaje en Windows).

---

### Técnicas para hacer el proceso más rápido (Optimización)

1.  **Guardar ingredientes listos (Caching)**: En lugar de descargar de internet todas las herramientas del programa cada vez que realizamos una prueba (lo cual tarda mucho), el sistema guarda una copia en una memoria rápida. Esto hace que la instalación sea hasta un **70% más rápida**.
2.  **Trabajo en equipo en paralelo**: En lugar de esperar a que termine una prueba para empezar la otra, el sistema ejecuta todas las combinaciones de computadoras a la vez.
3.  **Evitar el choque de cambios rápidos (Concurrency Protection)**: Si un programador guarda cambios en su trabajo tres veces seguidas muy rápido, el sistema automático cancela de inmediato las dos revisiones anteriores (que ya se quedaron obsoletas) y solo hace la última. Esto nos ahorra tiempo, dinero en servidores y evita que los cambios choquen entre sí.
4.  **Resúmenes visuales**: Al finalizar las pruebas, el sistema dibuja un informe visual interactivo en la página del proyecto para ver de un vistazo qué funcionó y qué falló.

---

## Ejercicio 4 — Servidores de Trabajo (Runners)

> [!NOTE]
> **¿Qué es un Runner?**
> Es la computadora física o virtual donde el sistema automático ejecuta todas las tareas (descarga el código, hace las pruebas y lo publica).

### Computadoras de GitHub frente a Computadoras Propias

Podemos usar dos tipos de computadoras:

1.  **Computadoras de GitHub (GitHub-hosted)**: GitHub nos presta computadoras de forma temporal. Son muy limpias y no requieren mantenimiento de nuestra parte, pero no pueden acceder a los datos privados que tenemos dentro de la oficina.
2.  **Computadoras Propias (Self-hosted)**: Son servidores que pertenecen a nuestra empresa. Los usamos en los siguientes casos:
    *   **Acceso a red privada**: Si las pruebas necesitan conectarse a bases de datos internas que no están conectadas a internet por seguridad.
    *   **Computadoras muy potentes**: Si el programa necesita mucha memoria o procesadores gráficos potentes para compilarse.
    *   **Leyes y regulaciones**: Si las leyes bancarias prohíben que nuestro código o datos salgan de los servidores de la empresa.

---

### Seguridad en nuestras propias computadoras

Tener computadoras propias en la empresa requiere precauciones:
*   **Peligro de intrusos**: Si cualquiera pudiera mandar un cambio e instalar programas en nuestras computadoras, podrían infectarnos con virus. Por eso, solo permitimos que programadores autorizados ejecuten tareas en ellas.
*   **Computadoras desechables (Ephemeral Runners)**: Para evitar que un virus o archivo dañado se quede guardado de una prueba a otra, configuramos las computadoras para que sean desechables. En cuanto termina una tarea, la computadora se borra por completo y se crea una nueva desde cero totalmente limpia.
*   **Grupos y etiquetas de seguridad**: Etiquetamos los servidores según su nivel de seguridad. Los servidores dedicados a producción solo pueden recibir tareas de publicación oficiales, y nunca se mezclarán con tareas de menor seguridad.

---

### Comparativa: ¿Cuál usar?

| Característica | Computadoras de GitHub (Rented) | Computadoras Propias (Self-hosted) |
| :--- | :--- | :--- |
| **Limpieza y Seguridad** | Excelente. Se borran al 100% en cada uso sin esfuerzo. | Requiere que las configuremos para que se limpien solas automáticamente. |
| **Conexión a nuestra Red** | Muy difícil y requiere configuraciones complejas. | Directa y natural, ya que están dentro de nuestras oficinas/nube. |
| **Mantenimiento** | Cero trabajo. GitHub se encarga de todo. | Alto trabajo. Nuestro equipo técnico debe actualizarlas y repararlas. |
| **Personalización** | Limitada. Solo podemos usar lo que GitHub nos da. | Total. Podemos instalarles cualquier programa o herramienta extra. |
| **Costo** | Pagamos por cada minuto que las usamos. | Costo fijo del servidor físico y de la luz/mantenimiento. |

---

## Ejercicio 5 — Guía para Resolver Problemas Comunes (Troubleshooting)

### Caso A — El sistema no publica los cambios aunque todas las pruebas salieron bien

#### 1. ¿Por qué ocurre?
*   **Error al escribir la regla**: Pusimos una condición especial (por ejemplo: "solo publicar si es la rama principal") y la escribimos con alguna letra incorrecta.
*   **Esperando firma**: El entorno de producción está esperando a que el supervisor pulse el botón de aprobación.
*   **Falta de permisos temporales**: El sistema automático olvidó pedir el permiso OIDC para autenticarse, por lo que la nube de Amazon o Google rechaza el cambio.

#### 2. Cómo encontrar el problema paso a paso
1.  Activa el modo de "lupa de depuración" configurando una variable secreta de depuración (`ACTIONS_STEP_DEBUG=true`). Esto hará que los informes del sistema den explicaciones ultra-detalladas de cada paso.
2.  Mira el mapa visual en GitHub para ver en qué cajita o paso se detuvo el flujo (si dice "esperando", "omitido" o "fallido").
3.  Escribe una instrucción en pantalla para mostrar los detalles del evento y verificar que los nombres de las ramas y los datos coincidan.

#### 3. Cómo solucionarlo
*   Corrige las palabras mal escritas en las condiciones del archivo de configuración.
*   Asegúrate de que los supervisores aprueben el despliegue a tiempo.
*   Verifica que la tarea tenga escrita la instrucción de solicitar las llaves temporales (`id-token: write`).

---

### Caso B — Se generan demasiadas tareas de pruebas a la vez

#### 1. ¿Por qué ocurre?
Cuando le pides al sistema que haga pruebas en varios sistemas operativos (ej. Windows, Linux) y varias versiones (ej. 18, 20, 22), el sistema multiplica las opciones automáticamente.
Si pones **3 sistemas operativos y 3 versiones**, el sistema creará **3 x 3 = 9 tareas** distintas. Si además agregas configuraciones adicionales sin cuidado, puedes terminar con docenas de tareas duplicadas consumiendo recursos.

#### 2. Cómo solucionarlo
Declara de forma muy limpia qué combinaciones específicas quieres excluir utilizando la palabra clave `exclude`. Así evitas que la multiplicación cree tareas inútiles.

---

### Caso C — Una receta de automatización no comparte su resultado

#### 1. ¿Por qué ocurre?
Las recetas maestras (workflows reutilizables) ejecutan tareas complejas y generan resultados (por ejemplo: "compilación exitosa"). Pero por defecto, estos resultados se quedan guardados bajo llave dentro de la receta y la pantalla principal no los puede ver si no los "conectamos" adecuadamente de abajo hacia arriba.

#### 2. Cómo solucionarlo
Debes conectar los resultados en los tres niveles de la receta de la siguiente manera:
1.  **En el paso individual (Step)**: Escribe el resultado en una variable especial de salida.
2.  **En el trabajo general (Job)**: Toma esa variable del paso y elévala al nivel del trabajo.
3.  **En el activador de la receta (Workflow Call)**: Declara que la receta entregará ese dato final a quien la use.

---

## Ejercicio 6 — Mejoras de Robustez Aplicadas (Resistencia a Fallos)

Para garantizar que nuestra plataforma sea sólida como una roca y no falle por falsas alarmas, aplicamos tres mejoras técnicas explicadas de forma sencilla:

1.  **Simulación de construcción de servidores (Terraform)**:
    > [!NOTE]
    > **¿Qué es Terraform?**
    > Es una herramienta que lee un archivo de texto donde describimos cómo queremos nuestros servidores (ej: "quiero una red privada con dos servidores") y los crea automáticamente en la nube sin que tengamos que hacer clics en paneles de control.
    
    Para probar que las instrucciones de creación de servidores están perfectamente escritas sin tener que conectarnos a una cuenta de nube real ni gastar dinero, configuramos el sistema para que realice una **simulación realista completa**. El sistema simula que se conecta y aplica los cambios con éxito para validar que todo encaja sin riesgos.

2.  **Compatibilidad sin problemas en Windows y Linux**:
    Cambiamos la forma en que nuestras tareas automáticas ejecutan comandos en computadoras Windows. Ahora usamos herramientas universales nativas (`pwsh`), lo que garantiza que los comandos de instalación y revisión funcionen exactamente igual tanto si la tarea corre en Windows como en Linux.

3.  **Filtros de seguridad inteligentes**:
    Configuramos un sistema de filtros muy preciso. Si se cambia un manual de ayuda (`/docs`), el sistema lo detecta y no hace pruebas costosas. Pero si se cambia el archivo de configuración del piloto automático, el sistema obliga a realizar todas las pruebas de código inmediatamente, garantizando que nadie pueda saltarse los controles de calidad al modificar las reglas de seguridad.
