// Jenkinsfile for a multi-stage CI/CD pipeline
pipeline {
    agent any // Or 'agent { label 'your-jenkins-agent-label' }' if using agents

    // Define environment variables
    environment {
        // GitHub repository details
        GIT_REPO_URL = 'https://github.com/Testing-Devops-Org/java-hello-world.git'
        GIT_CREDENTIALS_ID = 'github-credentials' // From Jenkins credentials

        // Maven and build details
        M2_HOME = tool 'M3' // Referencing your Maven tool configuration in Jenkins
        JDK_HOME = tool 'JDK_17' // Referencing your JDK tool configuration in Jenkins
        MAVEN_TARGET_JAR = 'target/java-hello-world-1.0-SNAPSHOT.jar' // Update if your pom.xml changes
        ARTIFACT_ID = 'java-hello-world' // From pom.xml
        VERSION = '1.0.0' // Can be dynamically pulled from pom.xml in a real pipeline
        BUILD_NAME = "${env.JOB_NAME}-${env.BUILD_NUMBER}"

        // SonarCloud details
        SONARCLOUD_SCANNER_HOME = tool 'SonarScanner' 
        SONARCLOUD_URL = 'https://sonarcloud.io'
        SONARCLOUD_PROJECT_KEY = 'Testing-Devops-Org_java-hello-world' // From your SonarCloud project page
        SONARCLOUD_ORGANIZATION = 'testing-devops-org' // From your SonarCloud organization (all lowercase)
        SONARCLOUD_TOKEN_ID = 'sonarcloud-auth-token' // From Jenkins credentials

        // Docker Hub details
        DOCKER_HUB_USERNAME = 'harshadm25' // <-- REPLACE WITH YOUR ACTUAL DOCKER HUB USERNAME!
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-credentials' // From Jenkins credentials
        DOCKER_IMAGE_NAME = "${DOCKER_HUB_USERNAME}/java-hello-world" // Matches your Dockerfile

        // JFrog Artifactory details
        ARTIFACTORY_CREDENTIALS_ID = 'artifactory-credentials' // From Jenkins credentials
        ARTIFACTORY_SERVER_ID = 'my-artifactory' // Matches ID in Jenkins global config
        ARTIFACTORY_REPO_RELEASE = 'libs-release-local' // Artifactory local Maven repo for releases
        ARTIFACTORY_REPO_SNAPSHOT = 'libs-snapshot-local' // Artifactory local Maven repo for snapshots (optional for this pipeline)

        // EC2 Deployment details
        EC2_DEPLOY_USER = 'ubuntu'
        EC2_DEPLOY_HOST = '18.191.202.22' 
        EC2_DEPLOY_SSH_CREDENTIALS_ID = 'ec2-deployment-ssh-key' 
        APP_DEPLOY_PATH = '/opt/app' // Directory on EC2 for the JAR

        // EKS Deployment details (if applicable)
        EKS_CLUSTER_NAME = 'YOUR_EKS_CLUSTER_NAME' // <-- REPLACE WITH YOUR EKS CLUSTER NAME!
        AWS_REGION = 'us-east-1' // <-- REPLACE WITH YOUR AWS REGION!
        AWS_CREDENTIALS_ID = 'aws-credentials' // From Jenkins credentials (if not using EC2 role)
    }

    tools {
        maven 'M3'
        jdk 'JDK_17'
        tool(type: hudson.plugins.sonar.SonarRunnerInstallation, name: 'SonarScanner')ss
        // Add Docker if installed as a tool rather than via 'sagent any'
        // docker 'docker' // If you have a Docker tool configured
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out Git repository: ${GIT_REPO_URL}"
                git credentialsId: GIT_CREDENTIALS_ID, url: GIT_REPO_URL, branch: 'master'
            }
        }

        stage('Build') {
            steps {
                echo "Building with Maven..."
                sh "${M2_HOME}/bin/mvn clean install -DskipTests" // Clean, compile, package. Skip tests for faster build.
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                echo "Running SonarCloud analysis..."
                withCredentials([string(credentialsId: SONARCLOUD_TOKEN_ID, variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarCloudServer') { // This MUST match the name in Jenkins Global Config
                        sh "${SONARCLOUD_SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONARCLOUD_PROJECT_KEY} \
                            -Dsonar.organization=${SONARCLOUD_ORGANIZATION} \
                            -Dsonar.host.url=${SONARCLOUD_URL} \
                            -Dsonar.token=${SONAR_TOKEN}" // Using the token from credentials
                    }
                }
            }
            post {
                // Optional: Gate the pipeline based on SonarCloud quality gate status
                // This pauses the pipeline until SonarCloud analysis is done and quality gate passes.
                success {
                    echo "Waiting for SonarCloud Quality Gate status..."
                    timeout(time: 5, unit: 'MINUTES') { // Adjust timeout as needed
                        withSonarQubeEnv('SonarCloudServer') {
                            sh "${SONARCLOUD_SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=${SONARCLOUD_PROJECT_KEY} -Dsonar.organization=${SONARCLOUD_ORGANIZATION} -Dsonar.host.url=${SONARCLOUD_URL} -Dsonar.token=${SONAR_TOKEN} org.sonarqube.cli.SonarQubeCli:sonar:sonar"
                        }
                    }
                }
            }
        }

        stage('Publish to Artifactory') {
            steps {
                echo "Publishing artifact to Artifactory..."
                script {
                    def server = Artifactory.server(ARTIFACTORY_SERVER_ID)
                    def buildInfo = Artifactory.newBuildInfo()
                    buildInfo.name = BUILD_NAME
                    buildInfo.number = env.BUILD_NUMBER

                    def rtMaven = Artifactory.newMavenBuild()
                    rtMaven.tool = 'M3' // Maven tool ID
                    rtMaven.jdk = 'JDK_17' // JDK tool ID

                    rtMaven.deployer.deploy artifacts: fileList(MAVEN_TARGET_JAR), \
                        buildInfo: buildInfo, \
                        repository: ARTIFACTORY_REPO_RELEASE, \
                        serverId: ARTIFACTORY_SERVER_ID // Use the configured server ID

                    server.publishBuildInfo buildInfo // Publish build info to Artifactory
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo "Building Docker image..."
                script {
                    docker.withRegistry("https://index.docker.io/v1/", DOCKER_HUB_CREDENTIALS_ID) {
                        def customImage = docker.build "${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}", "-f Dockerfile ."
                        customImage.push()
                        customImage.push('latest') // Push as latest as well
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "Deploying to EC2 instance: ${EC2_DEPLOY_HOST}"
                sshPublisher(
                    publishers: [
                        sshPublisherEntry(
                            configName: 'EC2-Deployment-Server', // This MUST match the name in Jenkins Global Config
                            transfers: [
                                sshTransfer(
                                    sourceFiles: MAVEN_TARGET_JAR,
                                    removePrefix: 'target/', // Remove the 'target/' part from the path
                                    remoteDirectory: APP_DEPLOY_PATH,
                                    execCommand: """
                                        sudo systemctl stop myapp.service || true # Stop if running
                                        sudo cp ${APP_DEPLOY_PATH}/${ARTIFACT_ID}-${VERSION}.jar ${APP_DEPLOY_PATH}/ # Copy in place
                                        sudo chown ${EC2_DEPLOY_USER}:${EC2_DEPLOY_USER} ${APP_DEPLOY_PATH}/${ARTIFACT_ID}-${VERSION}.jar
                                        sudo systemctl daemon-reload
                                        sudo systemctl start myapp.service
                                        echo "Application deployed and service restarted."
                                    """
                                )
                            ]
                        )
                    ]
                )
            }
            post {
                success {
                    echo "EC2 deployment completed successfully for ${ECAY_DEPLOY_HOST}."
                    // Optional: Add a health check here if your app exposes one
                    // sh "curl -f http://${EC2_DEPLOY_HOST}:8080/health || exit 1"
                }
                failure {
                    echo "EC2 deployment failed for ${EC2_DEPLOY_HOST}."
                }
            }
        }

        stage('Deploy to EKS') {
            // This stage is optional. Uncomment and configure if you have an EKS cluster ready.
            // If you don't have an EKS cluster, you can comment out or delete this stage.
            when {
                expression { return env.EKS_CLUSTER_NAME != 'YOUR_EKS_CLUSTER_NAME' } // Only run if EKS_CLUSTER_NAME is set
            }
            steps {
                echo "Deploying to EKS cluster: ${EKS_CLUSTER_NAME} in region ${AWS_REGION}"
                withAWS(credentials: AWS_CREDENTIALS_ID, region: AWS_REGION) { // Use IAM role for EC2 if no separate credentials
                    sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                    kubectl apply -f k8s/deployment.yaml
                    kubectl rollout status deployment/java-hello-world-deployment
                    """
                }
            }
            post {
                success {
                    echo "EKS deployment completed successfully for ${EKS_CLUSTER_NAME}."
                    echo "To check the service, run: kubectl get svc java-hello-world-service -o wide"
                }
                failure {
                    echo "EKS deployment failed for ${EKS_CLUSTER_NAME}."
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished for ${env.JOB_NAME}-${env.BUILD_NUMBER}"
            // Clean up workspace
            cleanWs()
        }
        success {
            echo "Pipeline succeeded!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}