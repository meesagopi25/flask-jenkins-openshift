pipeline {

    agent any

    options {
        // IMPORTANT: allows pipeline to access credentials in User → mg1982
        withCredentialsUserId()
    }

    environment {
        APP_NAME    = "flask-app"
        OC_PROJECT  = "mg1982-dev"                       // change only if your project differs
        OC_SERVER   = "https://172.30.0.1:443"           // sandbox: internal API, works inside cluster
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -R'
            }
        }

        stage('Quick Python check (optional)') {
            steps {
                sh '''
                  if command -v python3 >/dev/null 2>&1; then
                    cd flask-app
                    python3 -m py_compile app.py || { echo "Python syntax error"; exit 1; }
                  else
                    echo "python3 not installed in Jenkins pod, skipping check"
                  fi
                '''
            }
        }

        stage('Login to OpenShift') {
            steps {
                withCredentials([string(credentialsId: 'oc-token', variable: 'OC_TOKEN')]) {
                    sh '''
                      echo "Logging in to OpenShift..."
                      oc login --token=$OC_TOKEN --server='${OC_SERVER}' --insecure-skip-tls-verify
                      oc project ${OC_PROJECT}
                    '''
                }
            }
        }

        stage('Ensure BuildConfig exists') {
            steps {
                sh '''
                  oc project ${OC_PROJECT}

                  if ! oc get bc/${APP_NAME} >/dev/null 2>&1; then
                    echo "BuildConfig not found — creating..."
                    oc apply -f openshift/bc-flask-app.yaml
                  else
                    echo "BuildConfig found — ensuring it is up-to-date..."
                    oc apply -f openshift/bc-flask-app.yaml
                  fi
                '''
            }
        }

        stage('Build Image in OpenShift') {
            steps {
                sh '''
                  oc project ${OC_PROJECT}
                  echo "Starting OpenShift Binary Build..."
                  oc start-build ${APP_NAME} --from-dir=flask-app --follow --wait
                '''
            }
        }

        stage('Deploy / Rollout') {
            steps {
                sh '''
                  oc project ${OC_PROJECT}
                  echo "Applying DeploymentConfig..."
                  oc apply -f openshift/dc-flask-app.yaml

                  echo "Waiting for rollout..."
                  oc rollout status deployment/${APP_NAME} --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            sh '''
              oc project ${OC_PROJECT}
              echo "=== Application Route ==="
              oc get route ${APP_NAME} -o jsonpath='{.spec.host}{"\\n"}'
            '''
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}
