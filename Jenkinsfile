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
           stage('Check') {
               steps {
                   sh 'sleep 15'
                   sh 'docker compose ps'
                   sh 'curl -f http://172.17.0.1:5000/'
               }
           }
       }
   }