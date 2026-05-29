pipeline{
  agent any
  environment{
    DOCKER_TAG = getDockerTag()
  }
  stages{
    stage('Initialize setting'){
      steps{
        script{
         currentBuild.displayName = "Current-build-is-#${currentBuild.number}" 
        }
      }
    }
    stage('Build Docker Image'){
      steps{
       sh "docker build . -t manashchauhan/nodeapp/v1:${DOCKER_TAG}" 
      }
    }
    stage('Push Docker Image'){
      steps{
        withCredentials([string(credentialsId: 'DockerHubPW', variable: 'dockerHubPwd')]) {
          sh "docker login -u manashchauhan -p ${dockerHubPwd}"
          sh "docker push manashchauhan/nodeapp/v1:${DOCKER_TAG}" 
        }
      }
    }
  }
}

def getDockerTag(){
  def dockerTag = sh script: 'git rev-parse HEAD', returnStdout: true
  return dockerTag
}
