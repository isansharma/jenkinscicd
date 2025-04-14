pipeline{
    agent {
        docker{
        image 'isansharma/maven-docker-3.8.1-17:1'
        args '--user root -v /var/run/docker.sock:/var/run/docker.sock ' // mount Docker socket to access the host's Docker daemon
    }
    }
    environment {
        DOCKER_IMAGE = 'isansharma/jenkinscicd'
        DOCKER_CREDENTIALS_ID = 'dockerhub'  // Jenkins credential ID
        DOCKERFILE_LOCATION = 'java-maven-sonar-argocd-helm-k8s/spring-boot-app'
        GIT_CREDENTIALS_ID = 'github' // Jenkins credentials ID
        REPO_URL = 'https://github.com/isansharma/jenkinscicd.git' // or HTTPS URL
        FILE_PATH = 'deployment.yml'
        BRANCH = 'main'
    }
    stages{
        stage('Checkout'){
            steps{
                git branch: 'main', url: 'https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero.git'
            }
        }
        stage('Build'){
            steps{
                sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn clean package'
            }
        }
        stage('SonarQube'){
            steps{
                withSonarQubeEnv(credentialsId: 'sonarqube', installationName: 'sonarqube') {
                    sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn sonar:sonar'
                }
            }
        }
        stage('BuildImage'){
            steps{
                script {
                    sh 'cd ${DOCKERFILE_LOCATION} && docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} . '
                    
                }
            }
        }
        stage('Docker'){
            steps{
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                        echo 'Logged in to Docker Hub'
                    }
            }
        }
    }
    stage('PushImage'){
        steps{
            script {
                    def dockerImage = docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}") 
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                        dockerImage.push()
                        // dockerImage.push('latest')
                    }
             }
         }
    }
    stage('DeleteImage'){
        steps{
            script {
                    sh "docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true"
            }
        }
    }
    stage('Testing Directory'){
        steps{
            sh " mkdir -p  manifest && cd manifest "
        }
    }
    stage('CloneManifestRepo'){
        steps{
            git branch: 'main', url: 'https://github.com/isansharma/jenkinscicd.git'
            // git credentialsId: "${env.GIT_CREDENTIALS_ID}",
            // url: "${env.REPO_URL}",
            // branch: 'main'
        }
    }
    stage('ModifyManifest'){
        steps{
            sh '''
                    IMAGE_TAG="isansharma/jenkinscicd:${BUILD_NUMBER}"
                    sed -i "s|isansharma/jenkinscicd:[^ ]*|$IMAGE_TAG|" ${FILE_PATH}
                '''
        }
    }
    stage('CommitAndPushChanges'){
        steps{
            withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config --global user.email "isansharma@gmail.com"
                        git config --global user.name "isansharma"
                        git config --global --add safe.directory /var/lib/jenkins/workspace/Jenkins-CICD

                        git add ${FILE_PATH}
                        git commit -m "Update image tag to build ${BUILD_NUMBER}" || echo "No changes to commit"

                        REPO_URL_WITH_TOKEN=$(echo ${REPO_URL} | sed "s|https://|https://${GITHUB_TOKEN}@|")
                        git push $REPO_URL_WITH_TOKEN HEAD:${BRANCH}
                    '''
                }
        }
    }
}
}
