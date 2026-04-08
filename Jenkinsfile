pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'iliasaas/demo-java-app'
        APP_DIR      = 'demo-java-app'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Read Version') {
            steps {
                dir("${APP_DIR}") {
                    script {
                        env.VERSION = sh(
                            script: "grep -m1 '<version>' pom.xml | sed 's/.*<version>//' | sed 's/<\\/version>.*//'",
                            returnStdout: true
                        ).trim()
                        echo "==> Building version: v${env.VERSION}"
                    }
                }
            }
        }

        stage('Maven Build') {
            steps {
                dir("${APP_DIR}") {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                dir("${APP_DIR}") {
                    sh 'mvn test'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir("${APP_DIR}") {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            mvn sonar:sonar \
                                -Dsonar.projectKey=demo-java-app \
                                -Dsonar.projectName='Demo Java App'
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir("${APP_DIR}") {
                    script {
                        dockerImage = docker.build("${DOCKER_IMAGE}:v${env.VERSION}")
                    }
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        dockerImage.push("v${env.VERSION}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                script {
                    sh """
                        sed -i 's|githubID:.*|githubID: iliasaas|' helm/app/values.yaml
                        sed -i 's|tag:.*|tag: v${env.VERSION}|' helm/app/values.yaml
                    """
                }
            }
        }

        stage('Push Helm Update') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh """
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins CI"
                        git add helm/app/values.yaml
                        git commit -m "[Jenkins] Update image tag to v${env.VERSION}" || true
                        git push https://\${GIT_USER}:\${GIT_PASS}@github.com/IliasAAs/devops-demo-project.git HEAD:main
                    """
                }
            }
        }
    }

    post {
        success {
            echo "==> Pipeline v${env.VERSION} SUCCESS"
        }
        failure {
            echo '==> Pipeline FAILED'
        }
    }
}
