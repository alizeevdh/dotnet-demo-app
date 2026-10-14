pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'docker compose build'
            }
        }

        // schema.sql op de Docker-host (VM) zetten, anders maakt de bind mount van
        // docker-compose.yml er een lege map van en crasht MariaDB.
        stage('Prepare schema') {
            steps {
                sh '''
                    docker run --rm -i --entrypoint sh \
                      -v "$(pwd)/TodoApp:/dest" mariadb:11 \
                      -c 'rm -rf /dest/schema.sql && cat > /dest/schema.sql' \
                      < TodoApp/schema.sql
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d'
            }
        }

        stage('Check') {
            steps {
                sh 'sleep 15'
                sh 'docker compose ps'
                sh 'curl -fsS -o /dev/null http://172.17.0.1:5000/ && echo "Todo-app OK"'
            }
        }
    }
}