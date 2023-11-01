pipeline {
  agent none

  stages {
    stage("worker build") {
      agent {
        docker {
          image 'maven:3.6.1-jdk-8-slim'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
    
      when {
        changeset "**/worker/**"
      }
            
      steps {
        echo 'Compiling worker app'
        dir('worker') {
          sh 'mvn compile'
        }
      }
    }

    stage("worker test") {
      when {
        changeset "**/worker/**"
      }
              
      agent {
        docker {
          image 'maven:3.6.1-jdk-8-slim'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
  
      steps {
        echo 'running unit tests on worker app'
        dir('worker') {
          sh 'mvn clean test'
        }
      }
    }
          
    stage("worker package") {
      when {
        branch 'master'
        changeset "**/worker/**"
      }
              
      agent {
        docker {
          image 'maven:3.6.1-jdk-8-slim'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
  
      steps {
        echo "Packaging worker app int a jarfile"
        dir('worker') {
          sh 'mvn package -DskipTests'
          archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }
      }
    }

    stage("worker docker-package") {
      agent any
            
      when {
        branch 'master'
        changeset "**/worker/**"
      }
            
      steps {
        echo 'Packaging worker app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1','dockerlogin') {
            def workerImage = docker.build("jbournepers/worker:v${env.BUILD_ID}", "./ worker")
            workerImage.push()
            workerImage.push("${env.BRANCH_NAME}")
            workerImage.push("latest")
          }
        }
      }
    }

  // result build
    stage("result build") {
      when {
        changeset "**/result/**"
      }
      steps {
        echo 'Building result app'
        dir('result') {
          sh 'npm install'
        }
      }
    }
      
    stage("result test") {
      when {
        changeset "**/result/**"
      }
      steps {
        dir('result') {
          sh '''
            npm install
            npm test
          '''
        }
      }
    }
    
    stage('result docker-package') {
      agent any
      when {
        changeset '**/result/**'
        branch 'master'
      }
      steps {
        echo 'Packaging result app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def resultImage = docker.build("jbournepers/result:v${env.BUILD_ID}", './result')
            resultImage.push()
            resultImage.push("${env.BRANCH_NAME}")
            resultImage.push('latest')
          }
        }
      }
    }
    
 // vote build
    stage('vote build') {
      agent {
        docker {
          image 'python:2.7.16-slim'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Compiling vote app.'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
        }
      }
    }

    stage('vote test') {
      agent {
        docker {
          image 'python:2.7.16-slim'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Running Unit Tests on vote app.'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
          sh 'nosetests -v'
        }

      }
    }

    stage('vote integration') { 
      agent any 
      when {
        changeset "**/vote/**" 
        branch 'master' 
      } 
    
      steps { 
        echo 'Running Integration Tests on vote app' 
        dir('vote') { 
          sh 'sh integration_test.sh' 
        } 
      } 
    } 

    stage('vote docker-package') {
      agent any
      steps {
        echo 'Packaging vote app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            // ./vote is the path to the Dockerfile that Jenkins will find from the Github repo
            def voteImage = docker.build("jbournepers/vote:${env.GIT_COMMIT}", "./vote")
            voteImage.push()
            voteImage.push("${env.BRANCH_NAME}")
            voteImage.push("latest")
          }
        }
      }
    }

  // Scan with sonarscanner
    stage('SonarQube') {
      agent any
      when {
        branch 'master'
      }
      
      environment {
        sonarpath = tool 'SonarScanner'
      }
      
      steps {
        echo "Running Sonarqube Analysis"
        withSonarQubeEnv('sonar-instavote') {
          sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
        }
      }
    }
    
    stage("Quality Gate") {
      steps {
        timeout(time: 1, unit: 'HOURS') {
          // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
          // true = set pipeline to UNSTABLE, false = don't
          waitForQualityGate abortPipeline: true
        }
      }
    } 
           
  // Deploy it all
    stage('deploy to dev') {
      agent any
      when {
        branch 'master'
      }
      steps {
        echo 'Deploy instavote with docker compose'
        sh 'docker-compose up -d'
      }
    }
  }
  post {
    always {
      echo 'Building multribranch pipeline for worker is completed..'
    }
  }
}
