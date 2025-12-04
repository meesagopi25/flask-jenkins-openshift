pipeline {

    agent any

    environment {
        APP_NAME    = "flask-app"
        OC_PROJECT  = "mg1982-dev"
        OC_SERVER   = "https://kubernetes.default.svc"   
        // Jenkins pod now already has OC_TOKEN injected from DeploymentConfig
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
                  echo "Using OC_TOKEN injected into Jenkins pod..."
                  echo "Logging in to OpenShift..."

                  oc login --token=$OC_TOKEN --server=${OC_SERVER}

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

                  if ! oc get bc/${APP_NAME} >/dev-null 2>&1; then
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
                  oc start-build ${APP_NAME} --from-dir=flask-app --follow --wait
                '''
            }
        }

        stage('Deploy / Rollout') {
            steps {
                sh '''
                  oc project ${OC_PROJECT}
                  oc apply -f openshift/dc-flask-app.yaml
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
