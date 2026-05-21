pipeline {
  agent any

  tools {
    jdk    'jdk-21'
    maven  'maven-3'
    nodejs 'node-20'
  }

  /*
   * ============================================================
   * JENKINS CREDENTIALS  (Manage Jenkins → Credentials → Global)
   * ============================================================
   *
   *   ec2-ssh-key   Kind : SSH Username with private key
   *                 ID   : ec2-ssh-key
   *                 User : ubuntu
   *                 Key  : private key (.pem) for EC2-B
   *
   *   sms-env-file  Kind : Secret file
   *                 ID   : sms-env-file
   *                 File : your filled-in production .env
   *                        (see .env.example)
   *
   * ============================================================
   * JENKINS GLOBAL ENV VARS  (Manage Jenkins → System → Global properties)
   * ============================================================
   *
   *   AWS_REGION    e.g. ap-south-1
   *                 EC2-A needs an IAM Instance Profile with
   *                 ssm:GetParameter on arn:aws:ssm:*:*:parameter/sms/*
   *
   *   TF_EC2_B_IP   Manual fallback if not using SSM.
   * ============================================================
   */

  environment {
    EC2_USER   = 'ubuntu'
    DEPLOY_DIR = '/opt/sms'
    BUILD_TAG  = "${env.BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        echo '📥 Checking out source code...'
        checkout scm
      }
    }

    stage('Backend Build') {
      steps {
        dir('backend_modified') {
          sh 'mvn clean package -DskipTests -q'
        }
        sh 'cp backend_modified/target/*.jar backend_modified/target/sms-backend.jar'
      }
      post {
        always {
          junit allowEmptyResults: true,
                testResults: 'backend_modified/target/surefire-reports/*.xml'
        }
      }
    }

    stage('Frontend Build') {
      steps {
        dir('frontend') {
          sh 'npm ci --silent'
          sh 'npm run build'
        }
      }
    }

    stage('Resolve EC2-B IP') {
      steps {
        script {
          def awsRegion = env.AWS_REGION ?: 'ap-south-1'
          def ip = ''
          try {
            ip = sh(
              script: """
                aws ssm get-parameter \
                  --name '/sms/ec2-b-public-ip' \
                  --region ${awsRegion} \
                  --query 'Parameter.Value' \
                  --output text 2>/dev/null || true
              """,
              returnStdout: true
            ).trim()
          } catch (Exception ignored) {}

          if (!ip || ip == 'None' || ip == '') {
            ip = env.TF_EC2_B_IP ?: ''
          }
          if (!ip) {
            error('❌ Cannot resolve EC2-B IP. Run terraform apply first, or set TF_EC2_B_IP in Jenkins globals.')
          }
          env.EC2_HOST = ip
          echo "🎯 Target EC2-B: ${env.EC2_HOST}"
        }
      }
    }

    stage('Deploy to EC2-B') {
      when { branch 'main' }
      steps {
        echo "🚀 Deploying build #${BUILD_TAG} to EC2-B (${env.EC2_HOST})..."

        withCredentials([
          sshUserPrivateKey(credentialsId: 'ec2-ssh-key',
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER'),
          file(credentialsId: 'sms-env-file', variable: 'ENV_FILE')
        ]) {

          sh """
            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o BatchMode=yes \
              ${EC2_USER}@${EC2_HOST} \
              'mkdir -p ${DEPLOY_DIR}/{releases,frontend,monitoring/grafana/provisioning/datasources}'
          """

          sh """
            SCP="scp -i ${SSH_KEY} -o StrictHostKeyChecking=no -o BatchMode=yes"

            \$SCP backend_modified/target/sms-backend.jar \
              ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/releases/sms-backend-${BUILD_TAG}.jar

            scp -r -i ${SSH_KEY} -o StrictHostKeyChecking=no -o BatchMode=yes \
              frontend/dist/ \
              ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/frontend/

            \$SCP devops/nginx/sms.conf \
              ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/sms.conf

            \$SCP monitoring/prometheus.yml \
              ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/monitoring/prometheus.yml

            \$SCP monitoring/grafana/provisioning/datasources/prometheus.yml \
              ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/monitoring/grafana/provisioning/datasources/prometheus.yml

            \$SCP ${ENV_FILE} \
              ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/.env

            \$SCP devops/scripts/deploy.sh \
              ${EC2_USER}@${EC2_HOST}:${DEPLOY_DIR}/deploy.sh
          """

          sh """
            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -o BatchMode=yes \
              ${EC2_USER}@${EC2_HOST} \
              'chmod +x ${DEPLOY_DIR}/deploy.sh && ${DEPLOY_DIR}/deploy.sh ${BUILD_TAG}'
          """
        }
      }
    }

    stage('Health Check') {
      when { branch 'main' }
      steps {
        echo '❤️  Waiting for services to be ready...'
        sh 'sleep 30'

        sh """
          for i in 1 2 3; do
            curl -sf http://${EC2_HOST}:8082/actuator/health \
              && echo '✅ Backend healthy' && break
            echo "Attempt \$i/3 failed — retrying in 15s..."
            sleep 15
          done || { echo '❌ Backend health check failed'; exit 1; }
        """

        sh "curl -sf http://${EC2_HOST}:80            && echo '✅ Frontend reachable'   || { echo '❌ Frontend unreachable'; exit 1; }"
        sh "curl -sf http://${EC2_HOST}:9090/-/healthy && echo '✅ Prometheus healthy'  || echo '⚠️  Prometheus not yet ready (non-fatal)'"
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'backend_modified/target/sms-backend.jar',
                       allowEmptyArchive: true
    }
    success {
      echo "✅ Build #${BUILD_TAG} deployed to ${env.EC2_HOST ?: 'EC2-B'}"
    }
    failure {
      echo "❌ Build #${BUILD_TAG} failed — check stage logs above"
    }
  }
}
