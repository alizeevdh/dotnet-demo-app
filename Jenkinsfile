// pipeline {
//     agent any

//     stages {
//         stage('Build') {
//             steps {
//                 sh 'docker compose build'
//             }
//         }

//         // schema.sql op de Docker-host (VM) zetten, anders maakt de bind mount van
//         // docker-compose.yml er een lege map van en crasht MariaDB.
//         stage('Prepare schema') {
//             steps {
//                 sh '''
//                     docker run --rm -i --entrypoint sh \
//                       -v "$(pwd)/TodoApp:/dest" mariadb:11 \
//                       -c 'rm -rf /dest/schema.sql && cat > /dest/schema.sql' \
//                       < TodoApp/schema.sql
//                 '''
//             }
//         }

//         stage('Deploy') {
//             steps {
//                 sh 'docker compose down || true'
//                 sh 'docker compose up -d'
//             }
//         }

//         stage('Check') {
//             steps {
//                 sh 'sleep 15'
//                 sh 'docker compose ps'
//                 sh 'curl -fsS -o /dev/null http://172.17.0.1:5000/ && echo "Todo-app OK"'
//             }
//         }
//     }
// }

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'docker compose build'
            }
        }

        // Unit tests tegen een tijdelijke wegwerpdatabase, vóór de deployment.
        // Falen de tests, dan wordt Deploy overgeslagen en blijft de draaiende versie ongemoeid.
        stage('Test') {
            steps {
                sh '''
                    set -e
                    docker rm -f test-db dotnet-tests 2>/dev/null || true

                    docker run -d --name test-db \
                      -e MARIADB_ROOT_PASSWORD=sekrit \
                      -e MARIADB_DATABASE=todo_db \
                      -e MARIADB_USER=todo_usr \
                      -e MARIADB_PASSWORD=letmeinplz \
                      mariadb:11

                    for i in $(seq 1 20); do
                      docker exec test-db mariadb-admin ping -h 127.0.0.1 -uroot -psekrit --silent && break
                      sleep 3
                    done

                    # schema in todo_db en in todo_test_db (de tests mogen kiezen)
                    docker exec -i test-db mariadb -h 127.0.0.1 -uroot -psekrit todo_db < TodoApp/schema.sql
                    docker exec test-db mariadb -h 127.0.0.1 -uroot -psekrit \
                      -e "CREATE DATABASE IF NOT EXISTS todo_test_db; GRANT ALL ON todo_test_db.* TO 'todo_usr'@'%'; FLUSH PRIVILEGES;"
                    docker exec -i test-db mariadb -h 127.0.0.1 -uroot -psekrit todo_test_db < TodoApp/schema.sql

                    # testcontainer deelt het netwerk van de testdatabase (localhost:3306)
                    docker create --name dotnet-tests --network container:test-db -w /src \
                      mcr.microsoft.com/dotnet/sdk:10.0 dotnet test TodoApp.Tests/TodoApp.Tests.csproj
                    docker cp . dotnet-tests:/src
                    docker start -a dotnet-tests
                '''
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
                // containers starten vanzelf opnieuw na een herstart van de VM
                sh 'docker update --restart unless-stopped todoapp todoappdb'
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

    post {
        always {
            sh 'docker rm -f test-db dotnet-tests 2>/dev/null || true'
        }
    }
}