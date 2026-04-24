// Build petclinic, run tests (no mysql/postgres ITs), sonar, OWASP ZAP baseline, deploy jar with ansible.
// Polls git every minute. PRODUCTION_VM_IP is set on the jenkins container.

pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 90, unit: 'MINUTES')
  }

  triggers {
    // jenkins scm polling can't go below about a minute
    pollSCM('* * * * *')
  }

  environment {
    SONAR_HOST = 'http://sonarqube:9000'
    JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
    PRODUCTION_VM_IP = "${env.PRODUCTION_VM_IP}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'chmod +x ./mvnw || true'
        sh './mvnw -B -ntp clean package -DskipTests'
      }
    }

    stage('Tests') {
      steps {
        sh './mvnw -B -ntp test -Dsurefire.excludes=**/PostgresIntegrationTests.java,**/MySqlIntegrationTests.java'
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('SonarQube analysis') {
      steps {
        script {
          def token = ''
          if (fileExists('/var/jenkins_home/sonar_token')) {
            token = readFile('/var/jenkins_home/sonar_token').trim()
          }
          if (!token?.trim()) {
            error('No sonar token file. Run ansible again or add the sonar-token credential in jenkins.')
          }
          withEnv(["SONAR_TOKEN=${token}"]) {
            sh '''
              ./mvnw -B -ntp sonar:sonar \
                -Dsonar.host.url="${SONAR_HOST}" \
                -Dsonar.projectKey=spring-petclinic \
                -Dsonar.token="${SONAR_TOKEN}"
            '''
          }
        }
      }
    }

    stage('Security scan (OWASP ZAP)') {
      steps {
        sh '''
          set +e
          rm -rf zap-report-dir
          mkdir -p zap-report-dir
          JAR=$(ls target/spring-petclinic-*.jar | head -1)
          nohup java -jar "$JAR" --server.port=8090 > /tmp/petclinic-zap.log 2>&1 &
          echo $! > /tmp/petclinic-zap.pid
          sleep 75
          CID=$(docker ps -qf "name=^jenkins$")
          docker run --rm --user root \
            -v "$WORKSPACE:/zap/wrk:z" \
            --network "container:${CID}" \
            ghcr.io/zaproxy/zaproxy:stable \
            zap-baseline.py -t http://127.0.0.1:8090 -I \
              -r /zap/wrk/zap-report-dir/zap-report.html \
              -J /zap/wrk/zap-report-dir/zap-report.json || true
          if [ -f /tmp/petclinic-zap.pid ]; then
            kill "$(cat /tmp/petclinic-zap.pid)" 2>/dev/null || true
          fi
        '''
      }
      post {
        always {
          publishHTML([
            reportName: 'ZAP Baseline Report',
            reportDir: 'zap-report-dir',
            reportFiles: 'zap-report.html',
            keepAll: true,
            alwaysLinkToLastBuild: true,
            allowMissing: true
          ])
        }
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          chmod 600 /var/jenkins_home/.ssh/id_rsa || true
          JAR=$(ls "${WORKSPACE}"/target/spring-petclinic-*.jar | head -1)
          ansible-playbook /var/jenkins_home/deploy-playbook.yml \
            -i "${PRODUCTION_VM_IP}," \
            -u akanshc \
            --private-key /var/jenkins_home/.ssh/id_rsa \
            -e "jar_path=${JAR}" \
            --ssh-common-args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
        '''
      }
    }
  }

  post {
    success {
      echo 'Pipeline completed successfully.'
    }
    failure {
      echo 'Pipeline failed.'
    }
  }
}
