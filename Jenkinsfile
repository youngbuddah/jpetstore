pipeline {
    agent { label 'mynode' }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/youngbuddah/jpetstore.git'
            }
        }
        stage('Maven Compile') {
            steps { sh 'mvn clean compile' }
        }
        stage('Maven Test') {
            steps { sh 'mvn test' }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Petshop \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=Petshop"
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build WAR File') {
            steps { sh 'mvn clean install -DskipTests=true' }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Build and Push to Docker Hub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t petshop ."
                        sh "docker tag petshop ironfang26/petshop:latest"
                        sh "docker push ironfang26/petshop:latest"
                    }
                }
            }
        }
        stage('TRIVY') {
            steps {
                sh "trivy image ironfang26/petshop:latest > trivy.txt"
                archiveArtifacts artifacts: 'trivy.txt'
            }
        }
        stage('Deploy to Container') {
            steps {
                sh 'docker rm -f pet1 || true'
                sh 'docker run -d --name pet1 -p 8080:8080 ironfang26/petshop:latest'
            }
        }
        stage('Terraform - Create EKS Cluster') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'
                    sh 'aws eks update-kubeconfig --region ap-south-1 --name cluster123'
                }
            }
        }
        stage('K8s Deploy') {
            steps { sh 'kubectl apply -f deployment.yaml' }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "${currentBuild.result}",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'abhaybendekar1996@gmail.com',
                attachmentsPattern: 'trivy.txt'
        }
    }
}

