//    pipeline {
//        agent any
//        stages {
//            stage('Preparation') {
//                steps {
//                    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
//                        sh 'docker compose down'
//                    }
//                }
//            }
//            stage('Build') {
//                steps {
//                    sh 'docker compose build'
//                }
//            }
//            stage('Deploy') {
//                steps {
//                    sh 'docker compose up -d'
//                }
//            }
//            stage('Unit tests') {
//                steps {
//                 sh '''
//                     docker rm -f dotnet-tests || true
//                     docker create --name dotnet-tests --network container:todoappdb -w /src mcr.microsoft.com/dotnet/sdk:10.0 dotnet test
//                     docker cp . dotnet-tests:/src
//                     docker start -a dotnet-tests
//                 '''
//             }
//             post {
//                 always {
//                     sh 'docker rm -f dotnet-tests || true'
//                 }
//             }
//         }
//            stage('Check') {
//                steps {
//                    sh 'sleep 15'
//                    sh 'docker compose ps'
//                    sh 'curl -f http://172.17.0.1:5000/'
//                }
//            }
//        }
//    }

pipeline {
    agent any

    stages {
        stage('Preparation') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh 'docker compose down -v'
                }
            }
        }

        stage('Build Containers') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Start Database') {
            steps {
                // Start enkel de DB en wacht tot deze healthy is
                sh 'docker compose up -d todoappdb'
                sh 'docker compose ps'
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    docker rm -f dotnet-tests || true
                    
                    # Maak een tijdelijke runner container aan in hetzelfde netwerk als todoappdb
                    docker create --name dotnet-tests \
                        --network container:todoappdb \
                        -e ConnectionStrings__TodoDb="Server=127.0.0.1;Port=3306;Database=todo_test_db;User=todo_usr;Password=letmeinplz;" \
                        -w /src \
                        mcr.microsoft.com/dotnet/sdk:10.0 dotnet test

                    # Kopieer de broncode naar de container
                    docker cp . dotnet-tests:/src

                    # Voer de tests uit
                    docker start -a dotnet-tests
                '''
            }
            post {
                always {
                    sh 'docker rm -f dotnet-tests || true'
                }
            }
        }

        stage('Deploy Application') {
            steps {
                // Start de volledige stack (inclusief de webapp)
                sh 'docker compose up -d'
            }
        }

        stage('Health Check') {
            steps {
                sleep 10
                sh 'docker compose ps'
                sh 'curl -f http://172.17.0.1:5000/'
            }
        }
    }

    post {
        failure {
            echo 'Pipeline mislukt! Bekijk de logs voor details.'
        }
    }
}