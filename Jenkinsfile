// Build petclinic, run tests (no mysql/postgres ITs), sonar, OWASP ZAP baseline, deploy jar with ansible.
// Polls git every minute. PRODUCTION_VM_IP is set on the jenkins container.

pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 90, unit: 'MINUTES')
  }

  triggers {
    pollSCM('* * * * *')
  }

  environment {
    SONAR_HOST = 'http://sonarqube:9000'
    JAVA_HOME  = '/usr/lib/jvm/java-21-openjdk-amd64'
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

    stage('Unit Tests') {
      steps {
        sh '''
          ./mvnw -B -ntp test \
            -Dsurefire.excludes=**/PostgresIntegrationTests.java,**/MySqlIntegrationTests.java
        '''
      }
      post {
        always {
          junit allowEmptyResults: true,
               testResults: '**/target/surefire-reports/*.xml'
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
          JAR=$(ls target/spring-petclinic-*.jar | head -1)
          nohup java -jar "$JAR" --server.port=8090 \
            > /tmp/petclinic-zap.log 2>&1 &
          APP_PID=$!
          sleep 75

          # figure out the host-side path for the jenkins workspace dir
          CID=$(docker ps -qf "name=^jenkins$")
          JENKINS_HOME_HOST=$(docker inspect jenkins \
            --format '{{ range .Mounts }}{{ if eq .Destination "/var/jenkins_home" }}{{ .Source }}{{ end }}{{ end }}')
          HOST_WS="${JENKINS_HOME_HOST}/workspace/spring-petclinic"

          docker run --rm --user root \
            -v "${HOST_WS}:/zap/wrk:z" \
            --network "container:${CID}" \
            ghcr.io/zaproxy/zaproxy:stable \
            zap-baseline.py \
              -t http://127.0.0.1:8090 \
              -I \
              -r zap-report.html \
              -J zap-report.json \
            || true

          # stop the app
          kill $APP_PID 2>/dev/null || true

          # keep a copy on the jenkins volume for easy access
          mkdir -p /var/jenkins_home/zap-reports
          cp zap-report.html /var/jenkins_home/zap-reports/ 2>/dev/null
          cp zap-report.json /var/jenkins_home/zap-reports/ 2>/dev/null
          true
        '''
      }
      post {
        always {
          publishHTML([
            reportDir: '/var/jenkins_home/zap-reports',
            reportFiles: 'zap-report.html',
            reportName: 'ZAP Security Report',
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
