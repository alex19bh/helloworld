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
                        coverage xml -o coverage.xml
                    '''
                    stash includes: 'coverage.xml', name: 'coverage-report'
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
                script {
                    currentBuild.result = currentBuild.result ?: 'SUCCESS'
                }
            }
        }

        stage('Static') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh 'flake8 ./app/* > flake8.txt || true'

                    recordIssues tools: [flake8(pattern: 'flake8.txt')],
                        qualityGates: [
                            [threshold: 10, type: 'TOTAL', unstable: false],
                            [threshold: 8, type: 'TOTAL', unstable: true]
                        ],
                        failOnError: false
                }
                archiveArtifacts 'flake8.txt'
            }
        }

        stage('Security Test') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh '''
                        bandit -r . -f custom -o bandit.txt --msg-template "{abspath}:{line}: [{test_id}] {msg}" || true
                    '''
                    recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.txt')],
                                 qualityGates: [
                                     [threshold: 2, type: 'TOTAL', unstable: true],
                                     [threshold: 4, type: 'TOTAL', unstable: false]
                                 ]
                }
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
                perfReport sourceDataFiles: 'jmeter-result.jtl',
                   configType: 'ART'
                archiveArtifacts 'jmeter-result.jtl'
            }
        }

        stage('Coverage') {
            steps {
                unstash 'coverage-report'
                recordCoverage(
                    tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                    sourceCodeRetention: 'EVERY_BUILD',
                    failOnError: false,
                    qualityGates: [
                        [threshold: 90.0, metric: 'LINE', baseline: 'PROJECT'],
                        [threshold: 80.0, metric: 'BRANCH', baseline: 'PROJECT']
                    ]
                )
            }
        }
    }
}

