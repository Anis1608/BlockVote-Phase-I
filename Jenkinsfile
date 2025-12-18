pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:

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
        // PATH-BASED DOCKER REGISTRY
        REGISTRY_HOST = "nexus.imcc.com:8085"
        REGISTRY_PATH = "blockvote-2401098"

        BACKEND_IMAGE = "blockvote-backend"
        FRONTEND_IMAGE = "blockvote-frontend"

        // Kubernetes namespace (change if required)
        NAMESPACE = "blockvote-2401098"
    }

    stages {

        stage('CHECK') {
            steps {
                echo "DEBUG: BlockVote Pipeline Started"
            }
        }

        stage('Build Backend Image') {
            steps {
                container('dind') {
                    sh """
                    docker build -t ${BACKEND_IMAGE}:${BUILD_NUMBER} backend
                    """
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                container('dind') {
                    sh """
                    docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} frontend
                    """
                }
            }
        }

        stage('Login to Nexus') {
            steps {
                container('dind') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'student',
                        passwordVariable: 'Imcc@2025'
                    )]) {
                        sh """
                        docker login ${REGISTRY_HOST} -u $USER -p $PASS
                        """
                    }
                }
            }
        }

        stage('Push Images to Nexus') {
            steps {
                container('dind') {
                    sh """
                    docker tag ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                      ${REGISTRY_HOST}/${REGISTRY_PATH}/${BACKEND_IMAGE}:${BUILD_NUMBER}

                    docker tag ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                      ${REGISTRY_HOST}/${REGISTRY_PATH}/${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    docker push ${REGISTRY_HOST}/${REGISTRY_PATH}/${BACKEND_IMAGE}:${BUILD_NUMBER}
                    docker push ${REGISTRY_HOST}/${REGISTRY_PATH}/${FRONTEND_IMAGE}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('dind') {
                    sh """
                    sed -i 's|BACKEND_TAG|${BUILD_NUMBER}|g' k8s/backend-deployment.yaml
                    sed -i 's|FRONTEND_TAG|${BUILD_NUMBER}|g' k8s/frontend-deployment.yaml

                    kubectl apply -f k8s/ -n ${NAMESPACE}
                    """
                }
            }
        }
    }
}
