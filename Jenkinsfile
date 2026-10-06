   pipeline {
       agent any
       stages {
           stage('Preparation') {
               steps {
                   catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                       sh 'docker compose down'
                   }
               }
           }
           stage('Build') {
               steps {
                   sh 'docker compose build'
               }
           }
           stage('Deploy') {
               steps {
                   sh 'docker compose up -d'
               }
           }
           stage('Unit tests') {
               steps {
                sh '''
                    docker rm -f dotnet-tests || true
                    docker create --name dotnet-tests --network container:todoappdb -w /src mcr.microsoft.com/dotnet/sdk:10.0 dotnet test
                    docker cp . dotnet-tests:/src
                    docker start -a dotnet-tests
                '''
            }
            post {
                always {
                    sh 'docker rm -f dotnet-tests || true'
                }
            }
        }
           stage('Check') {
               steps {
                   sh 'sleep 15'
                   sh 'docker compose ps'
                   sh 'curl -f http://172.17.0.1:5000/'
               }
           }
       }
   }