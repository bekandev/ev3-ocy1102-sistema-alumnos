# Sistema Alumnos

API y portal web en Flask para consultar alumnos de SecureDev. El pipeline de Jenkins construye la app, corre las pruebas, revisa el código y las dependencias, la escanea con OWASP ZAP y recién ahí la despliega a producción. Producción la vigilo con Prometheus y Grafana.

## Rutas

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/` | Portada con enlaces y formularios |
| GET | `/hello?name=Ana` | Saludo |
| GET | `/alumnos` · `/alumnos/<id>` | Listado y detalle de alumnos |
| GET | `/alumnos/buscar?nombre=Ana` | Búsqueda por nombre |
| POST | `/login` | Acceso del administrador |
| GET | `/health` · `/version` | Estado y versión desplegada |
| GET | `/metrics` | Métricas para Prometheus |

## Estructura

| Archivo | Para qué sirve |
|---|---|
| `app.py` | La aplicación |
| `tests/` | Pruebas con pytest |
| `requirements.txt` | Dependencias de ejecución |
| `requirements-dev.txt` | Dependencias para las pruebas |
| `Jenkinsfile` | Pipeline de CI/CD |
| `.github/dependabot.yml` | Configuración de Dependabot |
| `scripts/deploy.sh` | Despliega en staging (5001) o producción (5000) y hace una prueba de humo |
| `scripts/zap_scan.py` | Escaneo de OWASP ZAP (spider, pasivo y activo) |
| `scripts/trazabilidad.py` | Registro build ↔ versión ↔ commit ↔ hallazgos ↔ despliegue |
| `scripts/revisar_local.sh` | Pruebas, Bandit y pip-audit en local antes de hacer push |
| `scripts/simular_ataque.py` | Ataque de prueba contra un entorno propio para ver la alerta en Grafana |
| `docs/REGISTRO_SEGURIDAD.md` | Registro de revisiones, vulnerabilidades, incidentes y dependencias |

## Correr en local

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt -r requirements-dev.txt
.venv/bin/python -m pytest
.venv/bin/python app.py        # http://127.0.0.1:5000
```

Para que funcione el login hay que definir `ADMIN_CLAVE_HASH` con un hash generado por werkzeug, y en producción también `SECRET_KEY`:

```bash
python3 -c "from werkzeug.security import generate_password_hash; print(generate_password_hash('tu-clave'))"
```

## Pipeline

1. **Build**: crea el entorno virtual e instala dependencias. La versión queda como `1.0.<build>+<commit>`.
2. **Test**: pytest, con reporte JUnit.
3. **Security Review (SAST)**: Bandit sobre `app.py`. Si hay hallazgos Medium o High el build queda UNSTABLE.
4. **Dependency Management**: pip-audit sobre `requirements.txt`. Si hay CVEs el build queda UNSTABLE.
5. **Deploy Staging**: despliega en el puerto 5001.
6. **DAST - OWASP ZAP**: escanea staging. Si hay alertas High o Medium el build queda UNSTABLE.
7. **Deploy Producción**: solo corre si todo lo anterior salió bien.

Al final de cada build `scripts/trazabilidad.py` agrega una fila al registro de trazabilidad y Jenkins archiva los reportes (`reports/`).

## Vulnerabilidades corregidas

Cada corrección va en su propio commit con el ID del hallazgo.

| ID | Problema | Corrección |
|---|---|---|
| VULN-01 | Inyección SQL en `/alumnos/buscar` | Consulta parametrizada |
| VULN-02 | El error de SQL se mostraba al usuario | Mensaje genérico, el detalle va al log |
| VULN-03 | XSS reflejado en `/hello` | Se escapa el parámetro `name` |
| VULN-04 | `SECRET_KEY` escrita en el código | Se lee de una variable de entorno |
| VULN-05 | Clave admin en MD5 y por defecto `admin/admin` | Hash de werkzeug desde variable de entorno |
| VULN-06 | Login sin protección CSRF | Token CSRF en el formulario |
| VULN-07 | `debug=True` y escucha en `0.0.0.0` | Debug apagado y `127.0.0.1` |
| VULN-08 | Faltaban cabeceras de seguridad | CSP, X-Frame-Options y X-Content-Type-Options |
| VULN-09 | Sin límite de intentos de login | Bloqueo (429) tras 5 intentos fallidos por IP |
| VULN-10 | Cookie de sesión sin SameSite | `SameSite=Lax` |

El detalle está en [`docs/REGISTRO_SEGURIDAD.md`](docs/REGISTRO_SEGURIDAD.md).

## Dependencias

Dependabot revisa `requirements*.txt` todos los días y abre un PR con las actualizaciones. Cuando se fusiona, Jenkins vuelve a pasar las pruebas, Bandit, pip-audit y ZAP antes de desplegar.

## Monitoreo

La app expone en `/metrics`: `http_requests_total`, `login_fallidos_total`, `peticiones_sospechosas_total{tipo}` y `app_info{version}`. Prometheus las lee cada 5 segundos y Grafana tiene un dashboard de producción y una alerta cuando aparecen peticiones con patrones de ataque.
