pipeline {
    agent any

    parameters {
        choice(name: 'ACTION', choices: ['BUILD', 'SCAN', 'PUSH', 'DEPLOY_SCALE'], description: 'Choose action to perform')
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Provide IMAGE_TAG for scan, push, or deploy')
        string(name: 'REPLICA_COUNT', defaultValue: '1', description: 'Replica count for frontend, backend, database')
    }

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_2"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp2.git'
            }
        }

        stage('Build Docker Images') {
            when {
                expression { params.ACTION == 'BUILD' }
            }
            environment {
                IMAGE_TAG = "${env.BUILD_NUMBER}"
            }
            steps {
                script {
                    def containers = [
                        [name: "frontend", folder: "frontend"],
                        [name: "backend", folder: "backend"]
                    ]
                    containers.each { c ->
                        sh "docker build -t ${c.name}:${IMAGE_TAG} ./${c.folder}"
                    }
                }
            }
        }

        stage('Scan Docker Images') {
            when {
                expression { params.ACTION == 'SCAN' }
            }
            steps {
                script {
                    if (!params.IMAGE_TAG) {
                        error "IMAGE_TAG must be provided for scanning"
                    }
                    def containers = ["frontend", "backend"]
                    containers.each { c ->
                        sh """
                            trivy image ${c}:${params.IMAGE_TAG} \
                            --severity CRITICAL,HIGH \
                            --format json \
                            -o ${c}-${TRIVY_OUTPUT_JSON}
                        """
                        archiveArtifacts artifacts: "${c}-${TRIVY_OUTPUT_JSON}", fingerprint: true

                        def vulnerabilities = sh(script: """
                            jq '[.Results[] |
                                 (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                                 (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                                ] | length' ${c}-${TRIVY_OUTPUT_JSON}
                        """, returnStdout: true).trim()

                        if (vulnerabilities.toInteger() > 0) {
                            error "CRITICAL/HIGH vulnerabilities found in ${c}!"
                        }
                    }
                }
            }
        }

        stage('Push Docker Images') {
            when {
                expression { params.ACTION == 'PUSH' }
            }
            steps {
                script {
                    if (!params.IMAGE_TAG) {
                        error "IMAGE_TAG must be provided for pushing images"
                    }
                    def containers = ["frontend", "backend"]
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                        containers.each { c ->
                            def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/${c}:${params.IMAGE_TAG}"
                            sh "docker tag ${c}:${params.IMAGE_TAG} ${fullImage}"
                            sh "docker push ${fullImage}"
                            sh "docker rmi ${c}:${params.IMAGE_TAG} || true"
                        }
                    }
                }
            }
        }

        stage('Deploy & Scale') {
            when {
                expression { params.ACTION == 'DEPLOY_SCALE' }
            }
            steps {
                script {
                    if (!params.IMAGE_TAG) {
                        error "IMAGE_TAG must be provided for deploying/scaling"
                    }

                    echo "Updating YAMLs with IMAGE_TAG..."
                    sh """
                        sed -i 's/__IMAGE_TAG__/${params.IMAGE_TAG}/g' k8s/frontenddeployment.yaml
                        sed -i 's/__IMAGE_TAG__/${params.IMAGE_TAG}/g' k8s/backenddeployment.yaml
                    """

                    echo "Applying deployments and services..."
                    sh "kubectl apply -f k8s/"

                    echo "Scaling deployments..."
                    sh """
                        kubectl scale deployment frontend --replicas=${params.REPLICA_COUNT}
                        kubectl scale deployment backend  --replicas=${params.REPLICA_COUNT}
                        kubectl scale deployment database --replicas=${params.REPLICA_COUNT}
                        kubectl get deployments
                    """
                }
            }
        }

    }
}


