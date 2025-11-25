pipeline {
    agent any

    environment {
        SONARQUBE = 'sonarqube-clg'  // Jenkins SonarQube server name
        NEXUS_RAW = "http://nexus.imcc.com/repository/2401098-BlockVote-Anis-Khan/"  // YOUR RAW repo
        NEXUS_USER = "student"
        NEXUS_PASS = "Imcc@2025"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Anis1608/BlockVote.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker build -t blockvote-backend:latest backend/'
                sh 'docker build -t blockvote-frontend:latest frontend/'
            }
        }

        stage('Export Docker Images (.tar)') {
            steps {
                sh 'docker save -o backend.tar blockvote-backend:latest'
                sh 'docker save -o frontend.tar blockvote-frontend:latest'
            }
        }

        stage('Upload to Nexus RAW Repo') {
            steps {
                sh """
                curl -u ${NEXUS_USER}:${NEXUS_PASS} \
                --upload-file backend.tar \
                ${NEXUS_RAW}/backend.tar

                curl -u ${NEXUS_USER}:${NEXUS_PASS} \
                --upload-file frontend.tar \
                ${NEXUS_RAW}/frontend.tar
                """
            }
        }

        stage('Auto Deployment on College Server') {
            steps {
                sh """
                docker load -i backend.tar
                docker stop blockvote-backend || true
                docker rm blockvote-backend || true
                docker run -d --name blockvote-backend -p 5000:5000 blockvote-backend:latest

                docker load -i frontend.tar
                docker stop blockvote-frontend || true
                docker rm blockvote-frontend || true
                docker run -d --name blockvote-frontend -p 3000:80 blockvote-frontend:latest
                """
            }
        }
    }
}
