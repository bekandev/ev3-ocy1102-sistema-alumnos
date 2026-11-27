// Jenkinsfile - pipeline de "Sistema Alumnos" (SecureDev)
// build, pruebas, revisiones de seguridad, staging con ZAP y despliegue a produccion
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // Revisa GitHub cada ~3 minutos: cada push (tuyo o un PR de Dependabot fusionado) dispara un build.
    triggers {
        pollSCM('H/3 * * * *')
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Versión = 1.0.<nº de build>+<commit corto>: une cada despliegue con su commit (trazabilidad)
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'git log -1 --format="Commit %h de %an: %s"'
                // Cada build parte limpio: sin reportes viejos y con un entorno virtual nuevo
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'python3 -m venv .venv'
                sh '.venv/bin/pip install --quiet -r requirements.txt -r requirements-dev.txt'
                sh '.venv/bin/python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh '.venv/bin/python -m pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        stage('Security Review (SAST)') {
            steps {
                // bandit (SAST): el json lo usa trazabilidad.py, el txt es para leerlo
                sh 'bandit -r app.py -f json -o reports/bandit.json --exit-zero'
                sh 'bandit -r app.py -f txt -o reports/bandit.txt --exit-zero'
                sh 'cat reports/bandit.txt'
                // si hay hallazgos medium/high el build queda UNSTABLE (y no se despliega a produccion)
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE', message: 'Bandit encontro hallazgos medium/high') {
                    sh 'bandit -q -ll -r app.py'
                }
            }
        }

        stage('Dependency Management') {
            steps {
                // pip-audit revisa si requirements.txt tiene CVEs conocidos
                sh 'pip-audit -r requirements.txt -f json -o reports/pip-audit.json || true'
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE', message: 'pip-audit encontro dependencias vulnerables') {
                    sh 'pip-audit -r requirements.txt --desc on > reports/pip-audit.txt 2>&1; rc=$?; cat reports/pip-audit.txt; exit $rc'
                }
            }
        }

        stage('Deploy Staging') {
            steps {
                sh 'bash scripts/deploy.sh staging 5001'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                // escaneo de zap sobre staging (spider + pasivo + activo)
                script {
                    def rc = sh(returnStatus: true, script: 'python3 scripts/zap_scan.py http://jenkins:5001')
                    if (rc == 2) {
                        unstable('ZAP encontro alertas high/medium')
                    } else if (rc != 0) {
                        error("Fallo el escaneo de ZAP (codigo ${rc})")
                    }
                }
            }
        }

        stage('Deploy Producción') {
            // solo se despliega a produccion si todo lo anterior paso
            when {
                beforeAgent true
                expression { currentBuild.currentResult == 'SUCCESS' }
            }
            steps {
                sh 'bash scripts/deploy.sh produccion 5000'
                script { env.PRODUCCION = env.APP_VERSION }
            }
        }
    }

    post {
        always {
            // Registro de trazabilidad: build <-> versión <-> commit <-> pruebas <-> hallazgos <-> despliegue
            withEnv(["RESULTADO=${currentBuild.currentResult}"]) {
                sh 'python3 scripts/trazabilidad.py'
            }
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}
