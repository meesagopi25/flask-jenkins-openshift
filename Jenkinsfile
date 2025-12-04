pipeline {

    agent any

    environment {
        APP_NAME    = "flask-app"
        OC_PROJECT  = "mg1982-dev"
        OC_SERVER   = "https://kubernetes.default.svc"
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
                sh '''
                  echo "Logging into OpenShift using SERVICE ACCOUNT TOKEN..."

                  # Service account token mounted inside the Jenkins pod
                  SA_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

                  oc login \
                    --token="$SA_TOKEN" \
                    --server="${OC_SERVER}" \
                    --insecure-skip-tls-verify

                  oc project ${OC_PROJECT}

                  echo "Logged in as:"
                  oc whoami
                '''
            }
        }

        stage('Ensure BuildConfig exists') {
            steps {
                sh '''
                  oc project ${OC_PROJECT}

                  if ! oc get bc/${APP_NAME} >/dev/null 2>&1; then
                    echo "Creating BuildConfig..."
                    oc apply -f openshift/bc-flask-app.yaml
                  else
                    echo "BuildConfig exists — applying latest YAML..."
                    oc apply -f openshift/bc-flask-app.yaml
                  fi
                '''
            }
        }

        stage('Build Image in OpenShift') {
            steps {
                sh '''
                  oc project ${OC_PROJECT}
                  echo "Starting OpenShift build..."
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
              echo "Application URL:"
              oc get route ${APP_NAME} -o jsonpath='{.spec.host}{"\\n"}'
            '''
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}
