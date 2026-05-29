pipeline{
  agent any
  environment{
    DOCKER_TAG = ""
  }
  stages{
    stage('Initialize setting'){
      steps{
        script{
         currentBuild.displayName = "Current-build-is-#${currentBuild.number}"
         env.DOCKER_TAG = getDockerTag()
        }
      }
    }
    stage('Build and Push Docker Image'){
      steps{
        withCredentials([string(credentialsId: 'DockerHubPW', variable: 'dockerHubPwd')]) {
          sh "docker login -u manashchauhan -p ${dockerHubPwd}" 
          sh "docker buildx build --platform linux/amd64 -t manashchauhan/nodeapp:${env.DOCKER_TAG} --push ."
        }
      }
    }
    stage('Deploy to k8s'){
      steps{
        sh "chmod +x changeTag.sh"
        sh "./changeTag.sh ${DOCKER_TAG}"
        sshagent(['dev-server']) {
          sh "scp -o StrictHostKeyChecking=no node-app-pod.yml ec2-user@172.31.46.23:/home/ec2-user/"
          sh "ssh ec2-user@172.31.46.23 kubectl apply -f ."
      }
      }
    }
  }
}

def getDockerTag(){
  def dockerTag = sh(script: 'git rev-parse HEAD', returnStdout: true).trim().take(7)
  return dockerTag
}
