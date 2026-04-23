pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 90, unit: 'MINUTES')
  }

  triggers {
    pollSCM('H/2 * * * *')
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

    stage('Unit tests') {
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
            error('Missing /var/jenkins_home/sonar_token. Re-run Ansible (Sonar token bootstrap) or configure Jenkins credential sonar-token.')
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

    stage('Security scan (ZAP baseline)') {
      steps {
        sh '''
          set +e
          JAR=$(ls target/spring-petclinic-*.jar | head -1)
          nohup java -jar "$JAR" --server.port=8090 > /tmp/petclinic-zap.log 2>&1 &
          echo $! > /tmp/petclinic-zap.pid
          sleep 75
          CID=$(docker ps -qf "name=^jenkins$")
          docker run --rm \
            -v "$WORKSPACE:/zap/wrk:z" \
            --network "container:${CID}" \
            ghcr.io/zaproxy/zaproxy:stable \
            zap-baseline.py -t http://127.0.0.1:8090 -I -r /zap/wrk/zap-report.html -J /zap/wrk/zap-report.json || true
          if [ -f /tmp/petclinic-zap.pid ]; then
            kill "$(cat /tmp/petclinic-zap.pid)" 2>/dev/null || true
          fi
        '''
      }
      post {
        always {
          publishHTML([
            reportName: 'ZAP Baseline Report',
            reportDir: '.',
            reportFiles: 'zap-report.html',
            keepAll: true,
            alwaysLinkToLastBuild: true,
            allowMissing: true
          ])
        }
      }
    }

    stage('Deploy to production (Ansible)') {
      steps {
        sh '''
          set -euo pipefail
          chmod 600 /var/jenkins_home/.ssh/id_rsa || true
          JAR=$(ls "${WORKSPACE}"/target/spring-petclinic-*.jar | head -1)
          ansible-playbook /var/jenkins_home/deploy-playbook.yml \
            -i "${PRODUCTION_VM_IP}," \
            -u gdiwan \
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
