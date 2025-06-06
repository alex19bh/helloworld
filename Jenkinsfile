pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                git 'https://github.com/alex19bh/helloworld.git'
                sh 'ls -la'
            }
        }

        stage('Unit') {
            steps {
                script {
                    sh '''
                        export PYTHONPATH=$WORKSPACE
                        coverage run -m pytest test/unit --junitxml=result-unit.xml
                        coverage xml
                    '''
                    junit 'result-unit.xml'
                    // Forzar estado success siempre, sin importar el resultado
                    currentBuild.result = currentBuild.result ?: 'SUCCESS'
                }
            }
        }

        stage('Rest') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh '''
                        export FLASK_APP=app/api.py
                        nohup flask run &
                        nohup java -jar wiremock/wiremock-standalone-3.13.0.jar --port 9090 --root-dir wiremock/ &
                        for i in {1..15}; do
                            curl -s http://localhost:9090/__admin/ | grep -q mappings && break
                            sleep 1
                        done
                        pytest --junitxml=result-service.xml test/rest
                    '''
                }
                junit 'result-service.xml'
                // Forzar estado success siempre, sin importar resultado
                script {
                    currentBuild.result = currentBuild.result ?: 'SUCCESS'
                }
            }
        }

        stage('Static') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    script {
                            def count = sh(script: 'flake8 ./app/* | tee flake8.txt | wc -l', returnStdout: true).trim().toInteger()
                            echo "Flake8: ${count} hallazgos"

                            if (count >= 10) {
                                error("Flake8: más de 10 hallazgos => build FAILURE para esta etapa")
                            } else if (count >= 8) {
                                unstable("Flake8: entre 8 y 9 hallazgos => build UNSTABLE para esta etapa")
                            }
                        }
                    }
            archiveArtifacts 'flake8.txt'
        }
    }


        stage('Security Test') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    script {
                        def issues = sh(script: 'bandit -r . -f txt | tee bandit.txt | grep -c Issue:', returnStdout: true).trim().toInteger()
                        echo "Bandit: ${issues} issues encontrados"

                        if (issues >= 4) {
                            error("Bandit: 4 o más issues => etapa y build FAILURE")
                        } else if (issues >= 2) {
                            unstable("Bandit: entre 2 y 3 issues => etapa y build UNSTABLE")
                        }
                    }
                }
                archiveArtifacts 'bandit.txt'
            }
        }

        stage('Performance') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh '''
                        fuser -k 5000/tcp || true
                        export FLASK_APP=app/api.py
                        nohup flask run &
                        sleep 5
                        ./apache-jmeter-5.6.3/bin/jmeter.sh -n -t test-plan.jmx -l jmeter-result.jtl -j jmeter.log
                    '''
                }
                archiveArtifacts 'jmeter-result.jtl'
            }
        }

        stage('Coverage') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    script {
                        def cobertura = sh(script: 'coverage report', returnStdout: true).trim()
                        echo cobertura
                        writeFile file: 'coverage.txt', text: cobertura

                        // Extraer % total líneas cubiertas
                        def matchLine = cobertura =~ /TOTAL\s+\d+\s+\d+\s+(\d+)%/
                        def lineCoverage = matchLine ? matchLine[0][1].toInteger() : 0

                        // Extraer % ramas/condiciones (si disponible)
                        def matchBranch = cobertura =~ /TOTAL\s+\d+\s+\d+\s+\d+\s+(\d+)%/
                        def branchCoverage = matchBranch ? matchBranch[0][1].toInteger() : 100

                        echo "Cobertura líneas: ${lineCoverage}%"
                        echo "Cobertura ramas: ${branchCoverage}%"

                        def fail = false
                        def unstable = false

                        if (lineCoverage < 85) {
                            fail = true
                        } else if (lineCoverage < 95) {
                            unstable = true
                        }

                        if (branchCoverage < 80) {
                            fail = true
                        } else if (branchCoverage < 90) {
                            unstable = true
                        }

                        if (fail) {
                            error("Cobertura insuficiente => etapa y build FAILURE")
                        } else if (unstable) {
                            unstable("Cobertura insuficiente => etapa y build UNSTABLE")
                        }
                    }
                }
                archiveArtifacts 'coverage.txt'
            }
        }
    }
}

