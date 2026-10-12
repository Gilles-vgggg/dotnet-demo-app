node {
    stage('Checkout') {
        checkout scm
    }

    stage('Preparation') {
        sh '''
            docker compose down --remove-orphans || true
        '''
    }

    stage('Database') {
        sh '''
            echo "Starting MariaDB..."
            docker compose up -d todoappdb

            echo "Waiting for MariaDB..."
            for i in $(seq 1 30); do
                if docker exec todoappdb mariadb-admin ping \
                    -h localhost -uroot -psekrit --silent; then
                    echo "MariaDB is ready!"
                    break
                fi

                sleep 2
            done

            echo "Creating test database..."
            docker exec todoappdb mariadb -uroot -psekrit -e \
              "CREATE DATABASE IF NOT EXISTS todo_test_db;
               GRANT ALL PRIVILEGES ON todo_test_db.* TO 'todo_usr'@'%';
               FLUSH PRIVILEGES;"

            echo "Creating application database if necessary..."
            docker exec todoappdb mariadb -uroot -psekrit -e \
              "CREATE DATABASE IF NOT EXISTS todo_db;
               GRANT ALL PRIVILEGES ON todo_db.* TO 'todo_usr'@'%';
               FLUSH PRIVILEGES;"

            echo "Loading schemas..."
            cat TodoApp/schema.sql | \
              docker exec -i todoappdb mariadb \
              -utodo_usr -pletmeinplz todo_test_db

            cat TodoApp/schema.sql | \
              docker exec -i todoappdb mariadb \
              -utodo_usr -pletmeinplz todo_db
        '''
    }

    stage('Test') {
        sh '''
            echo "Running .NET tests..."

            docker run --rm \
              --network host \
              --volumes-from jenkins_server \
              -w "$WORKSPACE" \
              mcr.microsoft.com/dotnet/sdk:10.0 \
              dotnet test ./TodoApp.Tests/TodoApp.Tests.csproj
        '''
    }

    stage('Build') {
        sh '''
            docker compose build
        '''
    }

    stage('Deploy') {
        sh '''
            docker compose up -d
        '''
    }

    stage('Verify') {
        sh '''
            for i in $(seq 1 30); do
                if curl -fsS http://172.16.0.10:8081/ > /dev/null; then
                    echo "Todo application is reachable!"
                    exit 0
                fi

                echo "Waiting for Todo application..."
                sleep 2
            done

            echo "Application did not become reachable."
            docker compose ps
            docker compose logs --no-color --tail=100
            exit 1
        '''
    }
}