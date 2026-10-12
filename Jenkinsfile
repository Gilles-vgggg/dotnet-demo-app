node {
    stage('Checkout') {
        checkout scm
    }

    stage('Test') {
        sh '''
            docker run --rm \
              -v "$WORKSPACE":/src \
              -w /src \
              mcr.microsoft.com/dotnet/sdk:10.0 \
              dotnet test
        '''
    }

    stage('Preparation') {
        sh '''
            docker compose down --remove-orphans || true
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