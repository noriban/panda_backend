def imageName="noriban/devopscoreback"
def dockerRegistry=""
def registryCredentials="dockerhub"
def dockerTag=""

pipeline {
    agent {
        label 'agent'
    }
    
    environment {
        scanerHome = tool 'SonarQube'
    }
    
    stages {
        stage('Git checkout scm') {
            steps {
            	checkout scm
		}
        }

        stage('Unit tests') {
            steps {
                sh 'pip3 install -r requirements.txt'
                sh 'python3 -m pytest --cov=. --cov-report xml:test-results/coverage.xml --junitxml=test-results/pytest-report.xml'
            }
        }
        
        stage('SonarQube') {
            steps {
                withSonarQubeEnv("SonarQube") {
                    sh "${scanerHome}/bin/sonar-scanner"
                }
                timeout(time:1, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true   
                }
            }
        }
        
        stage('Docker') {
            steps {
                script{
                    dockerTag = "RC-${env.BUILD_ID}.${env.GIT_COMMIT.take(7)}"
                    //.${env.GIT_COMMIT.take(7)}
                    applicationImage=docker.build("$imageName:$dockerTag", ".")
                }
            }
        }
        
        stage('Pushing image to Artifactory') {
            steps {
                script {
                    docker.withRegistry("$dockerRegistry", "$registryCredentials") {
                        applicationImage.push()
                        applicationImage.push('latest')
                    }
                }
            }
        }
        stage ('Push to repo') {
            steps {
                dir('ArgoCD') {
                    withCredentials([gitUsernamePassword(credentialsId: 'git', gitToolName: 'Default')]) {
                        git branch: 'main', url: 'https://github.com/noriban/panda_argocd.git'
                        sh """ cd backend
                        git config --global user.email "norbertrom01@gmail.com"
                        git config --global user.name "noriban"
                        sed -i "s#$imageName.*#$imageName:$dockerTag#g" backend.yml
                        git commit -am "Set new $dockerTag tag."
                        git diff
                        git push origin main
                        """
                    }                  
                } 
            }
        }
    }
    post{
        always {
            junit testResults: "test-results/*.xml"
            cleanWs()
        }
    }
}