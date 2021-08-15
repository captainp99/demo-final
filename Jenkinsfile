pipeline {
  agent any
  environment {
    scannerHome = tool name: 'sonar_scanner_dotnet'
    username = 'parasjain01'
    registry = 'paras22/nagp-devops-assign-1'
    GKE_CREDENTIALS_ID = 'gcpcreds'
  }
  
  stages {
    stage('Checkout') {
      steps {
        git url: 'https://github.com/captainp99/demo-final.git'
      }
    }
    stage('nuget restore') {
      steps {
        bat "dotnet restore"
      }
    }
    stage('start sonarqube analysis') {
      when {
        branch 'master'
      }
      steps {
        echo "Start SonarQube Analysis"
        withSonarQubeEnv("Test_Sonar") {
          bat "${scannerHome}\\SonarScanner.MSBuild.exe begin /k:sonar-${username} -d:sonar.cs.opencover.reportsPaths=test-project/coverage.opencover.xml -d:sonar.cs.xunit.reportsPaths='test-project/TestResults/TestFileReport.xml'"
        }
      }
    }
	  
  stage('Build') {
    steps {
      bat 'dotnet clean'
      bat 'dotnet build -c Release -o DevopsWebApp/app/build'
    }
  }

  stage('Test Case Execution') {
    steps {
      echo "Execute Unit Test"
      bat "dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover -l:trx;LogFileName=TestFileReport.xml"
    }
  }

  stage('stop sonarqube analysis') {
    when {
      branch 'master'
    }
    steps {
      withSonarQubeEnv('Test_Sonar') {
        bat "${scannerHome}\\SonarScanner.MSBuild.exe end"
      }
    }
  }
  stage('Release artifact') {
    when {
      branch 'develop'
    }
    steps {
      bat 'dotnet publish -c Release'
    }
  }
  stage('Docker image') {
    steps {
      bat "docker build -t i-${username}-$env.BRANCH_NAME ."
      bat "docker tag i-${username}-$env.BRANCH_NAME ${registry}$env.BRANCH_NAME:$BUILD_NUMBER"
      bat "docker tag i-${username}-$env.BRANCH_NAME ${registry}$env.BRANCH_NAME:latest"
    }

  }
  stage('Containers') {
    steps {
      parallel(
        "Precontainer Check": {
          script {
            checkContainerExist = bat(script: "docker ps -qa -f name=c-${username}-${BRANCH_NAME}", returnStdout: true).trim().readLines().drop(1).join('')
            if (checkContainerExist) {
              echo ":: Found container - c-${username}-${BRANCH_NAME}"
              checkContainerRunning = bat(script: "docker ps -q -f name=c-${username}-${BRANCH_NAME}", returnStdout: true).trim().readLines().drop(1).join('')
              if (checkContainerRunning) {
                echo ":: Stopping running container - c-${username}-${BRANCH_NAME}"
                bat "docker stop c-${username}-${BRANCH_NAME}";
              }
              echo ":: Removing stopped container - ${username}-${BRANCH_NAME}"
              bat "docker rm c-${username}-${BRANCH_NAME}";
            }
          }
        },
        PushtoDockerHub: {
          withDockerRegistry(credentialsId: 'DockerHub', url: '') {
            bat "docker push ${registry}$env.BRANCH_NAME:$BUILD_NUMBER"
            bat "docker push ${registry}$env.BRANCH_NAME:latest"
          }
        }
      )
    }
  }
  stage('Docker deployment') {
    steps {
      script {
        if (env.BRANCH_NAME == 'master') {
          env.port = 7200
        } else {
          env.port = 7300
        }
        bat "docker run --name c-${username}-$env.BRANCH_NAME -d -p  ${port}:80 i-${username}-${BRANCH_NAME}"
      }
    }
  }
  stage('Kubernetes Deployment') {
	  
    steps {
      script {
                    withCredentials ([file(credentialsId: env.GKE_CREDENTIALS_ID, variable: 'GC_KEY')]) {
                        echo 'Authenticating From Google Cloud.....'
                        bat "gcloud auth activate-service-account --key-file=${GC_KEY}"
                        echo 'Authentication complete'

                        echo 'Connecting To Cluster'
                        bat "gcloud container clusters get-credentials cluster-1 --zone us-central1-c --project optimistic-yeti-321307"
                        echo 'Cluster Connection complete'
                    }
                }
      bat "kubectl apply -f config.yaml --namespace=kubernetes-cluster-${username}"
      bat "kubectl apply -f deployment.yaml --namespace=kubernetes-cluster-${username}"
    }
  }
}
post {
  always {
    echo "Test Report Generation Step"
    xunit([MSTest(deleteOutputFiles: true, failIfNotNew: true, pattern: 'test-project\\TestResults\\TestFileReport.xml', skipNoTestFiles: true)])
  }
}
}
