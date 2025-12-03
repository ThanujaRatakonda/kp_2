pipeline {
    agent any

    parameters {
        choice(name: 'ACTION', choices: ['BUILD', 'SCAN', 'PUSH', 'DEPLOY_SCALE'], description: 'Choose what to run')
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Required for SCAN, PUSH, DEPLOY')
        string(name: 'REPLICA_COUNT', defaultValue: '1', description: 'Replica count for DEPLOY_SCALE')
    }

    environment {
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_2"
        TRIVY_OUTPUT_JSON = "trivy.json"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp2.git'
            }
        }

        /* ======================= BUILD ======================= */
        stage('Build Docker Images') {
            when { expression { params.ACTION == 'BUILD' } }
            steps {
                script {
                    def tag = env.BUILD_NUMBER

                    sh "docker build -t frontend:${tag} ./frontend"
                    sh "docker build -t backend:${tag} ./backend"

                    echo "BUILD COMPLETE → IMAGE_TAG=${tag}"
                }
            }
        }

        /* ======================= SCAN ======================= */
        stage('Scan Docker Images') {
            when { expression { params.ACTION == 'SCAN' } }
            steps {
                script {
                    if (!params.IMAGE_TAG) { error "IMAGE_TAG must be provided for SCAN" }

                    def imgs = ["frontend", "backend"]
                    imgs.each { img ->
                        sh """
                            trivy image ${img}:${params.IMAGE_TAG} \
                            --severity CRITICAL,HIGH \
                            --format json \
                            -o ${img}-scan.json
                        """
                        archiveArtifacts artifacts: "${img}-scan.json", fingerprint: true
                    }
                }
            }
        }

        /* ======================= PUSH ======================= */
        stage('Push Docker Images') {
            when { expression { params.ACTION == 'PUSH' } }
            steps {
                script {
                    if (!params.IMAGE_TAG) { error "IMAGE_TAG must be provided for PUSH" }

                    def imgs = ["frontend", "backend"]

                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'U', passwordVariable: 'P')]) {
                        sh "echo \$P | docker login ${HARBOR_URL} -u \$U --password-stdin"

                        imgs.each { img ->
                            def full = "${HARBOR_URL}/${HARBOR_PROJECT}/${img}:${params.IMAGE_TAG}"
                            sh "docker tag ${img}:${params.IMAGE_TAG} ${full}"
                            sh "docker push ${full}"
                        }
                    }
                }
            }
        }

        /* ======================= DEPLOY & SCALE ======================= */
        stage('Deploy & Scale') {
            when { expression { params.ACTION == 'DEPLOY_SCALE' } }
            steps {
                script {
                    if (!params.IMAGE_TAG) { error "IMAGE_TAG must be provided for DEPLOY_SCALE" }

                    sh """
                        sed -i 's/__IMAGE_TAG__/${params.IMAGE_TAG}/g' k8s/frontenddeployment.yaml
                        sed -i 's/__IMAGE_TAG__/${params.IMAGE_TAG}/g' k8s/backenddeployment.yaml
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


