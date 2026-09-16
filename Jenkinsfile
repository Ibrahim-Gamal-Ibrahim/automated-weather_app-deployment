pipeline {

    agent {
        label 'weatherapp-agent'
    }

    options {
        skipDefaultCheckout(true)
    }

    environment {
        DOCKERHUB_USER = 'ibrahimgamal10203040'
        NAMESPACE      = 'default'
    }

    stages {

        // =========================================================
        // 1. CHECKOUT
        // =========================================================

        stage('Checkout') {
            steps {
                container('git') {
                    git branch: 'main',
                        url: 'https://github.com/Ibrahim-Gamal-Ibrahim/automated-weather_app-deployment.git'
                }
            }
        }


        // =========================================================
        // 2. DOCKER HUB AUTHENTICATION
        // =========================================================

        stage('Prepare Docker Auth') {
            steps {
                container('buildkit') {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {

                        sh '''
                            mkdir -p ~/.docker

                            AUTH=$(printf "%s:%s" "$DOCKER_USER" "$DOCKER_PASS" \
                                | base64 | tr -d '\\n')

                            cat > ~/.docker/config.json <<EOF
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "$AUTH"
    }
  }
}
EOF
                        '''
                    }
                }
            }
        }


        // =========================================================
        // 3. BUILD AUTH
        // =========================================================

        stage('Build Auth') {
            steps {
                container('buildkit') {

                    sh '''
                        echo "Building Auth image..."

                        buildctl-daemonless.sh build \
                          --frontend dockerfile.v0 \
                          --local context=auth \
                          --local dockerfile=auth \
                          --output type=image,name=${DOCKERHUB_USER}/weatherapp-auth:latest,push=true
                    '''
                }
            }
        }


        // =========================================================
        // 4. BUILD UI
        // =========================================================

        stage('Build UI') {
            steps {
                container('buildkit') {

                    sh '''
                        echo "Building UI image..."

                        buildctl-daemonless.sh build \
                          --frontend dockerfile.v0 \
                          --local context=ui \
                          --local dockerfile=ui \
                          --output type=image,name=${DOCKERHUB_USER}/weatherapp-ui:latest,push=true
                    '''
                }
            }
        }


        // =========================================================
        // 5. BUILD WEATHER
        // =========================================================

        stage('Build Weather') {
            steps {
                container('buildkit') {

                    sh '''
                        echo "Building Weather image..."

                        buildctl-daemonless.sh build \
                          --frontend dockerfile.v0 \
                          --local context=weather \
                          --local dockerfile=weather \
                          --output type=image,name=${DOCKERHUB_USER}/weatherapp-weather:latest,push=true
                    '''
                }
            }
        }


        // =========================================================
        // 6. CREATE / UPDATE KUBERNETES SECRETS
        // =========================================================

        stage('Create Kubernetes Secrets') {
            steps {
                container('kubectl') {

                    withCredentials([

                        string(
                            credentialsId: 'mysql-root-password',
                            variable: 'MYSQL_ROOT_PASSWORD'
                        ),

                        string(
                            credentialsId: 'mysql-auth-password',
                            variable: 'MYSQL_AUTH_PASSWORD'
                        ),

                        string(
                            credentialsId: 'jwt-secret',
                            variable: 'JWT_SECRET'
                        ),

                        string(
                            credentialsId: 'weather-api-key',
                            variable: 'WEATHER_API_KEY'
                        ),

                        file(
                            credentialsId: 'weatherapp-tls-crt',
                            variable: 'TLS_CRT'
                        ),

                        file(
                            credentialsId: 'weatherapp-tls-key',
                            variable: 'TLS_KEY'
                        )
                    ]) {

                        sh '''

                            echo "Creating MySQL Secret..."

                            kubectl create secret generic mysql-secret \
                              --from-literal=root-password="$MYSQL_ROOT_PASSWORD" \
                              --from-literal=auth-password="$MYSQL_AUTH_PASSWORD" \
                              --dry-run=client \
                              -o yaml | kubectl apply \
                              -n ${NAMESPACE} \
                              -f -


                            echo "Creating Auth Secret..."

                            kubectl create secret generic auth-secret \
                              --from-literal=secret-key="$JWT_SECRET" \
                              --dry-run=client \
                              -o yaml | kubectl apply \
                              -n ${NAMESPACE} \
                              -f -


                            echo "Creating Weather Secret..."

                            kubectl create secret generic weather \
                              --from-literal=apikey="$WEATHER_API_KEY" \
                              --dry-run=client \
                              -o yaml | kubectl apply \
                              -n ${NAMESPACE} \
                              -f -


                            echo "Creating TLS Secret..."

                            kubectl create secret tls weatherapp-ui-tls \
                              --cert="$TLS_CRT" \
                              --key="$TLS_KEY" \
                              --dry-run=client \
                              -o yaml | kubectl apply \
                              -n ${NAMESPACE} \
                              -f -
                        '''
                    }
                }
            }
        }


        // =========================================================
        // 7. DEPLOY MYSQL
        // =========================================================

        stage('Deploy MySQL') {
            steps {
                container('kubectl') {

                    sh '''

                        echo "Deploying MySQL headless service..."

                        kubectl apply \
                          -f kubernetes/authentication/mysql/headless-service.yaml \
                          -n ${NAMESPACE}


                        echo "Deploying MySQL StatefulSet..."

                        kubectl apply \
                          -f kubernetes/authentication/mysql/statefulset.yaml \
                          -n ${NAMESPACE}
                    '''
                }
            }
        }


        // =========================================================
        // 8. INITIALIZE MYSQL
        // =========================================================

        stage('Initialize Database') {
            steps {
                container('kubectl') {

                    sh '''

                        echo "Removing old MySQL init Job if it exists..."

                        kubectl delete job mysql-init-job \
                          -n ${NAMESPACE} \
                          --ignore-not-found=true


                        echo "Creating MySQL init Job..."

                        kubectl apply \
                          -f kubernetes/authentication/mysql/init-job.yaml \
                          -n ${NAMESPACE}


                        echo "Waiting for database initialization..."

                        kubectl wait \
                          --for=condition=complete \
                          job/mysql-init-job \
                          -n ${NAMESPACE} \
                          --timeout=120s
                    '''
                }
            }
        }


        // =========================================================
        // 9. DEPLOY SERVICES
        // =========================================================

        stage('Deploy Services') {
            steps {
                container('kubectl') {

                    sh '''

                        kubectl apply \
                          -f kubernetes/authentication/service.yaml \
                          -n ${NAMESPACE}

                        kubectl apply \
                          -f kubernetes/weather/service.yaml \
                          -n ${NAMESPACE}

                        kubectl apply \
                          -f kubernetes/ui/service.yaml \
                          -n ${NAMESPACE}
                    '''
                }
            }
        }


        // =========================================================
        // 10. DEPLOY AUTH
        // =========================================================

        stage('Deploy Auth') {
            steps {
                container('kubectl') {

                    sh '''

                        if kubectl get deployment weatherapp-auth \
                            -n ${NAMESPACE} >/dev/null 2>&1
                        then
                            EXISTED=true
                        else
                            EXISTED=false
                        fi


                        kubectl apply \
                          -f kubernetes/authentication/deployment.yaml \
                          -n ${NAMESPACE}


                        if [ "$EXISTED" = "true" ]
                        then
                            echo "Restarting existing Auth deployment..."

                            kubectl rollout restart \
                              deployment/weatherapp-auth \
                              -n ${NAMESPACE}
                        else
                            echo "First Auth deployment."
                        fi
                    '''
                }
            }
        }


        // =========================================================
        // 11. DEPLOY WEATHER
        // =========================================================

        stage('Deploy Weather') {
            steps {
                container('kubectl') {

                    sh '''

                        if kubectl get deployment weatherapp-weather \
                            -n ${NAMESPACE} >/dev/null 2>&1
                        then
                            EXISTED=true
                        else
                            EXISTED=false
                        fi


                        kubectl apply \
                          -f kubernetes/weather/deployment.yaml \
                          -n ${NAMESPACE}


                        if [ "$EXISTED" = "true" ]
                        then
                            echo "Restarting existing Weather deployment..."

                            kubectl rollout restart \
                              deployment/weatherapp-weather \
                              -n ${NAMESPACE}
                        else
                            echo "First Weather deployment."
                        fi
                    '''
                }
            }
        }


        // =========================================================
        // 12. DEPLOY UI
        // =========================================================

        stage('Deploy UI') {
            steps {
                container('kubectl') {

                    sh '''

                        if kubectl get deployment release-name-weatherapp-ui \
                            -n ${NAMESPACE} >/dev/null 2>&1
                        then
                            EXISTED=true
                        else
                            EXISTED=false
                        fi


                        kubectl apply \
                          -f kubernetes/ui/deployment.yaml \
                          -n ${NAMESPACE}


                        if [ "$EXISTED" = "true" ]
                        then
                            echo "Restarting existing UI deployment..."

                            kubectl rollout restart \
                              deployment/release-name-weatherapp-ui \
                              -n ${NAMESPACE}
                        else
                            echo "First UI deployment."
                        fi
                    '''
                }
            }
        }


        // =========================================================
        // 13. DEPLOY INGRESS
        // =========================================================

        stage('Deploy Ingress') {
            steps {
                container('kubectl') {

                    sh '''

                        kubectl apply \
                          -f kubernetes/ui/ingress.yaml \
                          -n ${NAMESPACE}
                    '''
                }
            }
        }


        // =========================================================
        // 14. VERIFY ROLLOUTS
        // =========================================================

        stage('Verify Rollouts') {
            steps {
                container('kubectl') {

                    sh '''

                        echo "Waiting for Auth..."

                        kubectl rollout status \
                          deployment/weatherapp-auth \
                          -n ${NAMESPACE} \
                          --timeout=180s


                        echo "Waiting for Weather..."

                        kubectl rollout status \
                          deployment/weatherapp-weather \
                          -n ${NAMESPACE} \
                          --timeout=180s


                        echo "Waiting for UI..."

                        kubectl rollout status \
                          deployment/release-name-weatherapp-ui \
                          -n ${NAMESPACE} \
                          --timeout=180s
                    '''
                }
            }
        }


        // =========================================================
        // 15. FINAL STATUS
        // =========================================================

        stage('Verify Application') {
            steps {
                container('kubectl') {

                    sh '''

                        echo "===== DEPLOYMENTS ====="
                        kubectl get deployments -n ${NAMESPACE}

                        echo ""
                        echo "===== STATEFULSETS ====="
                        kubectl get statefulsets -n ${NAMESPACE}

                        echo ""
                        echo "===== PODS ====="
                        kubectl get pods -n ${NAMESPACE} -o wide

                        echo ""
                        echo "===== SERVICES ====="
                        kubectl get services -n ${NAMESPACE}

                        echo ""
                        echo "===== INGRESS ====="
                        kubectl get ingress -n ${NAMESPACE}
                    '''
                }
            }
        }
    }


    // =============================================================
    // PIPELINE RESULT
    // =============================================================

    post {

        success {
            echo '''
================================================
 WEATHER APPLICATION DEPLOYED SUCCESSFULLY
================================================
'''
        }

        failure {
            echo '''
================================================
 PIPELINE FAILED
 Check the failed stage logs.
================================================
'''
        }

        always {
            echo 'Weather application pipeline finished.'
        }
    }
}