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
            git branch: 'master',
                url: 'https://github.com/nanineelapu/EcommerceApp.git'
        }
    }

    stage('2. Verify Workspace') {
        steps {
            sh '''
            pwd
            ls -la
            ls -la EcommerceApp
            '''
        }
    }

    stage('3. SonarQube Code Quality') {
        steps {
            dir('EcommerceApp') {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    mvn clean verify sonar:sonar \
                    -Dsonar.projectKey=EcommerceApp \
                    -DskipTests
                    '''
                }
            }
        }
    }

    stage('4. Quality Gate Check') {
        steps {
            timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }

    stage('5. Maven Build') {
        steps {
            dir('EcommerceApp') {
                sh 'mvn clean package -DskipTests'
            }
        }
    }

    stage('6. Ansible Deploy') {
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

    always {
        cleanWs()
    }
}
}
