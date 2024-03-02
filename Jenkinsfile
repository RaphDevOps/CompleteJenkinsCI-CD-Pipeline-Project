
def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]
pipeline {
    agent any
    
    environment {
        WORKSPACE = "${env.WORKSPACE}"
    }
    
    tools {
        maven 'localMaven'
    }

    stages {
        stage('Git checkout') {
            steps {
                echo 'Cloning the application repo'
                git branch: 'main', url: 'https://github.com/RaphDevOps/CompleteJenkinsCI-CD-Pipeline-Project.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
            post {
                success {
                    echo 'Archiving....'
                    archiveArtifacts artifacts: '**/*.war', followSymlinks: false
                }
            }
        }
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        stage('Checkstyle Code Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        stage('SonarQube Scanning') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                       -Dsonar.projectKey=CICD-pipeline \
                       -Dsonar.host.url=http://54.234.39.126:9000 \
                       -Dsonar.login=32bcc49c549b86c3ad47166c1bf02f756ab837f4
                    '''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        stage('Upload artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh "sed -i \"s/.*<username><\\/username>/<username>$USER_NAME<\\/username>/g\" ${WORKSPACE}/nexus-setup/settings.xml"
                    sh "sed -i \"s/.*<password><\\/password>/<password>$PASSWORD<\\/password>/g\" ${WORKSPACE}/nexus-setup/settings.xml"
                    sh "cp ${WORKSPACE}/nexus-setup/settings.xml /var/lib/jenkins/.m2"
                    sh "mvn deploy -DskipTests"
                }
            }
            post {
                success {
                    echo 'Artefacts have been backed up onto Nexus..!'
                }
                failure {
                    echo 'Artifact upload failed hence removing the settings.xml file which might cause issues on the check-style'
                    sh 'sudo rm -f /var/lib/jenkins/.m2/settings.xml'
                }
            }
        }
        stage('Deploy to DEV env') {
      environment {
        HOSTS = "dev"
      }
      steps {
        sh "ansible-playbook ${WORKSPACE}/deploy.yaml --extra-vars \"hosts=$HOSTS workspace_path=$WORKSPACE\""
             }
        }
        stage('Deploy to STAGE env') {
      environment {
        HOSTS = "stage"
      }
      steps {
        sh "ansible-playbook ${WORKSPACE}/deploy.yaml --extra-vars \"hosts=$HOSTS workspace_path=$WORKSPACE\""
             }
        }
        
       stage('Approval') {
           
       steps {
        input('Do you want to proceed ?')
             }
        }
        
        stage('Deploy to PROD env') {
      environment {
        HOSTS = "prod"
      }
      steps {
        sh "ansible-playbook ${WORKSPACE}/deploy.yaml --extra-vars \"hosts=$HOSTS workspace_path=$WORKSPACE\""
             }
        }
    }
    post { 
    always { 
        echo 'I will always say Hello again!'
        slackSend channel: '#full-automated-cicd-infra-deployment', color: COLOR_MAP[currentBuild.currentResult], message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}  \n More info at:  ${env.BUILD_URL}" 
    }
 }
}

