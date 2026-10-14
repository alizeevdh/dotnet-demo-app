pipeline {
    agent any

    stages {

        // 1. Image van de webapp bouwen (de database gebruikt het officiële image mariadb:11)
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

                    # De tests blijken verbinding te maken met de database 'todo_db'.
                    # Dit is een wegwerpcontainer, dus we maken die database hier ook aan
                    # (de echte database in de app-deployment blijft onaangeroerd).
                    docker exec todoapp-test-db \
                      mariadb -h 127.0.0.1 -uroot -psekrit \
                      -e "CREATE DATABASE IF NOT EXISTS todo_db; GRANT ALL PRIVILEGES ON todo_db.* TO 'todo_usr'@'%'; FLUSH PRIVILEGES;"

                    docker exec -i todoapp-test-db \
                      mariadb -h 127.0.0.1 -uroot -psekrit todo_db \
                      < TodoApp/schema.sql

                    docker exec todoapp-test-db \
                      mariadb -h 127.0.0.1 -uroot -psekrit \
                      -e "SHOW TABLES FROM todo_test_db; SHOW TABLES FROM todo_db;"
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

                    # --network container:... laat de testcontainer het netwerk van de testdatabase delen,
                    # zodat de tests de database bereiken op localhost:3306 (zoals in de README).
                    docker create \
                      --name "$TEST_CONTAINER" \
                      --network container:todoapp-test-db \
                      -w /src \
                      -e ConnectionStrings__TodoDb="Server=127.0.0.1;Port=3306;Database=todo_test_db;User=todo_usr;Password=letmeinplz;" \
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

        // 7. schema.sql op de Docker-host zetten.
        //    docker-compose.yml mount ./TodoApp/schema.sql als bind mount. Compose geeft aan de
        //    Docker-daemon (die op de VM draait) het pad van de Jenkins-workspace door:
        //    /var/jenkins_home/workspace/<job>/TodoApp/schema.sql. Dat pad bestaat op de VM niet,
        //    waardoor Docker er een lege MAP van maakt en MariaDB crasht.
        //    Oplossing: het bestand met een hulpcontainer op dat pad van de VM zetten.
        stage('Prepare schema on Docker host') {
            steps {
                sh '''
                    set -e

                    docker rm -f schema-copier 2>/dev/null || true

                    # Hulpcontainer met de workspace-map als bind mount (de map wordt op de VM aangemaakt)
                    docker run -d --name schema-copier \
                      --entrypoint sleep \
                      -v "$(pwd)/TodoApp:/dest" \
                      mariadb:11 120

                    # Eventuele lege map van een eerdere mislukte run opruimen
                    docker exec schema-copier rm -rf /dest/schema.sql
                    docker cp TodoApp/schema.sql schema-copier:/dest/schema.sql
                    docker exec schema-copier ls -l /dest/schema.sql

                    docker rm -f schema-copier
                '''
            }
        }

        // 8. Echte applicatie uitrollen met het ongewijzigde docker-compose.yml
        stage('Deploy') {
            steps {
                sh '''
                    set -e

                    docker compose down || true

                    if ! docker compose up -d; then
                      echo "docker compose up failed, logs of the database:"
                      docker logs todoappdb || true
                      exit 1
                    fi

                    docker compose ps
                '''
            }
        }

        // 9. Controleren of de uitgerolde app antwoordt
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
            sh 'docker rm -f schema-copier 2>/dev/null || true'
            sh 'docker network rm ci-test-network 2>/dev/null || true'
        }
    }
}