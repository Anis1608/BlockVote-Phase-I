pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command: ["cat"]
    tty: true

  - name: dind
    image: docker:dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
    args:
    - "--storage-driver=overlay2"
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent

  - name: jnlp
    image: jenkins/inbound-agent:3309.v27b_9314fd1a_4-1
    env:
    - name: JENKINS_AGENT_WORKDIR
      value: "/home/jenkins/agent"
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: workspace-volume

  volumes:
  - name: workspace-volume
    emptyDir: {}
"""
        }
    }

    environment {
        SONARQUBE = 'sonarqube-clg'
        SONAR_TOKEN = "sqp_51dc6dfb789de440cbc3320e8591365708d7018b"
        NEXUS_RAW = "http://nexus.imcc.com/repository/2401098-BlockVote-Anis-Khan/"
        NEXUS_USER = "student"
        NEXUS_PASS = "Imcc@2025"
    }

    stages {

        stage('CHECK') {
            steps {
                echo "DEBUG: BlockVote Pipeline Running"
            }
        }

        stage('SonarQube Scan') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv("sonarqube-clg") {
                        sh """
                            sonar-scanner \
                              -Dsonar.projectKey=blockvote-2401098 \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
                              -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                container('dind') {
                    sh """
                        docker build -t blockvote-backend:latest Backend/
                        docker build -t blockvote-frontend:latest Frontend/
                    """
                }
            }
        }

        stage('Export Docker Images') {
            steps {
                container('dind') {
                    sh """
                        docker save -o backend.tar blockvote-backend:latest
                        docker save -o frontend.tar blockvote-frontend:latest
                    """
                }
            }
        }

        stage('Upload to Nexus RAW Repo') {
            steps {
                sh """
                curl -u ${NEXUS_USER}:${NEXUS_PASS} --upload-file backend.tar ${NEXUS_RAW}/backend.tar
                curl -u ${NEXUS_USER}:${NEXUS_PASS} --upload-file frontend.tar ${NEXUS_RAW}/frontend.tar
                """
            }
        }

        stage('Auto Deployment (Docker Run)') {
            steps {
                container('dind') {
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
}
