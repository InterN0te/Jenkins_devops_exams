pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS = credentials('DOCKER_USERNAME_PASSWD')
        KUBECONFIG_CREDENTIALS = 'config'
        CAST_SERVICE_IMAGE = "${DOCKER_CREDENTIALS_USR}/cast-service"
        MOVIE_SERVICE_IMAGE = "${DOCKER_CREDENTIALS_USR}/movie-service"
    }

    stages {
        stage('Determine Tag and Namespace') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'master') {
                        env.IMAGE_TAG = 'latest'
                        env.NAMESPACE = 'staging'
                        env.INGRESS_HOST = 'datascientest-staging.debauchez.fr'
                        env.PROD_NAMESPACE = 'prod'
                        env.PROD_INGRESS_HOST = 'datascientest.debauchez.fr'
                    } else {
                        env.IMAGE_TAG = 'dev-latest'
                        env.NAMESPACE = 'dev'
                        env.INGRESS_HOST = 'datascientest-dev.debauchez.fr'
                        env.QA_NAMESPACE = 'qa'
                        env.QA_INGRESS_HOST = 'datascientest-qa.debauchez.fr'
                    }
                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Image Tag: ${env.IMAGE_TAG}"
                    echo "Initial Namespace: ${env.NAMESPACE}"
                    echo "Ingress Host: ${env.INGRESS_HOST}"
                }
            }
        }

        stage('Build Cast Service') {
            steps {
                script {
                    dir('cast-service') {
                        sh "docker build -t ${CAST_SERVICE_IMAGE}:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('Build Movie Service') {
            steps {
                script {
                    dir('movie-service') {
                        sh "docker build -t ${MOVIE_SERVICE_IMAGE}:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                script {
                    sh """
                        echo \${DOCKER_CREDENTIALS_PSW} | docker login -u \${DOCKER_CREDENTIALS_USR} --password-stdin
                        docker push ${CAST_SERVICE_IMAGE}:${IMAGE_TAG}
                        docker push ${MOVIE_SERVICE_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Prepare Helm Chart') {
            steps {
                script {
                    sh '''
                        cp -r charts charts-temp
                        sed -i '/containers:/a\\          {{- if .Values.secretName }}\\n          envFrom:\\n            - secretRef:\\n                name: {{ .Values.secretName }}\\n          {{- end }}\\n          {{- if .Values.env }}\\n          env:' charts-temp/templates/deployment.yaml
                        sed -i '/env:/a\\            {{- range $key, $value := .Values.env }}\\n            - name: {{ $key }}\\n              value: {{ $value | quote }}\\n            {{- end }}\\n          {{- end }}' charts-temp/templates/deployment.yaml
                    '''
                }
            }
        }

        stage('Deploy to Initial Namespace') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG')]) {
                        sh """
                            kubectl create namespace ${env.NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                            helm upgrade --install cast-service ./charts-temp \\
                                --namespace ${env.NAMESPACE} \\
                                --set image.repository=${CAST_SERVICE_IMAGE} \\
                                --set image.tag=${IMAGE_TAG} \\
                                --set secretName=cast-db-secret \\
                                --set ingress.enabled=true \\
                                --set ingress.className=traefik \\
                                --set 'ingress.hosts[0].host'=${env.INGRESS_HOST} \\
                                --set 'ingress.hosts[0].paths[0].path'=/api/v1/casts \\
                                --set 'ingress.hosts[0].paths[0].pathType'=Prefix \\
                                --set service.type=NodePort \\
                                --set service.port=80 \\
                                --set service.targetPort=8000 \\
                                --set fullnameOverride=cast-service

                            helm upgrade --install movie-service ./charts-temp \\
                                --namespace ${env.NAMESPACE} \\
                                --set image.repository=${MOVIE_SERVICE_IMAGE} \\
                                --set image.tag=${IMAGE_TAG} \\
                                --set secretName=movie-db-secret \\
                                --set 'env.CAST_SERVICE_HOST_URL'="http://cast-service/api/v1/casts/" \\
                                --set ingress.enabled=true \\
                                --set ingress.className=traefik \\
                                --set 'ingress.hosts[0].host'=${env.INGRESS_HOST} \\
                                --set 'ingress.hosts[0].paths[0].path'=/api/v1/movies \\
                                --set 'ingress.hosts[0].paths[0].pathType'=Prefix \\
                                --set service.type=NodePort \\
                                --set service.port=80 \\
                                --set service.targetPort=8000 \\
                                --set fullnameOverride=movie-service
                        """
                    }
                }
            }
        }

        stage('Manual Approval for Production') {
            when {
                branch 'master'
            }
            steps {
                script {
                    input message: 'Deploy to Production?', ok: 'Deploy to Prod'
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'master'
            }
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG')]) {
                        sh """
                            kubectl create namespace ${env.PROD_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                            helm upgrade --install cast-service ./charts-temp \\
                                --namespace ${env.PROD_NAMESPACE} \\
                                --set image.repository=${CAST_SERVICE_IMAGE} \\
                                --set image.tag=${IMAGE_TAG} \\
                                --set secretName=cast-db-secret \\
                                --set ingress.enabled=true \\
                                --set ingress.className=traefik \\
                                --set 'ingress.hosts[0].host'=${env.PROD_INGRESS_HOST} \\
                                --set 'ingress.hosts[0].paths[0].path'=/api/v1/casts \\
                                --set 'ingress.hosts[0].paths[0].pathType'=Prefix \\
                                --set service.type=NodePort \\
                                --set service.port=80 \\
                                --set service.targetPort=8000 \\
                                --set fullnameOverride=cast-service

                            helm upgrade --install movie-service ./charts-temp \\
                                --namespace ${env.PROD_NAMESPACE} \\
                                --set image.repository=${MOVIE_SERVICE_IMAGE} \\
                                --set image.tag=${IMAGE_TAG} \\
                                --set secretName=movie-db-secret \\
                                --set 'env.CAST_SERVICE_HOST_URL'="http://cast-service/api/v1/casts/" \\
                                --set ingress.enabled=true \\
                                --set ingress.className=traefik \\
                                --set 'ingress.hosts[0].host'=${env.PROD_INGRESS_HOST} \\
                                --set 'ingress.hosts[0].paths[0].path'=/api/v1/movies \\
                                --set 'ingress.hosts[0].paths[0].pathType'=Prefix \\
                                --set service.type=NodePort \\
                                --set service.port=80 \\
                                --set service.targetPort=8000 \\
                                --set fullnameOverride=movie-service
                        """
                    }
                }
            }
        }

        stage('Manual Approval for QA') {
            when {
                not {
                    branch 'master'
                }
            }
            steps {
                script {
                    input message: 'Deploy to QA?', ok: 'Deploy to QA'
                }
            }
        }

        stage('Deploy to QA') {
            when {
                not {
                    branch 'master'
                }
            }
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS}", variable: 'KUBECONFIG')]) {
                        sh """
                            kubectl create namespace ${env.QA_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                            helm upgrade --install cast-service ./charts-temp \\
                                --namespace ${env.QA_NAMESPACE} \\
                                --set image.repository=${CAST_SERVICE_IMAGE} \\
                                --set image.tag=${IMAGE_TAG} \\
                                --set secretName=cast-db-secret \\
                                --set ingress.enabled=true \\
                                --set ingress.className=traefik \\
                                --set 'ingress.hosts[0].host'=${env.QA_INGRESS_HOST} \\
                                --set 'ingress.hosts[0].paths[0].path'=/api/v1/casts \\
                                --set 'ingress.hosts[0].paths[0].pathType'=Prefix \\
                                --set service.type=NodePort \\
                                --set service.port=80 \\
                                --set service.targetPort=8000 \\
                                --set fullnameOverride=cast-service

                            helm upgrade --install movie-service ./charts-temp \\
                                --namespace ${env.QA_NAMESPACE} \\
                                --set image.repository=${MOVIE_SERVICE_IMAGE} \\
                                --set image.tag=${IMAGE_TAG} \\
                                --set secretName=movie-db-secret \\
                                --set 'env.CAST_SERVICE_HOST_URL'="http://cast-service/api/v1/casts/" \\
                                --set ingress.enabled=true \\
                                --set ingress.className=traefik \\
                                --set 'ingress.hosts[0].host'=${env.QA_INGRESS_HOST} \\
                                --set 'ingress.hosts[0].paths[0].path'=/api/v1/movies \\
                                --set 'ingress.hosts[0].paths[0].pathType'=Prefix \\
                                --set service.type=NodePort \\
                                --set service.port=80 \\
                                --set service.targetPort=8000 \\
                                --set fullnameOverride=movie-service
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                sh 'docker logout || true'
                sh 'rm -rf charts-temp || true'
            }
        }
        success {
            echo "Pipeline completed successfully! Deployed to ${env.NAMESPACE}"
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}
