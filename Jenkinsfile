pipeline {
    agent any

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp2"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'SCALE_ONLY'],
            description: 'Choose FULL_PIPELINE or SCALE_ONLY'
        )
        string(name: 'REPLICA_COUNT', defaultValue: '1', description: 'Replica count for frontend & backend')
    }

    stages {

        /* ---------------------------------------------- */
        /* CHECKOUT                                       */
        /* ---------------------------------------------- */
        stage('Checkout') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                git 'https://github.com/ThanujaRatakonda/kp2.git'
            }
        }

        /* ---------------------------------------------- */
        /* 1. BUILD DOCKER IMAGES                         */
        /* ---------------------------------------------- */
        stage('Build Docker Images') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                script {
                    def containers = [
                        [name: "frontend", folder: "frontend"],
                        [name: "backend", folder: "backend"]
                    ]

                    containers.each { c ->
                        echo "Building Docker image for ${c.name}..."
                        sh "docker build -t ${c.name}:${IMAGE_TAG} ./${c.folder}"
                    }
                }
            }
        }

        /* ---------------------------------------------- */
        /* 2. SCAN DOCKER IMAGES                          */
        /* ---------------------------------------------- */
        stage('Scan Docker Images') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                script {
                    def containers = ["frontend", "backend"]

                    containers.each { img ->
                        echo "Running Trivy scan for ${img}:${IMAGE_TAG}..."

                        sh """
                            trivy image ${img}:${IMAGE_TAG} \
                            --severity CRITICAL,HIGH \
                            --format json \
                            -o ${TRIVY_OUTPUT_JSON}
                        """

                        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

                        def vulnerabilities = sh(script: """
                            jq '[.Results[] |
                                 (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                                 (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                                ] | length' ${TRIVY_OUTPUT_JSON}
                        """, returnStdout: true).trim()

                        if (vulnerabilities.toInteger() > 0) {
                            error "CRITICAL/HIGH vulnerabilities found in ${img}!"
                        }
                    }
                }
            }
        }

        /* ---------------------------------------------- */
        /* 3. PUSH IMAGES TO HARBOR                       */
        /* ---------------------------------------------- */
        stage('Push Images to Harbor') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                script {
                    def containers = ["frontend", "backend"]

                    containers.each { img ->
                        def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/${img}:${IMAGE_TAG}"

                        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                            sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"

                            sh "docker tag ${img}:${IMAGE_TAG} ${fullImage}"
                            sh "docker push ${fullImage}"
                        }

                        sh "docker rmi ${img}:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        /* ---------------------------------------------- */
        /* APPLY K8s DEPLOYMENT                           */
        /* ---------------------------------------------- */
        stage('Apply Kubernetes Deployment') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                script {
                    sh """
                        sed -i 's/__IMAGE_TAG__/${IMAGE_TAG}/g' k8s/frontenddeployment.yaml
                        sed -i 's/__IMAGE_TAG__/${IMAGE_TAG}/g' k8s/backenddeployment.yaml
                    """

                    sh """
                        kubectl delete deployment frontend --ignore-not-found
                        kubectl delete deployment backend  --ignore-not-found
                        kubectl delete deployment database --ignore-not-found

                        kubectl delete service frontend --ignore-not-found
                        kubectl delete service backend  --ignore-not-found
                        kubectl delete service database --ignore-not-found
                    """

                    sh "kubectl apply -f k8s/"
                }
            }
        }

        /* ---------------------------------------------- */
        /* SCALE DEPLOYMENTS (ALWAYS RUNS)                */
        /* ---------------------------------------------- */
        stage('Scale Deployments') {
            steps {
                script {
                    echo "Scaling replicas to ${params.REPLICA_COUNT}"

                    sh "kubectl scale deployment frontend --replicas=${params.REPLICA_COUNT}"
                    sh "kubectl scale deployment backend  --replicas=${params.REPLICA_COUNT}"

                    sh "kubectl get deployments"
                }
            }
        }
    }
}

