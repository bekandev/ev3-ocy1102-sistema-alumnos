# Registro de seguridad · Sistema Alumnos

Responsable: Jose Manuel Gonzalez Toloza · Repositorio: https://github.com/bekandev/ev3-ocy1102-sistema-alumnos · Fecha: 27-11-2026

Cada corrección va en su propio commit con el `VULN-xx` en el mensaje y los resultados salen de los builds de Jenkins. Ojo que las horas de Jenkins (tabla de trazabilidad) están en UTC y las del monitoreo en hora local (UTC-3).

## 1. Revisiones de seguridad realizadas

| # | Etapa del SDLC | Herramienta / técnica | Build | Resultado (resumen) |
|---|---|---|---|---|
| R1 | Código (SAST) | Bandit en el pipeline | #1 | 2 hallazgos High / 2 Medium / 1 Low (B324 MD5, B201 debug=True, B608 SQL armado con texto, B104 bind a 0.0.0.0, B105 clave en el código). Después de las correcciones, build #2: 0/0/0 |
| R2 | Dependencias (SCA) | pip-audit en el pipeline | #1 | 19 vulnerabilidades en 4 paquetes (Flask, Werkzeug, Jinja2, gunicorn). Después del PR #1 de Dependabot, build #7: 0 |
| R3 | Pruebas dinámicas (DAST) | OWASP ZAP sobre staging | #1 | 3 High / 3 Medium / 1 Low. Build #2: 0/0/1 (cookie sin SameSite), build #4 y #7: 0/0/0 |
| R4 | Revisión manual | Lectura de `app.py` | - | El login no tenía límite de intentos (fuerza bruta posible, app.py:213) y la búsqueda devolvía el error de SQLite junto con la consulta completa (app.py:209). También se vio que la clave del admin era el MD5 de "admin" (app.py:22) |
| R5 | Producción | Grafana + Prometheus | - | Durante la simulación de ataque la alerta "Ataque detectado en produccion" se disparó a las 17:33. El ataque quedó bloqueado por la aplicación |

## 2. Vulnerabilidades identificadas y corregidas

Las líneas son las del código original (commit `00bac91`).

| ID | Vulnerabilidad (CWE) | Detectada por | Ubicación | Riesgo | Corrección aplicada | Commit | Build que lo verifica |
|---|---|---|---|---|---|---|---|
| VULN-01 | Inyección SQL en /alumnos/buscar (CWE-89) | Bandit B608 y ZAP (SQL Injection, High) | `app.py:205` | Alto | La consulta usa parámetros (`LIKE ?`) en vez de meter el texto del usuario en el SQL | `dac60f8` | #2 |
| VULN-02 | Error de SQL mostrado al usuario (CWE-209) | ZAP (la prueba de SQLi provocó un HTTP 500) y revisión manual | `app.py:209` | Medio | Al usuario se le responde un mensaje genérico y el error real queda solo en el log | `b0bcae0` | #2 |
| VULN-03 | XSS en /hello (CWE-79) | ZAP (XSS reflejado y XSS DOM, High) | `app.py:154` | Alto | El parámetro `name` se escapa con `escape()` de markupsafe antes de ponerlo en el HTML | `f8fa809` | #2 |
| VULN-04 | Secret key escrita en el código (CWE-798 / CWE-259) | Bandit B105 | `app.py:20` | Alto | `SECRET_KEY` se lee de una variable de entorno y si no existe se genera una aleatoria con `secrets` | `91d2dd1` | #2 |
| VULN-05 | Clave del admin en MD5 y credencial por defecto admin/admin (CWE-327) | Bandit B324 | `app.py:221` (hash en `app.py:22`) | Alto | Se usa `check_password_hash` de werkzeug y el hash viene de la variable `ADMIN_CLAVE_HASH`, así que ya no existe admin/admin | `dd5f9c7` | #2 |
| VULN-06 | Login sin token CSRF (CWE-352) | ZAP (Absence of Anti-CSRF Tokens, Medium) | `app.py:69` (formulario) y `app.py:213` | Medio | Se genera un token por sesión, va en un campo oculto del formulario y /login lo valida | `bcca781` | #2 |
| VULN-07 | Modo debug activo y escucha en 0.0.0.0 (CWE-94 / CWE-605) | Bandit B201 y B104 | `app.py:241` | Alto | Debug solo se activa con `FLASK_DEBUG=1` y el arranque directo escucha en 127.0.0.1 | `6a606cd` | #2 |
| VULN-08 | Faltan cabeceras de seguridad (CWE-693 / CWE-1021) | ZAP (clickjacking y CSP Medium, X-Content-Type-Options Low) | `app.py:125` (after_request) | Medio | Se agregan Content-Security-Policy, X-Frame-Options: DENY y X-Content-Type-Options: nosniff a todas las respuestas | `1b3d1ad` | #2 |
| VULN-09 | Login sin límite de intentos, fuerza bruta (CWE-307) | Revisión manual y `scripts/simular_ataque.py` | `app.py:213` | Alto | Máximo 5 intentos fallidos por IP en 300 segundos, después responde HTTP 429 | `6d11ca7` | #2 |
| VULN-10 | Cookie de sesión sin SameSite (CWE-1275) | ZAP build #2 (Low) | `app.py:23` (configuración de la app) | Bajo | Se configura `SESSION_COOKIE_SAMESITE = "Lax"` | `fdcb9de` | #4 |

## 3. Resultados de OWASP ZAP

| Alerta de ZAP | Riesgo | URL de ejemplo | VULN relacionada | Cómo se mitiga | ¿Mitigada? Build |
|---|---|---|---|---|---|
| Cross Site Scripting (Reflected) | High | `http://jenkins:5001/hello?name=</h1><scrIpt>alert(1);</scRipt><h1>` | VULN-03 | Escapar el parámetro `name` | Sí, build #2 |
| SQL Injection | High | `http://jenkins:5001/alumnos/buscar?nombre='` | VULN-01, VULN-02 | Consulta parametrizada y error genérico | Sí, build #2 |
| Cross Site Scripting (DOM Based) | High | `http://jenkins:5001/hello?name=<script>alert(5397)</script>` | VULN-03 | Escapar el parámetro `name` | Sí, build #2 |
| Missing Anti-clickjacking Header | Medium | `http://jenkins:5001` | VULN-08 | X-Frame-Options: DENY y `frame-ancestors 'none'` en la CSP | Sí, build #2 |
| Content Security Policy (CSP) Header Not Set | Medium | `http://jenkins:5001/sitemap.xml` | VULN-08 | Cabecera Content-Security-Policy en todas las respuestas | Sí, build #2 |
| Absence of Anti-CSRF Tokens | Medium | `http://jenkins:5001` (formulario de /login) | VULN-06 | Token CSRF en el formulario y validación en /login | Sí, build #2 |
| X-Content-Type-Options Header Missing | Low | `http://jenkins:5001/alumnos/1` | VULN-08 | X-Content-Type-Options: nosniff | Sí, build #2 |
| Cookie without SameSite Attribute (apareció en build #2) | Low | `http://jenkins:5001` | VULN-10 | `SESSION_COOKIE_SAMESITE = "Lax"` | Sí, build #4 |

En los builds #4 y #7 ZAP solo deja 2 alertas informativas (Session Management Response Identified y User Agent Fuzzer). Esas no son vulnerabilidades.

## 4. Monitoreo e incidentes

Grafana lee las métricas que Prometheus saca de `/metrics` de producción: `peticiones_sospechosas_total` (por tipo), `login_fallidos_total` y `http_requests_total` (por código de estado). La regla de alerta es `sum(increase(peticiones_sospechosas_total{job="produccion"}[1m])) > 0`, evaluada cada 10 s. Para probarla se ejecutó `scripts/simular_ataque.py` contra producción.

| Hora | Panel / métrica de Grafana | Qué se observó | Entorno | Clasificación | Acción de respuesta | Verificación |
|---|---|---|---|---|---|---|
| 17:31:49 | `login_fallidos_total`, `peticiones_sospechosas_total`, `http_requests_total` con 429 | Primera oleada desde 192.168.117.5: 31 intentos de login como admin (26 bloqueados con 429), una inyección SQL y un XSS. Los contadores subieron pero la alerta no se disparó, porque las series recién se estaban creando y `increase()` necesita dos muestras | Producción | Intento de ataque web (fuerza bruta + SQLi + XSS) | Se revisó el log de producción: los avisos "Login fallido" y "Login bloqueado por exceso de intentos" vienen todos de la misma IP | Ninguna clave funcionó, la SQLi no devolvió datos y el XSS salió escapado |
| 17:32:56 | Mismas métricas | Segunda oleada: 31 intentos de login, los 31 bloqueados con 429. SQLi y XSS otra vez sin resultado | Producción | Mismo ataque, repetido | Se dejó correr para comprobar que la alerta funciona | Mismo resultado que la primera oleada |
| 17:33:00 | Alerta "Ataque detectado en produccion" | La alerta pasó a Alerting (Firing) con valor 2.18 | Producción | Severidad crítica según la regla. Sin impacto, la aplicación bloqueó todo | Se revisaron los paneles y el log (66 avisos en total, todos de 192.168.117.5). Las defensas que actuaron fueron el bloqueo por intentos (VULN-09), la consulta parametrizada (VULN-01) y el escape del HTML (VULN-03). En un caso real además se bloquearía la IP | No hubo login correcto ni datos expuestos |
| 17:34:00 | Alerta "Ataque detectado en produccion" | La alerta volvió a Normal al terminar el ataque | Producción | Incidente cerrado | Ninguna adicional | Los contadores dejaron de subir |

## 5. Gestión de dependencias

Dependabot está configurado en `.github/dependabot.yml` y agrupa las actualizaciones de pip en un solo PR. Como pip-audit corre en cada build, cualquier versión nueva se revisa antes de llegar a producción.

| Paquete | Versión inicial | Vulnerabilidades (IDs) | Versión final | Origen del cambio (PR de Dependabot / manual) | ¿Pruebas, pip-audit y ZAP siguen OK? Build |
|---|---|---|---|---|---|
| Flask | 2.2.2 | PYSEC-2023-62 (CVE-2023-30861), PYSEC-2026-2151 (GHSA-68rp-wp8r-4726) | 3.1.3 | PR #1 de Dependabot (agrupado) | Sí, build #7 (9/9 pruebas, pip-audit 0, ZAP 0/0/0) |
| Werkzeug | 2.2.2 | PYSEC-2023-58 (GHSA-xg9f-g7g7-2323), PYSEC-2023-57 (GHSA-px8h-6qxv-m22q), PYSEC-2023-221 (GHSA-hrfv-mqp8-q5rw), PYSEC-2026-2046 (GHSA-hgf8-39gv-g3f2), PYSEC-2026-2045 (CVE-2024-49766), PYSEC-2026-2044 (GHSA-87hc-h4r5-73f7), PYSEC-2026-2043 (GHSA-2g68-c3qc-8985), PYSEC-2026-2320 (CVE-2026-27199), PYSEC-2026-3417 (CVE-2024-49767), PYSEC-2026-4312 (CVE-2026-102598) | 3.1.9 | PR #1 de Dependabot (agrupado) | Sí, build #7 (9/9 pruebas, pip-audit 0, ZAP 0/0/0) |
| Jinja2 | 3.1.2 | PYSEC-2026-1473 (CVE-2024-22195), PYSEC-2026-1471 (GHSA-cpwx-vrp4-4pq7), PYSEC-2026-1474 (CVE-2024-34064), PYSEC-2026-1475 (GHSA-q2x7-8rv6-6q7h), PYSEC-2026-1472 (GHSA-gmj6-6f8f-6699) | 3.1.6 | PR #1 de Dependabot (agrupado) | Sí, build #7 (9/9 pruebas, pip-audit 0, ZAP 0/0/0) |
| gunicorn | 21.2.0 | PYSEC-2026-1434 (CVE-2024-1135), PYSEC-2026-1433 (CVE-2024-6827) | 27.0.1 | PR #1 de Dependabot (agrupado) | Sí, build #7 (9/9 pruebas, pip-audit 0, ZAP 0/0/0) |
| prometheus-client | 0.17.1 | Ninguna reportada por pip-audit | 0.26.0 | PR #1 de Dependabot (agrupado) | Sí, build #7 (9/9 pruebas, pip-audit 0, ZAP 0/0/0) |
| pytest (requirements-dev.txt) | 8.3.4 | PYSEC-2026-1845 (tmpdir inseguro, alerta de Dependabot; el pipeline audita solo requirements.txt) | 9.2.0 | PR #1 de Dependabot (agrupado) | Sí, build #7 (9/9 pruebas, pip-audit 0, ZAP 0/0/0) |

El PR #1 (commit `e98cbe3`, merge `c6e1ea4`) actualizó los 6 paquetes de una vez. Dependabot mostraba 20 alertas: las 19 de pip-audit más la de pytest en `requirements-dev.txt`. Al fusionarlo, GitHub cerró las 20 alertas como Fixed y los PRs #2 a #6, que eran actualizaciones sueltas de los mismos paquetes, se cerraron solos. Para comprobar que la actualización no metía vulnerabilidades nuevas se dejó correr el pipeline completo sobre `c6e1ea4`: en el build #7 pasaron las 9 pruebas, pip-audit dio 0 vulnerabilidades, Bandit 0/0/0 y ZAP 0/0/0, y recién ahí se desplegó a producción.

## 6. Trazabilidad

`scripts/trazabilidad.py` agrega una fila al registro al final de cada build y Jenkins lo archiva como `reports/TRAZABILIDAD.md`. Estas son las filas del job `sistema-alumnos`:

| Build | Fecha (UTC) | Versión | Commit | Mensaje del commit | Pruebas | Bandit H/M/L | pip-audit | ZAP H/M/L | Resultado | Producción |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 2026-11-27 19:42 | 1.0.1+bb1103d | `bb1103d` | agregar dependabot | 9/9 OK | 2/2/1 | 19 vuln. | 3/3/1 | UNSTABLE | no desplegado |
| 2 | 2026-11-27 20:00 | 1.0.2+6d11ca7 | `6d11ca7` | VULN-09: bloquear login despues de varios intentos fallidos | 9/9 OK | 0/0/0 | 19 vuln. | 0/0/1 | UNSTABLE | no desplegado |
| 3 | 2026-11-27 20:00 | 1.0.3+6d11ca7 | `6d11ca7` | VULN-09: bloquear login despues de varios intentos fallidos | 9/9 OK | 0/0/0 | 19 vuln. | n/a | FAILURE | no desplegado |
| 4 | 2026-11-27 20:03 | 1.0.4+fdcb9de | `fdcb9de` | VULN-10: agregar SameSite a la cookie de sesion | 9/9 OK | 0/0/0 | 19 vuln. | 0/0/0 | UNSTABLE | no desplegado |
| 5 | 2026-11-27 20:26 | 1.0.5+9823ad4 | `9823ad4` | actualizar readme y limpiar comentarios | 9/9 OK | 0/0/0 | 19 vuln. | n/a | FAILURE | no desplegado |
| 6 | 2026-11-27 20:29 | 1.0.6+c6e1ea4 | `c6e1ea4` | Merge pull request #1 (Dependabot) | 9/9 OK | 0/0/0 | 0 vuln. | n/a | FAILURE | no desplegado |
| 7 | 2026-11-27 20:31 | 1.0.7+c6e1ea4 | `c6e1ea4` | Merge pull request #1 (Dependabot) | 9/9 OK | 0/0/0 | 0 vuln. | 0/0/0 | SUCCESS | 1.0.7+c6e1ea4 |

Los builds #3, #5 y #6 fallaron en Deploy Staging. La causa fue que dos jobs de prueba anteriores (`sistema-alumnos-old` y `sistema-alumnos-old2`) seguían habilitados, revisaban el mismo repositorio y desplegaban su propio staging en el puerto 5001 al mismo tiempo. Cuando me di cuenta los deshabilité y el build #7 pasó completo. Por esa misma razón el archivo `TRAZABILIDAD.md` que guarda Jenkins tiene además algunas filas de esos dos jobs; la tabla de arriba deja solo las de este job.

| Actividad | Hallazgo | Build que lo detectó | Commit / PR que lo resolvió | Build que lo verificó | Versión en producción |
|---|---|---|---|---|---|
| Revisión de seguridad (Bandit) | 2 High / 2 Medium / 1 Low: MD5, debug=True, SQL armado con texto, bind a 0.0.0.0, clave en el código | #1 | `dac60f8` (VULN-01), `91d2dd1` (VULN-04), `dd5f9c7` (VULN-05), `6a606cd` (VULN-07) | #2 | 1.0.7+c6e1ea4 |
| Prueba automatizada (OWASP ZAP) | 3 High / 3 Medium / 1 Low en el build #1 (XSS, SQLi, CSRF, cabeceras) y cookie sin SameSite en el build #2 | #1 | `dac60f8` a `6d11ca7` (VULN-01 a VULN-09) y `fdcb9de` (VULN-10) | #4 | 1.0.7+c6e1ea4 |
| Gestión de dependencias (Dependabot) | 19 vulnerabilidades en 4 paquetes según pip-audit, 20 alertas de Dependabot | #1 | PR #1 (`e98cbe3`, merge `c6e1ea4`) | #7 | 1.0.7+c6e1ea4 |
