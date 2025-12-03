pipeline {
    agent any

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp2"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
    }

    parameters {
        choice(name: 'ACTION', choices: ['full', 'scale'], description: 'Run full pipeline or only scale')
        string(name: 'REPLICA_COUNT', defaultValue: '1', description: 'Replica count for frontend & backend')
    }

    stages {

        stage('Checkout') {
            when { expression { params.ACTION == 'full' } }
            steps {
                git 'https://github.com/ThanujaRatakonda/kp2.git'
            }
        }

        stage('Build, Scan & Push Docker Images') {
            when { expression { params.ACTION == 'full' } }
            steps {
                script {
                    def containers = [
                        [name: "frontend", folder: "frontend"],
                        [name: "backend", folder: "backend"]
                    ]

                    containers.each { c ->

                        def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/${c.name}:${IMAGE_TAG}"

                        echo "Building Docker image for ${c.name}..."
                        sh "docker build -t ${c.name}:${IMAGE_TAG} ./${c.folder}"

                        echo "Running Trivy scan for ${c.name}..."
                        sh """
                            trivy image ${c.name}:${IMAGE_TAG} \
                            --severity CRITICAL,HIGH \
                            --format json \
                            -o ${TRIVY_OUTPUT_JSON}
                        """

                        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

                        def vulnerabilities = sh(
                            script: """
                                jq '[.Results[] |
                                     (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                                    ] | length' ${TRIVY_OUTPUT_JSON}
                            """,
                            returnStdout: true
                        ).trim()

                        if (vulnerabilities.toInteger() > 0) {
                            error "CRITICAL/HIGH vulnerabilities found in ${c.name}!"
                        }

                        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                            sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                            sh "docker tag ${c.name}:${IMAGE_TAG} ${fullImage}"
                            sh "docker push ${fullImage}"
                        }

                        sh "docker rmi ${c.name}:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        stage('Apply Kubernetes Deployment') {
            when { expression { params.ACTION == 'full' } }
            steps {
                script {
                    echo "Updating YAMLs with IMAGE_TAG..."
                    sh """
                        sed -i 's/__IMAGE_TAG__/${IMAGE_TAG}/g' k8s/frontenddeployment.yaml
                        sed -i 's/__IMAGE_TAG__/${IMAGE_TAG}/g' k8s/backenddeployment.yaml
                    """

                    echo "Deleting OLD deployments..."
                    sh """
                        kubectl delete deployment frontend --ignore-not-found
                        kubectl delete deployment backend  --ignore-not-found
                        kubectl delete deployment database --ignore-not-found

                        kubectl delete service frontend --ignore-not-found
                        kubectl delete service backend  --ignore-not-found
                        kubectl delete service database --ignore-not-found
                    """

                    echo "Applying new deployments..."
                    sh "kubectl apply -f k8s/"
                }
            }
        }

        stage('Scale Deployments') {
            // This stage ALWAYS runs (both full & scale)
            steps {
                script {
                    sh "kubectl scale deployment frontend --replicas=${params.REPLICA_COUNT}"
                    sh "kubectl scale deployment backend  --replicas=${params.REPLICA_COUNT}"

                    echo "Deployment Status:"
                    sh "kubectl get deployments"
                }
            }
        }
    }
}
