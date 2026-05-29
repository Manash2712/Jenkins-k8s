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
  }
}

def getDockerTag(){
  def dockerTag = sh script: 'git rev-parse HEAD', returnStdout: true
  return dockerTag
}
