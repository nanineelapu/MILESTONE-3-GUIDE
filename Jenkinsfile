// ============================================================
//  Milestone 3 - Ecommerce CI/CD Pipeline
//  Paste this into: Jenkins -> Job 'EcommerceApp' -> Pipeline script
//
//  Requires (configured in Step 9):
//    - Maven tool named exactly:      Maven
//    - SonarQube server named exactly: SonarQube
//    - SonarQube webhook -> Jenkins (Step 10 Part A)
//  Job MUST be named 'EcommerceApp' (the playbook reads that workspace path).
// ============================================================

pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {

        stage('1. Clone Source Code') {
            steps {
                // If the repo uses 'master', change branch below.
                git branch: 'master',
                    url: 'https://github.com/nanineelapu/EcommerceApp.git'
            }
        }

        stage('2. SonarQube Code Quality') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=EcommerceApp -DskipTests'
                }
            }
        }

        stage('3. Quality Gate Check') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('4. Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('5. Ansible Deploy') {
            steps {
                sh 'ansible-playbook -i /etc/ansible/hosts /etc/ansible/deploy.yml'
            }
        }
    }

    post {
        success {
            echo 'Pipeline SUCCESS - App deployed to Tomcat!'
        }
        failure {
            echo 'Pipeline FAILED - Deployment stopped.'
        }
    }
}
