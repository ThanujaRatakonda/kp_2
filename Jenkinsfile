pipeline {
    agent any

    parameters {
        choice(name: 'ACTION', choices: ['BUILD', 'SCAN', 'PUSH', 'DEPLOY_SCALE'], description: 'Choose pipeline action')
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Required only for SCAN/PUSH/DEPLOY')
        string(name: 'REPLICA_COUNT', defaultValue: '1', description: 'Replica count for Deploy Scale')
    }

    environment {
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_2"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp2.git'
            }
        }

        /* =======================
           1. BUILD
           =======================*/
        stage('Build Images') {
            when {
                anyOf {
                    expression { params.ACTION == 'BUILD' }      // Run in full pipeline
                    expression { params.ACTION == 'BUILD_ONLY' } // (optional)
                }
            }
            steps {
                script {
                    env.IMAGE_TAG = env.BUILD_NUMBER

                    sh "docker build -t frontend:${IMAGE_TAG} ./frontend"
                    sh "docker build -t backend:${IMAGE_TAG} ./backend"
                }
            }
        }

        /* =======================
           2. SCAN
           =======================*/
        stage('Scan Images') {
            when {
                anyOf {
                    expression { params.ACTION == 'BUILD' }  // include scan in full flow
                    expression { params.ACTION == 'SCAN' }   // manual scan
                }
            }
            steps {
                script {
                    def tag = (params.ACTION == 'BUILD') ? IMAGE_TAG : params.IMAGE_TAG

                    ["frontend", "backend"].each { img ->
                        sh """
                            trivy image ${img}:${tag} \
                                --severity CRITICAL,HIGH \
                                --format json -o ${img}-scan.json
                        """
                        archiveArtifacts artifacts: "${img}-scan.json", fingerprint: true
                    }
                }
            }
        }

        /* =======================
           3. PUSH
           =======================*/
        stage('Push Images') {
            when {
                anyOf {
                    expression { params.ACTION == 'BUILD' } // run in full flow
                    expression { params.ACTION == 'PUSH' }  // manual push
                }
            }
            steps {
                script {
                    def tag = (params.ACTION == 'BUILD') ? IMAGE_TAG : params.IMAGE_TAG

                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'U', passwordVariable: 'P')]) {
                        sh "echo \$P | docker login ${HARBOR_URL} -u \$U --password-stdin"

                        ["frontend", "backend"].each { img ->
                            def full = "${HARBOR_URL}/${HARBOR_PROJECT}/${img}:${tag}"
                            sh "docker tag ${img}:${tag} ${full}"
                            sh "docker push ${full}"
                        }
                    }
                }
            }
        }

        /* =======================
           4. DEPLOY + SCALE
           =======================*/
        stage('Deploy & Scale') {
            when {
                anyOf {
                    expression { params.ACTION == 'BUILD' }        // include deploy in full flow
                    expression { params.ACTION == 'DEPLOY_SCALE' } // manual deploy
                }
            }
            steps {
                script {
                    def tag = (params.ACTION == 'BUILD') ? IMAGE_TAG : params.IMAGE_TAG

                    sh """
                        sed -i 's/__IMAGE_TAG__/${tag}/g' k8s/frontenddeployment.yaml
                        sed -i 's/__IMAGE_TAG__/${tag}/g' k8s/backenddeployment.yaml
                    """

                    sh "kubectl apply -f k8s/"

                    sh """
                        kubectl scale deployment frontend --replicas=${params.REPLICA_COUNT}
                        kubectl scale deployment backend --replicas=${params.REPLICA_COUNT}
                    """

                    sh "kubectl get deploy"
                }
            }
        }
    }
}
