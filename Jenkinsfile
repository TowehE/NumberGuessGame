pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8.6'
        jdk 'JDK 11'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
                junit '**/target/surefire-reports/*.xml'
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }
        
        stage('Deploy to Test') {
            when {
                branch 'dev'
            }
            steps {
                // Copy WAR file to EC2 test server
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                    sh '''
                        scp -i $keyfile -o StrictHostKeyChecking=no target/NumberGuessGame.war ec2-user@52.87.206.58:/tmp/
                        ssh -i $keyfile -o StrictHostKeyChecking=no ec2-user@52.87.206.58 'sudo cp /tmp/NumberGuessGame.war /opt/tomcat/webapps/ && sudo /opt/tomcat/bin/shutdown.sh && sleep 5 && sudo /opt/tomcat/bin/startup.sh'
                    '''
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'dev'
            }
            steps {
                // Production deployment requires manual approval
                input 'Deploy to production?'
                
                // Copy WAR file to EC2 production server
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                    sh '''
                        scp -i $keyfile -o StrictHostKeyChecking=no target/NumberGuessGame.war ec2-user@52.87.206.58:/tmp/
                        ssh -i $keyfile -o StrictHostKeyChecking=no ec2-user@52.87.206.58 'sudo cp /tmp/NumberGuessGame.war /opt/tomcat/webapps/ && sudo /opt/tomcat/bin/shutdown.sh && sleep 5 && sudo /opt/tomcat/bin/startup.sh'
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            cleanWs()
        }
    }
}
