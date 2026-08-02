# Auditando Mi Propia Herramienta
### Una Autoauditoría de web-vuln-control-mapping, Cuatro Hallazgos Reales y Lo Que Me Enseñaron Sobre el Riesgo

**Caso de estudio | Sebastián Garay | GRC, Riesgo Humano y Shadow AI**

> Esta es una autoauditoría genuina, no un ejercicio de demostración. Cada hallazgo de abajo se descubrió mientras construía y mantenía activamente este proyecto, y cada arreglo se aplicó al repositorio real y en producción. Las fechas y los mensajes de commit son reales. Nada acá está montado.

## Por qué auditar tu propio código

La mayoría de los casos de estudio de portfolio analizan la brecha de otro. Eso sirve, pero tiene un riesgo: es fácil sonar sabio sobre fallas que uno no vivió. Este es distinto. `web-vuln-control-mapping` es mi propia herramienta, una pequeña app en Next.js que mapea vulnerabilidades web comunes a riesgo y a controles de gobernanza. A lo largo de construirla y mantenerla, surgieron cuatro problemas reales. Los documento como documentaría hallazgos en un trabajo con un cliente: condición, criterio, causa, efecto y recomendación, porque el formato importa más cuando el sujeto es tu propio trabajo. Es más fácil ser honesto sobre los errores ajenos que sobre los propios.

## Alcance y método

La auditoría cubre el código de la aplicación, su postura de dependencias y su capa de datos, tal como existían a lo largo de varias semanas de desarrollo activo. Los hallazgos se descubrieron con una mezcla de revisión manual de código, escaneo de dependencias (`npm audit`) y pruebas directas de ambos endpoints de la API. No salí a buscar problemas para inflar este informe. Cada hallazgo acá refleja una brecha real, no una hipotética.

## Hallazgo 1: Los endpoints de la API aceptaban entrada sin límite

**Condición:** Los dos endpoints de servidor, `/api/hash` y `/api/headers`, validaban que los campos de entrada fueran del tipo correcto (string), pero no ponían ningún límite a su longitud.

**Criterio:** Cualquier endpoint que acepte entrada del usuario debe acotar su tamaño antes de procesarla. Es un control básico bajo la función Proteger de NIST CSF (PR.PS, Seguridad de Plataforma) y práctica estándar para cualquier API pública.

**Causa:** La lógica de validación original chequeaba corrección (¿es un string?) pero no escala (¿qué tan grande es este string?). Es una omisión común porque el camino feliz, alguien pegando una contraseña normal o un set normal de headers HTTP, nunca ejercita el límite faltante.

**Efecto:** Un llamador podía enviar un payload arbitrariamente grande (megabytes de texto) a cualquiera de los dos endpoints, forzando al servidor a hashearlo o parsearlo, ocupando CPU sin costo para el atacante. Un vector de denegación de servicio trivial y sin autenticación, en una herramienta cuyo propósito entero es enseñar a pensar en exactamente esta clase de riesgo.

**Recomendación y remediación:** Se agregaron chequeos explícitos de longitud máxima (100.000 caracteres para `/api/hash`, 20.000 para `/api/headers`) devolviendo HTTP 413 al excederse. Ambos límites cubiertos con tests unitarios que afirman la respuesta 413. Corregido en el commit `fix(security): add input length limits on API routes and security headers`.

## Hallazgo 2: La herramienta que audita headers de seguridad no tenía ninguno propio

**Condición:** `web-vuln-control-mapping` incluye un Analizador de Headers que revisa en un sitio objetivo seis headers de seguridad estándar (HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy y CSP). El propio despliegue de la app no ponía ninguno.

**Criterio:** ISO/IEC 27001:2022 Anexo A.8.9 (Gestión de Configuración) y las líneas base de seguridad web esperan que una aplicación aplique a sí misma las protecciones que es capaz de aplicar, no solo que las describa para otros.

**Causa:** Next.js no pone estos headers por defecto, y agregarlos requiere una función `headers()` explícita en el archivo de config, un paso fácil de saltear cuando la funcionalidad visible (la herramienta analizadora) ya funciona bien de forma aislada.

**Efecto:** Más allá de la exposición directa (clickjacking, MIME sniffing, filtración de referrer), había un problema de credibilidad: cualquiera con el nivel técnico para correr mi propia herramienta contra mi propio sitio lo encontraría fallando su propio chequeo. Eso es peor que los headers faltantes en sí.

**Recomendación y remediación:** Se agregaron cinco de los seis headers directamente en `next.config.mjs` (Strict-Transport-Security, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy). Deliberadamente no se agregó Content-Security-Policy todavía: una CSP mal configurada puede romper los propios scripts y estilos de la app, y publicar una CSP rota es peor que no tener ninguna. Eso queda como un ítem abierto y documentado, no como un arreglo apurado. Corregido en el mismo commit que el Hallazgo 1.

## Hallazgo 3: La dependencia del framework cargaba catorce advisories sin parchear

**Condición:** El proyecto corría Next.js 14.2.35, la última versión de su línea de release, contra la cual `npm audit` reportaba catorce advisories publicados por separado (variantes de denegación de servicio, envenenamiento de caché, request smuggling, cross-site scripting y server-side request forgery), agrupados bajo una única entrada de severidad alta.

**Criterio:** Las dependencias con vulnerabilidades conocidas y divulgadas y sin parche disponible en su línea actual deberían triarse y actualizarse en un ciclo definido, no dejarse indefinidamente porque la versión actual todavía corre. Esto mapea a la función Identificar de NIST CSF (ID.RA, Evaluación de Riesgos) aplicada a componentes de terceros.

**Causa:** La línea 14.x había llegado a su release final; cada corrección para estos advisories solo existía en la siguiente versión mayor. Actualizar una versión mayor de framework trae su propio riesgo de breaking changes, por lo que se había diferido.

**Efecto:** Antes del triage, el conteo crudo (catorce advisories, uno calificado como alto) se veía alarmante. Después de leer realmente cada uno contra el código, ocho de los catorce resultaron inaplicables: requerían funcionalidades que este proyecto no usa para nada (el optimizador de imágenes, middleware, ruteo i18n, rewrites, upgrades de WebSocket, nonces de CSP). La exposición residual real era más acotada de lo que el conteo sugería, pero no cero.

**Recomendación y remediación:** Se actualizó a Next.js 15 y React 19 juntos, usando una rama dedicada, un build y test local completos, y un smoke test manual antes de mergear. Verificado con un `npm audit` de seguimiento, que bajó de catorce advisories de severidad alta a uno solo moderado y sin relación (una dependencia interna de PostCSS, no explotable en el proceso de build de este proyecto). Corregido en el commit `chore: upgrade to Next.js 15, React 19, and lucide-react latest`.

## Hallazgo 4: La herramienta nombrada por el mapeo de controles nunca pobló sus controles

**Condición:** El proyecto se llama `web-vuln-control-mapping`, y tanto su README como la descripción de GitHub prometen que cada payload mapea a "un control relacionado (NIST 800-53, ISO 27001 Anexo A, OWASP ASVS)". La capa de datos contaba otra historia. En `lib/explain.ts`, los campos `owasp`, `control` y `mitigation` estaban declarados como opcionales y quedaron vacíos en las 23 entradas, y el panel Explain en `PayloadGenerator.tsx` renderizaba solo `summary`, `why` y `when`. El mapeo de controles por el que el proyecto está nombrado no existía ni en los datos ni en la UI.

**Criterio:** El nombre y el propósito declarado de un artefacto deben coincidir con lo que entrega. Es el estándar de honestidad-primero que aplico a mis propios casos de estudio, que donde interpreto en lugar de reportar un hecho, lo digo, y aplica con más rigor a una herramienta cuya premisa entera es conectar un hallazgo técnico con un control de gobernanza.

**Causa:** El esquema de tipos hacía opcionales los tres campos de gobernanza (`owasp?`, `control?`, `mitigation?`), así que una entrada era estructuralmente válida con ellos vacíos. El README describía el diseño previsto en lugar del estado publicado. Y a diferencia de los Hallazgos 1 al 3, este vacío nunca produjo un error en tiempo de ejecución, ningún build falló y ninguna página se rompió, así que nada lo forzó a salir a la superficie. Era un vacío de contenido, invisible para todos los tests.

**Efecto:** Este es el lugar más consecuente donde podría sentarse un vacío. El diferenciador de todo el proyecto es conectar la capa técnica con la gobernanza, y un revisor que abriera el repo nombrado por ese mapeo exacto habría encontrado el mapeo ausente. Una herramienta que nombra una capacidad que no entrega socava su propia premisa más de lo que lo haría una funcionalidad faltante en otro lado.

**Recomendación y remediación:** Se hicieron `owasp`, `control` y `mitigation` campos obligatorios en el tipo `PayloadExplanation`, así que uno vacío ahora es un error de compilación en lugar de una omisión silenciosa. Se poblaron las 23 entradas con su categoría real de OWASP Top 10 2021, un control mapeado (NIST SP 800-53 SI-10 / ISO 27001:2022 Anexo A.8.28 y relacionados), y una mitigación concreta por técnica. Se actualizó `PayloadGenerator.tsx` para renderizar los tres campos de gobernanza en el panel Explain, separados visualmente de los campos conceptuales. Verificado con un `tsc --noEmit` limpio, un build de producción exitoso y la suite de tests existente. Corregido en el commit `feat(explain): populate owasp/control/mitigation fields across all 23 payloads`.

## Qué tienen en común estos cuatro hallazgos

Los Hallazgos 1 al 3 son atajos impulsados por la fricción, no vacíos de conocimiento. Sabía que el input debía tener un límite; todavía no había llegado a hacerlo. Sabía que una herramienta que chequea headers de seguridad debería configurar los propios; era fácil saltearlo una vez que la feature visible funcionaba de forma aislada. Sabía que el framework estaba atrasado para una actualización mayor; el riesgo de breaking changes lo seguía postergando.

El Hallazgo 4 es otra cosa, y vale nombrarlo como tal: no un atajo tomado bajo presión, sino una promesa escrita antes de que el trabajo detrás estuviera terminado. El README describía hacia dónde iba la herramienta, y los datos nunca lo alcanzaron. Es su propia falla, el artefacto cuyo alcance declarado corre por delante de su estado publicado, y es fácil de pasar por alto precisamente porque, a diferencia de los primeros tres, nunca rompe nada de forma ruidosa.

Es el mismo patrón que escribí en el caso de estudio de la brecha de Uber de 2022: la brecha entre lo que un control debería hacer y lo que un atajo efectivamente termina publicando, bajo presión de tiempo, cuando nadie está mirando esa línea específica todavía. Los controles no fallan porque la gente no los entienda. Fallan porque el camino seguro era más lento que el atajo, en el momento en que el atajo se eligió.

## Conclusiones clave

- **Acotar el tamaño del input es un control básico que el camino feliz nunca te obliga a agregar.** Un pegado de tamaño normal nunca ejercita el límite faltante; hace falta un payload deliberadamente grande para exponerlo, y para entonces ya es un vector de denegación de servicio en vivo.
- **Una herramienta que enseña seguridad debería cumplir su propia vara.** El hallazgo del analizador de headers no era una falla técnica, era de credibilidad, y esas se acumulan más rápido que las técnicas.
- **Un conteo crudo de vulnerabilidades no es una evaluación de riesgos.** Catorce advisories sonaban graves. Leer cada uno contra el uso real de funcionalidades cortó la exposición real a más de la mitad antes de cambiar una sola línea de código.
- **Un nombre es una promesa, y los datos tienen que cumplirla.** El mapeo de controles por el que todo el proyecto está nombrado estaba declarado opcional en el tipo y vacío. Hacerlo un campo obligatorio convirtió un vacío de contenido fácil de pasar por alto en un error de compilación, así la promesa ya no puede desviarse de lo que se publica.

## Qué estoy aprendiendo de esto

Auditar el propio trabajo es más difícil de lo que parece, no técnicamente, sino psicológicamente. Cada uno de estos hallazgos existió durante días o semanas antes de que los escribiera, porque en el momento no se sentían como hallazgos, se sentían como atajos razonables que iba a limpiar después. Escribirlos en este formato, condición, criterio, causa, efecto, forzó un nivel de honestidad que arreglar el código en silencio nunca habría forzado.

Esa es probablemente la lección más transferible acá, más que cualquier arreglo individual. El trabajo de GRC no se trata principalmente de conocer los frameworks. Se trata de estar dispuesto a escribir, en un documento que otro va a leer, el momento exacto en que tu propio criterio eligió velocidad por sobre rigor. Lo hice cuatro veces en un proyecto. Espero encontrarlo de nuevo en el próximo.

---

*Fuentes: historial de commits de git y salida de `npm audit` del repositorio `web-vuln-control-mapping`, antes y después de la remediación. Esta es una autoauditoría de primera parte, no una evaluación independiente de terceros.*
