pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8.6'
        jdk 'JDK 11'
    }
    
    environment {
        SONAR_SCANNER = tool 'SonarQubeScanner'
        APP_NAME = "NumberGuessGame"
        VERSION = "${BUILD_NUMBER}"
        SLACK_CHANNEL = "#deployments"
        TOMCAT_TEST_PATH = "~/apache-tomcat-7.0.94"
        TOMCAT_PROD_PATH = "~/apache-tomcat-7.0.94"
        WAR_FILE = "target/NumberGuessGame-1.0-SNAPSHOT.war"
        DEPLOY_NAME = "NumberGuessGame.war"
        SERVER_IP = "52.87.206.58"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                
                // Save git commit info for later use
                script {
                    env.GIT_COMMIT_MSG = sh(script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh(script: 'git log -1 --pretty=%an ${GIT_COMMIT}', returnStdout: true).trim()
                }
            }
        }
        
        stage('Code Quality Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ${SONAR_SCANNER}/bin/sonar-scanner \
                        -Dsonar.projectKey=${APP_NAME} \
                        -Dsonar.projectName='Number Guess Game' \
                        -Dsonar.sources=src/main \
                        -Dsonar.tests=src/test \
                        -Dsonar.java.binaries=target/classes
                    """
                }
                
                // OWASP Dependency Check
                sh 'mvn org.owasp:dependency-check-maven:aggregate'
                dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
            }
        }
        
        stage('Build & Test') {
            steps {
                sh 'mvn clean compile test'
                junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                
                // Code coverage with JaCoCo
                jacoco execPattern: 'target/jacoco.exec'
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'target/site/jacoco',
                    reportFiles: 'index.html',
                    reportName: 'Code Coverage Report'
                ])
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }
        
        stage('Backup Current Deployment') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                    sh """
                        ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                            if [ -f ${TOMCAT_TEST_PATH}/webapps/${DEPLOY_NAME} ]; then
                                mkdir -p ~/backups
                                cp ${TOMCAT_TEST_PATH}/webapps/${DEPLOY_NAME} ~/backups/${DEPLOY_NAME}.\$(date +%Y%m%d%H%M%S)
                            fi
                        '
                    """
                }
            }
        }
        
        stage('Deploy to Test') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                    // Deploy to test Tomcat
                    sh """
                        scp -i \$keyfile -o StrictHostKeyChecking=no ${WAR_FILE} ec2-user@${SERVER_IP}:/tmp/
                        ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                            sudo cp /tmp/NumberGuessGame-1.0-SNAPSHOT.war ${TOMCAT_TEST_PATH}/webapps/${DEPLOY_NAME}
                            sudo ${TOMCAT_TEST_PATH}/bin/shutdown.sh
                            sleep 5
                            sudo ${TOMCAT_TEST_PATH}/bin/startup.sh
                        '
                    """
                    
                    // Wait for Tomcat to start
                    sleep 15
                    
                    // Run smoke tests
                    sh 'curl -f http://${SERVER_IP}:8080/${APP_NAME}/ || exit 1'
                }
            }
            post {
                success {
                    echo "Test deployment successful"
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'good',
                        message: "Test deployment successful for ${APP_NAME} ${VERSION}"
                    )
                }
                failure {
                    echo "Test deployment failed"
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'danger',
                        message: "Test deployment failed for ${APP_NAME} ${VERSION}"
                    )
                }
            }
        }
        
        stage('Performance Testing') {
            steps {
                // Run JMeter tests against test environment
                sh '''
                    mkdir -p jmeter/results
                    jmeter -n -t jmeter/test-plan.jmx -l jmeter/results/test-results.jtl -e -o jmeter/results/dashboard \
                    -Jhost=${SERVER_IP} -Jport=8080 -Jpath=${APP_NAME} -Jthreads=10 -Jrampup=30 -Jduration=60
                '''
                
                // Publish JMeter reports
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'jmeter/results/dashboard',
                    reportFiles: 'index.html',
                    reportName: 'Performance Test Report'
                ])
            }
        }
        
        stage('Security Scanning') {
            steps {
                // Run OWASP ZAP for security scanning
                sh '''
                    docker run --rm -v $(pwd)/zap-report:/zap/wrk owasp/zap2docker-stable zap-baseline.py \
                    -t http://${SERVER_IP}:8080/${APP_NAME}/ -g gen.conf -r zap-report.html
                '''
                
                // Publish ZAP report
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'zap-report',
                    reportFiles: 'zap-report.html',
                    reportName: 'Security Test Report'
                ])
            }
        }
        
        stage('Deploy to Production') {
            steps {
                // Require manual approval for production deployment
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Deploy to Production?', submitter: 'admin'
                }
                
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                    // Create backup of current production deployment
                    sh """
                        ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                            if [ -f ${TOMCAT_PROD_PATH}/webapps/${DEPLOY_NAME} ]; then
                                mkdir -p ~/prod-backups
                                cp ${TOMCAT_PROD_PATH}/webapps/${DEPLOY_NAME} ~/prod-backups/${DEPLOY_NAME}.\$(date +%Y%m%d%H%M%S)
                            fi
                        '
                    """
                    
                    // Deploy to production Tomcat with rolling update
                    sh """
                        scp -i \$keyfile -o StrictHostKeyChecking=no ${WAR_FILE} ec2-user@${SERVER_IP}:/tmp/
                        ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                            # Remove old app but keep Tomcat running for other apps
                            sudo rm -f ${TOMCAT_PROD_PATH}/webapps/${DEPLOY_NAME}
                            sudo rm -rf ${TOMCAT_PROD_PATH}/webapps/${APP_NAME}
                            
                            # Deploy new version
                            sudo cp /tmp/NumberGuessGame-1.0-SNAPSHOT.war ${TOMCAT_PROD_PATH}/webapps/${DEPLOY_NAME}
                            
                            # Create version marker
                            echo "${VERSION}" | sudo tee ${TOMCAT_PROD_PATH}/webapps/${APP_NAME}.version
                            
                            # Restart Tomcat
                            sudo ${TOMCAT_PROD_PATH}/bin/shutdown.sh
                            sleep 10
                            sudo ${TOMCAT_PROD_PATH}/bin/startup.sh
                        '
                    """
                    
                    // Wait for Tomcat to start
                    sleep 20
                    
                    // Run smoke tests
                    sh 'curl -f http://${SERVER_IP}:8080/${APP_NAME}/ || exit 1'
                    
                    // Mark deployment as successful
                    sh """
                        ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                            echo "${VERSION}" | sudo tee ${TOMCAT_PROD_PATH}/webapps/${APP_NAME}.deployed
                        '
                    """
                }
            }
            post {
                success {
                    // Create release tag
                    sh "git tag -a v${VERSION} -m 'Release version ${VERSION}'"
                    
                    // Send notifications
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'good',
                        message: "Production deployment successful for ${APP_NAME} ${VERSION}\nCommit: ${GIT_COMMIT_MSG}\nAuthor: ${GIT_AUTHOR}"
                    )
                    
                    // Send email notification
                    emailext (
                        subject: "Successful Deployment: ${APP_NAME} ${VERSION}",
                        body: """
                            <p>Deployment to production was successful!</p>
                            <p><b>Version:</b> ${VERSION}</p>
                            <p><b>Commit:</b> ${GIT_COMMIT_MSG}</p>
                            <p><b>Author:</b> ${GIT_AUTHOR}</p>
                            <p><b>Build URL:</b> <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                            <p><b>Application URL:</b> <a href="http://${SERVER_IP}:8080/${APP_NAME}/">http://${SERVER_IP}:8080/${APP_NAME}/</a></p>
                        """,
                        mimeType: 'text/html',
                        to: '${DEFAULT_RECIPIENTS}'
                    )
                }
                failure {
                    // Automatic rollback
                    withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                        sh """
                            ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                                # Find latest backup
                                LATEST_BACKUP=\$(ls -t ~/prod-backups/${DEPLOY_NAME}.* | head -1)
                                if [ -n "\$LATEST_BACKUP" ]; then
                                    # Restore from backup
                                    sudo cp \$LATEST_BACKUP ${TOMCAT_PROD_PATH}/webapps/${DEPLOY_NAME}
                                    sudo ${TOMCAT_PROD_PATH}/bin/shutdown.sh
                                    sleep 10
                                    sudo ${TOMCAT_PROD_PATH}/bin/startup.sh
                                    echo "Rollback to \$LATEST_BACKUP completed"
                                fi
                            '
                        """
                    }
                    
                    // Send failure notifications
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'danger',
                        message: "Production deployment FAILED for ${APP_NAME} ${VERSION}\nAutomatic rollback initiated\nCommit: ${GIT_COMMIT_MSG}\nAuthor: ${GIT_AUTHOR}"
                    )
                    
                    // Send email notification
                    emailext (
                        subject: "FAILED Deployment: ${APP_NAME} ${VERSION}",
                        body: """
                            <p>Deployment to production FAILED!</p>
                            <p>Automatic rollback has been initiated.</p>
                            <p><b>Version:</b> ${VERSION}</p>
                            <p><b>Commit:</b> ${GIT_COMMIT_MSG}</p>
                            <p><b>Author:</b> ${GIT_AUTHOR}</p>
                            <p><b>Build URL:</b> <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                        """,
                        mimeType: 'text/html',
                        to: '${DEFAULT_RECIPIENTS}'
                    )
                }
            }
        }
        
        stage('Set Up Monitoring') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                    sh """
                        ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                            # Set up health check script
                            cat > ~/health-check.sh << EOF
#!/bin/bash
# Health check script for ${APP_NAME}
TOMCAT_PATH=${TOMCAT_PROD_PATH}
APP_NAME=${APP_NAME}
LOGS_DIR=\$TOMCAT_PATH/logs
HEALTH_URL="http://localhost:8080/\$APP_NAME/"

# Check if Tomcat is running
if ! pgrep -f "tomcat" > /dev/null; then
    echo "CRITICAL: Tomcat is not running"
    exit 2
fi

# Check if WAR file is deployed
if [ ! -f \$TOMCAT_PATH/webapps/\$APP_NAME.war ] && [ ! -d \$TOMCAT_PATH/webapps/\$APP_NAME ]; then
    echo "CRITICAL: \$APP_NAME is not deployed"
    exit 2
fi

# Check application health
HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" \$HEALTH_URL)
if [ \$HTTP_CODE -ne 200 ]; then
    echo "WARNING: Application returned HTTP \$HTTP_CODE"
    exit 1
fi

# Check for Java exceptions in logs
EXCEPTIONS=\$(grep -c "Exception" \$LOGS_DIR/catalina.out)
if [ \$EXCEPTIONS -gt 0 ]; then
    echo "WARNING: Found \$EXCEPTIONS exceptions in logs"
    exit 1
fi

echo "OK: \$APP_NAME is running properly"
exit 0
EOF
                            chmod +x ~/health-check.sh
                            
                            # Set up cron job for health check
                            (crontab -l 2>/dev/null; echo "*/5 * * * * ~/health-check.sh | tee -a ~/healthcheck.log") | crontab -
                            # Set up log rotation
                            sudo bash -c 'cat > /etc/logrotate.d/tomcat << EOF
${TOMCAT_PROD_PATH}/logs/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 tomcat tomcat
    sharedscripts
    postrotate
        if [ -f ${TOMCAT_PROD_PATH}/bin/shutdown.sh ]; then
            ${TOMCAT_PROD_PATH}/bin/shutdown.sh
            sleep 5
            ${TOMCAT_PROD_PATH}/bin/startup.sh
        fi
    endscript
}
EOF'
                            
                            # Set up resource monitoring
                            cat > ~/tomcat-monitor.sh << EOF
#!/bin/bash
# Tomcat resource monitoring script
TOMCAT_PATH=${TOMCAT_PROD_PATH}
TIMESTAMP=\$(date +"%Y-%m-%d %H:%M:%S")
CPU_USAGE=\$(ps aux | grep tomcat | grep -v grep | awk '{print \$3}' | awk '{s+=\$1} END {print s}')
MEMORY_USAGE=\$(ps aux | grep tomcat | grep -v grep | awk '{print \$4}' | awk '{s+=\$1} END {print s}')
DISK_USAGE=\$(df -h | grep -w "/" | awk '{print \$5}' | sed 's/%//')
CONN_COUNT=\$(netstat -nat | grep :8080 | grep ESTABLISHED | wc -l)
THREAD_COUNT=\$(ps -eLf | grep tomcat | grep -v grep | wc -l)

echo "\$TIMESTAMP,\$CPU_USAGE,\$MEMORY_USAGE,\$DISK_USAGE,\$CONN_COUNT,\$THREAD_COUNT" >> ~/tomcat-metrics.csv
EOF
                            chmod +x ~/tomcat-monitor.sh
                            
                            # Set up cron job for resource monitoring
                            (crontab -l 2>/dev/null; echo "*/2 * * * * ~/tomcat-monitor.sh") | crontab -
                            
                            # Set up simple visualization
                            cat > ~/generate-report.sh << EOF
#!/bin/bash
# Generate HTML report from metrics
CSV_FILE=~/tomcat-metrics.csv
HTML_FILE=~/tomcat-report.html

echo "<html><head><title>Tomcat Monitoring Report</title>
<style>
body { font-family: Arial, sans-serif; margin: 20px; }
table { border-collapse: collapse; width: 100%; }
th, td { padding: 8px; text-align: left; border-bottom: 1px solid #ddd; }
th { background-color: #f2f2f2; }
tr:hover { background-color: #f5f5f5; }
.warning { background-color: #fff3cd; }
.critical { background-color: #f8d7da; }
</style></head><body>
<h1>Tomcat Monitoring Report for ${APP_NAME}</h1>
<p>Generated on \$(date)</p>
<h2>Latest Metrics</h2>
<table>
<tr><th>Timestamp</th><th>CPU Usage (%)</th><th>Memory Usage (%)</th><th>Disk Usage (%)</th><th>Connections</th><th>Threads</th></tr>" > \$HTML_FILE

# Add the latest 20 records
tail -20 \$CSV_FILE | while IFS=, read -r ts cpu mem disk conn threads; do
    if (( \$(echo "\$cpu > 80" | bc -l) || \$(echo "\$mem > 80" | bc -l) )); then
        echo "<tr class='critical'><td>\$ts</td><td>\$cpu</td><td>\$mem</td><td>\$disk</td><td>\$conn</td><td>\$threads</td></tr>" >> \$HTML_FILE
    elif (( \$(echo "\$cpu > 60" | bc -l) || \$(echo "\$mem > 60" | bc -l) )); then
        echo "<tr class='warning'><td>\$ts</td><td>\$cpu</td><td>\$mem</td><td>\$disk</td><td>\$conn</td><td>\$threads</td></tr>" >> \$HTML_FILE
    else
        echo "<tr><td>\$ts</td><td>\$cpu</td><td>\$mem</td><td>\$disk</td><td>\$conn</td><td>\$threads</td></tr>" >> \$HTML_FILE
    fi
done

echo "</table>
<h2>Application Status</h2>
<pre>\$(~/health-check.sh)</pre>
<h2>Summary</h2>
<p>Average CPU: \$(tail -100 \$CSV_FILE | cut -d',' -f2 | awk '{ total += \$1; count++ } END { print total/count }')%</p>
<p>Average Memory: \$(tail -100 \$CSV_FILE | cut -d',' -f3 | awk '{ total += \$1; count++ } END { print total/count }')%</p>
<p>Max Connections: \$(tail -100 \$CSV_FILE | cut -d',' -f5 | sort -nr | head -1)</p>
</body></html>" >> \$HTML_FILE
EOF
                            chmod +x ~/generate-report.sh
                            
                            # Set up scheduled report generation
                            (crontab -l 2>/dev/null; echo "0 * * * * ~/generate-report.sh") | crontab -
                            
                            # Set up simple alerting
                            cat > ~/check-alerts.sh << EOF
#!/bin/bash
# Simple alerting script
TOMCAT_PATH=${TOMCAT_PROD_PATH}
APP_NAME=${APP_NAME}
EMAIL="admin@example.com"

# Check CPU usage
CPU_USAGE=\$(ps aux | grep tomcat | grep -v grep | awk '{print \$3}' | awk '{s+=\$1} END {print s}')
if (( \$(echo "\$CPU_USAGE > 90" | bc -l) )); then
    echo "ALERT: High CPU usage (\$CPU_USAGE%) for \$APP_NAME" | mail -s "CRITICAL: High CPU Alert" \$EMAIL
fi

# Check memory usage
MEMORY_USAGE=\$(ps aux | grep tomcat | grep -v grep | awk '{print \$4}' | awk '{s+=\$1} END {print s}')
if (( \$(echo "\$MEMORY_USAGE > 85" | bc -l) )); then
    echo "ALERT: High memory usage (\$MEMORY_USAGE%) for \$APP_NAME" | mail -s "CRITICAL: High Memory Alert" \$EMAIL
fi

# Check disk space
DISK_USAGE=\$(df -h | grep -w "/" | awk '{print \$5}' | sed 's/%//')
if [ \$DISK_USAGE -gt 85 ]; then
    echo "ALERT: Low disk space (\$DISK_USAGE% used) on server" | mail -s "CRITICAL: Disk Space Alert" \$EMAIL
fi

# Check if application is responding
RESPONSE_CODE=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/\$APP_NAME/)
if [ \$RESPONSE_CODE -ne 200 ]; then
    echo "ALERT: Application not responding properly (HTTP \$RESPONSE_CODE)" | mail -s "CRITICAL: Application Down" \$EMAIL
fi
EOF
                            chmod +x ~/check-alerts.sh
                            
                            # Set up cron job for alerting
                            (crontab -l 2>/dev/null; echo "*/10 * * * * ~/check-alerts.sh") | crontab -
                        '
                    """
                }
                
                // Set up Prometheus JMX exporter for Tomcat metrics
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'keyfile')]) {
                    sh """
                        # Download JMX exporter jar
                        wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.16.1/jmx_prometheus_javaagent-0.16.1.jar
                        
                        # Create JMX exporter config
                        cat > jmx-config.yml << EOF
---
lowercaseOutputName: true
lowercaseOutputLabelNames: true
rules:
- pattern: ".*"
EOF
                        
                        # Upload to server
                        scp -i \$keyfile -o StrictHostKeyChecking=no jmx_prometheus_javaagent-0.16.1.jar ec2-user@${SERVER_IP}:/tmp/
                        scp -i \$keyfile -o StrictHostKeyChecking=no jmx-config.yml ec2-user@${SERVER_IP}:/tmp/
                        
                        # Install and configure
                        ssh -i \$keyfile -o StrictHostKeyChecking=no ec2-user@${SERVER_IP} '
                            sudo mkdir -p ${TOMCAT_PROD_PATH}/monitoring
                            sudo mv /tmp/jmx_prometheus_javaagent-0.16.1.jar ${TOMCAT_PROD_PATH}/monitoring/
                            sudo mv /tmp/jmx-config.yml ${TOMCAT_PROD_PATH}/monitoring/
                            
                            # Update Tomcat startup script
                            sudo sed -i "s/JAVA_OPTS=\\"\\\$JAVA_OPTS/JAVA_OPTS=\\"\\\$JAVA_OPTS -javaagent:${TOMCAT_PROD_PATH//\//\\/}\\/monitoring\\/jmx_prometheus_javaagent-0.16.1.jar=9404:${TOMCAT_PROD_PATH//\//\\/}\\/monitoring\\/jmx-config.yml/" ${TOMCAT_PROD_PATH}/bin/catalina.sh
                            
                            # Restart Tomcat to apply changes
                            sudo ${TOMCAT_PROD_PATH}/bin/shutdown.sh
                            sleep 10
                            sudo ${TOMCAT_PROD_PATH}/bin/startup.sh
                        '
                    """
                }
            }
        }
        
        stage('Documentation') {
            steps {
                // Generate deployment documentation
                sh """
                    mkdir -p documentation
                    cat > documentation/deployment.md << EOF
# Deployment Documentation for ${APP_NAME}

## Version Information
- **Version:** ${VERSION}
- **Build Date:** \$(date)
- **Git Commit:** ${GIT_COMMIT}
- **Git Author:** ${GIT_AUTHOR}
- **Commit Message:** ${GIT_COMMIT_MSG}

## Deployment Locations
- **Test Environment:** http://${SERVER_IP}:8080/${APP_NAME}/
- **Production Environment:** http://${SERVER_IP}:8080/${APP_NAME}/

## Monitoring
- **Health Check URL:** http://${SERVER_IP}:8080/${APP_NAME}/
- **Monitoring Dashboard:** http://${SERVER_IP}/tomcat-report.html
- **JMX Metrics:** http://${SERVER_IP}:9404/metrics

## Rollback Procedure
1. SSH into the server: \`ssh ec2-user@${SERVER_IP}\`
2. Find available backups: \`ls -la ~/prod-backups\`
3. Select a backup version to restore
4. Copy the backup to Tomcat: \`sudo cp ~/prod-backups/[BACKUP_FILE] ${TOMCAT_PROD_PATH}/webapps/${DEPLOY_NAME}\`
5. Restart Tomcat: \`sudo ${TOMCAT_PROD_PATH}/bin/shutdown.sh && sleep 10 && sudo ${TOMCAT_PROD_PATH}/bin/startup.sh\`

## Troubleshooting
- Check Tomcat logs: \`sudo tail -f ${TOMCAT_PROD_PATH}/logs/catalina.out\`
- Check application status: \`~/health-check.sh\`
- Check resource usage: \`cat ~/tomcat-metrics.csv | tail -10\`

## Contact Information
- **Deployment Team:** devops@example.com
- **Application Owner:** app-team@example.com
- **Emergency Contact:** 555-123-4567
EOF
                """
                
                // Archive documentation
                archiveArtifacts artifacts: 'documentation/*.md', fingerprint: true
            }
        }
    }
    
    post {
        always {
            // Clean workspace
            cleanWs()
            
            // Archive logs
            archiveArtifacts artifacts: '**/target/surefire-reports/**', fingerprint: true, allowEmptyArchive: true
            archiveArtifacts artifacts: 'jmeter/results/**', fingerprint: true, allowEmptyArchive: true
            archiveArtifacts artifacts: 'zap-report/**', fingerprint: true, allowEmptyArchive: true
        }
        success {
            echo 'Pipeline completed successfully!'
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'good',
                message: "Complete pipeline for ${APP_NAME} ${VERSION} finished successfully!"
            )
        }
        failure {
            echo 'Pipeline failed!'
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'danger',
                message: "Pipeline for ${APP_NAME} ${VERSION} FAILED!"
            )
        }
    }
}
