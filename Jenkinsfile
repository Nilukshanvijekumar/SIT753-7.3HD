pipeline {
    agent any

    environment {
        COMPOSE_PROJECT_NAME = "volunteerhub-ci-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Environment') {
            steps {
                bat '''
                    if not exist backend\\.env copy backend\\.env.example backend\\.env
                    if not exist frontend\\.env copy frontend\\.env.example frontend\\.env
                '''
            }
        }

        stage('Build') {
            steps {
                bat 'docker compose build'
            }
        }

        stage('Test') {
            steps {
                dir('backend') {
                    bat 'npm ci'
                    bat 'npm test'
                }
            }
        }

        stage('Code Quality') {
            steps {
                dir('backend') {
                    bat 'npm run lint'
                }
            }
        }

        stage('Security') {
            steps {
                dir('backend') {
                    bat 'npm audit --audit-level=high || exit /b 0'
                    echo 'Review npm audit output above for your PDF Security section.'
                }
            }
        }

        stage('Deploy') {
            steps {
                bat 'docker compose down || exit /b 0'
                bat 'docker compose up -d'
                sleep time: 60, unit: 'SECONDS'
            }
        }

        stage('Release') {
            steps {
                echo 'Production promotion: stable compose stack on ports 3300 (UI) and 5000 (API).'
                bat 'docker compose ps'
            }
        }

        stage('Monitoring') {
            steps {
                powershell '''
                    $ok = $false
                    for ($i = 0; $i -lt 12; $i++) {
                        try {
                            $r = Invoke-WebRequest -Uri "http://localhost:5000/health" -UseBasicParsing -TimeoutSec 5
                            if ($r.StatusCode -eq 200) {
                                $r.Content | Out-File -FilePath "health.json" -Encoding utf8
                                Get-Content "health.json"
                                $ok = $true
                                break
                            }
                        } catch {
                            Start-Sleep -Seconds 5
                        }
                    }
                    if (-not $ok) { throw "Health check failed: http://localhost:5000/health" }
                '''
                archiveArtifacts artifacts: 'health.json', allowEmptyArchive: false
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            bat 'docker compose logs --tail=80 || exit /b 0'
        }
        always {
            bat 'docker compose down || exit /b 0'
        }
    }
}
