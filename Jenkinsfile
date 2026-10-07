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

// pipeline {
//     agent any

//     stages {
//         stage('Preparation') {
//             steps {
//                 catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
//                     sh 'docker compose down -v'
//                 }
//             }
//         }

//         stage('Build Containers') {
//             steps {
//                 sh 'docker compose build'
//             }
//         }

//         stage('Start Database') {
//             steps {
//                 // Start enkel de DB en wacht tot deze healthy is
//                 sh 'docker compose up -d todoappdb'
//                 sh 'docker compose ps'
//             }
//         }

//         stage('Unit Tests') {
//             steps {
//                 sh '''
//                     docker rm -f dotnet-tests || true
                    
//                     # Maak een tijdelijke runner container aan in hetzelfde netwerk als todoappdb
//                     docker create --name dotnet-tests \
//                         --network container:todoappdb \
//                         -e ConnectionStrings__TodoDb="Server=127.0.0.1;Port=3306;Database=todo_test_db;User=todo_usr;Password=letmeinplz;" \
//                         -w /src \
//                         mcr.microsoft.com/dotnet/sdk:10.0 dotnet test

//                     # Kopieer de broncode naar de container
//                     docker cp . dotnet-tests:/src

//                     # Voer de tests uit
//                     docker start -a dotnet-tests
//                 '''
//             }
//             post {
//                 always {
//                     sh 'docker rm -f dotnet-tests || true'
//                 }
//             }
//         }

//         stage('Deploy Application') {
//             steps {
//                 // Start de volledige stack (inclusief de webapp)
//                 sh 'docker compose up -d'
//             }
//         }

//         stage('Health Check') {
//             steps {
//                 sleep 10
//                 sh 'docker compose ps'
//                 sh 'curl -f http://172.17.0.1:5000/'
//             }
//         }
//     }

//     post {
//         failure {
//             echo 'Pipeline mislukt! Bekijk de logs voor details.'
//         }
//     }
// }

pipeline {
    agent any

    stages {

        // 1. Images bouwen (database met schema + webapp)
        stage('Build') {
            steps {
                sh 'docker compose build'
            }
        }

        // 2. Tijdelijke testdatabase starten (los van de echte database)
        stage('Start test database') {
            steps {
                sh '''
                    set -e

                    docker network create ci-test-network 2>/dev/null || true
                    docker rm -f todoapp-test-db 2>/dev/null || true

                    docker run -d \
                      --name todoapp-test-db \
                      --network ci-test-network \
                      -e MARIADB_ROOT_PASSWORD=sekrit \
                      -e MARIADB_DATABASE=todo_test_db \
                      -e MARIADB_USER=todo_usr \
                      -e MARIADB_PASSWORD=letmeinplz \
                      mariadb:11

                    for i in $(seq 1 20); do
                      if docker exec todoapp-test-db \
                        mariadb-admin ping -h 127.0.0.1 -uroot -psekrit --silent; then
                        echo "Test database is ready"
                        break
                      fi

                      echo "Waiting for MariaDB test database ($i/20)..."
                      sleep 3
                    done

                    if ! docker exec todoapp-test-db \
                      mariadb-admin ping -h 127.0.0.1 -uroot -psekrit --silent; then
                      echo "MariaDB test database did not become ready"
                      docker logs todoapp-test-db
                      exit 1
                    fi
                '''
            }
        }

        // 3. Schema in de testdatabase laden
        stage('Initialize test database') {
            steps {
                sh '''
                    set -e

                    docker exec -i todoapp-test-db \
                      mariadb -h 127.0.0.1 -uroot -psekrit todo_test_db \
                      < TodoApp/schema.sql

                    docker exec todoapp-test-db \
                      mariadb -h 127.0.0.1 -uroot -psekrit \
                      -e "SHOW TABLES FROM todo_test_db;"
                '''
            }
        }

        // 4. Controleren of de testdatabase bereikbaar is via het netwerk
        stage('Verify database from network') {
            steps {
                sh '''
                    set -e

                    docker run --rm \
                      --network ci-test-network \
                      mariadb:11 \
                      mariadb \
                        -h todoapp-test-db \
                        -P 3306 \
                        -utodo_usr \
                        -pletmeinplz \
                        todo_test_db \
                        -e "SELECT 1 AS database_connection_ok;"
                '''
            }
        }

        // 5. Unit tests uitvoeren in een tijdelijke container met de .NET SDK
        //    (docker cp i.p.v. een bind mount: de workspace bestaat enkel in de Jenkins-container)
        stage('Test') {
            steps {
                sh '''
                    set -e

                    TEST_CONTAINER=dotnet-test-runner

                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    docker create \
                      --name "$TEST_CONTAINER" \
                      --network ci-test-network \
                      -w /src \
                      -e ConnectionStrings__TodoDb="Server=todoapp-test-db;Port=3306;Database=todo_test_db;User=todo_usr;Password=letmeinplz;" \
                      mcr.microsoft.com/dotnet/sdk:10.0 \
                      dotnet test TodoApp.Tests/TodoApp.Tests.csproj

                    docker cp . "$TEST_CONTAINER":/src
                    docker start -a "$TEST_CONTAINER"
                    docker rm "$TEST_CONTAINER"
                '''
            }
        }

        // 6. Testdatabase opruimen
        stage('Cleanup test database') {
            steps {
                sh '''
                    docker rm -f todoapp-test-db || true
                    docker network rm ci-test-network || true
                '''
            }
        }

        // 7. Echte applicatie uitrollen
        //    Het schema zit al in het database-image (Dockerfile.db),
        //    en todoapp wacht via depends_on tot de database 'healthy' is.
        stage('Deploy') {
            steps {
                sh '''
                    set -e

                    docker compose down || true
                    docker compose up -d
                    docker compose ps
                '''
            }
        }

        // 8. Controleren of de uitgerolde app antwoordt
        stage('Acceptance test') {
            steps {
                sh '''
                    set -e

                    # a) Via het Docker-netwerk van het project
                    #    (netwerknaam = jobnaam in kleine letters + _default)
                    for i in $(seq 1 12); do
                      if docker run --rm \
                        --network dotnetdemopipeline_default \
                        curlimages/curl:8.12.1 \
                        -fsS http://todoapp:8080/ > /tmp/todoapp.html; then

                        echo "Todo-app is reachable from the Docker network"
                        grep "<title>" /tmp/todoapp.html
                        break
                      fi

                      echo "Waiting for todo-app ($i/12)..."
                      sleep 5

                      if [ "$i" -eq 12 ]; then
                        echo "Todo-app did not become reachable from the Docker network"
                        docker logs todoapp || true
                        exit 1
                      fi
                    done

                    # b) Via de gepubliceerde poort 5000 van de VM (Docker-host)
                    curl -fsS http://172.17.0.1:5000/ > /dev/null
                    echo "Todo-app is reachable on published port 5000"
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f todoapp-test-db 2>/dev/null || true'
            sh 'docker rm -f dotnet-test-runner 2>/dev/null || true'
            sh 'docker network rm ci-test-network 2>/dev/null || true'
        }
    }
}