pipeline{
  agent any
  environment{
    DOCKER_TAG = getDockerTag()
  }
  stages{
    stage('Build Docker Image'){
      sh 'docker build -t manashchauhan/nodeapp/v1:${DOCKER_TAG}'
    }
  }
}

def getDockerTag(){
  def dockerTag = sh script: 'git rev-parse HEAD', returnstdOut: true
  return dockerTag
}
