pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hola desde Jenkins Alejandro Ortiz'
            }
        }

        stage('Checkout') {
            steps {
                git 'https://github.com/alex19bh/helloworld.git'
                sh 'ls -la'
                sh 'echo $WORKSPACE'
            }
        }

        stage('Build') {
            steps {
                echo 'No hay que hacer build porque es Python'
            }
        }

        stage('Pruebas') { 
            parallel {
                stage('Unit') {
                    steps {
                        sh '''
                            export PYTHONPATH=$WORKSPACE
                            pytest --junitxml=result-unit.xml $WORKSPACE/test/unit
                        '''
                    }
                }
                
                stage('Service') {
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh '''
                                export FLASK_APP=$WORKSPACE/app/api.py
                                export FLASK_ENV=development
                                nohup flask run 2>&1 &
                                nohup java -jar $WORKSPACE/wiremock/wiremock-standalone-3.13.0.jar --port 9090 --root-dir $WORKSPACE/wiremock/ 2>&1 & 
                                # Esperar a que el puerto 9090 esté disponible (máx 15s)
                                for i in {1..15}; do
                                curl -s http://localhost:9090/__admin/ | grep -q mappings && break
                                echo "Esperando a que WireMock inicie..."
                                sleep 1
                                done                    
                                export PYTHONPATH=$WORKSPACE
                                pytest --junitxml=result-service.xml $WORKSPACE/test/rest                   
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Resultados') {
            steps {
                junit 'result*.xml'
            }
        }
    }
}
