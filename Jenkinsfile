pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node26'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE = "emlykf/netflix:${BUILD_NUMBER}"
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/emlykf/devsecops-netflix-k8s.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                dir('app') {
                    withSonarQubeEnv('sonar-server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                    }
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('app') {
                    sh "npm install -g yarn"
                    sh "yarn install --frozen-lockfile"
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dir('app') {
                    dependencyCheck additionalArguments: '--scan ./ --disableNodeAudit --nodeAuditSkipDevDependencies --nodePackageSkipDevDependencies', nvdCredentialsId: 'nvd-api-key', odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml', failedTotalCritical: 1, failedTotalHigh: 1, stopBuild: true
                }
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                dir('app') {
                    sh "trivy fs . > trivyfs.txt"
                    sh "trivy fs --skip-db-update --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ."
                }
            }
        }
        stage("Docker Build") {
            steps {
                dir('app') {
                    withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_KEY')]) {
                        sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_KEY -t $IMAGE .'
                    }
                }
            }
        }
        stage("TRIVY") {
            steps {
                sh 'trivy image $IMAGE > trivyimage.txt'
                sh 'trivy image --skip-db-update --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 $IMAGE'
            }
        }
        stage("Docker Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker push $IMAGE'
                    }
                }
            }
        }
        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@netflix-jenkins"
                        sed -i "s|image: emlykf/netflix:.*|image: $IMAGE|" kubernetes/deployment.yml
                        git add kubernetes/deployment.yml
                        git commit -m "Update image to $IMAGE"
                        git push https://$GIT_USER:$GIT_TOKEN@github.com/emlykf/devsecops-netflix-k8s.git HEAD:main
                    '''
                }
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                    "Build Number: ${env.BUILD_NUMBER}<br/>" +
                    "URL: ${env.BUILD_URL}<br/>",
                to: 'your-email@example.com',
                attachmentsPattern: 'app/trivyfs.txt,trivyimage.txt'
        }
    }
}