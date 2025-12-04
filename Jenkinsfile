pipeline {
    agent any

    environment {
        APP_NAME    = "flask-app"
        OC_PROJECT  = "mg1982-dev"   // change if your project is different
        OC_SERVER   = "https://172.30.0.1:443" // adjust to your sandbox API URL
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
                // Jenkins OpenShift image may not have python3; if it fails, you can comment this stage out
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
                    echo "Creating ImageStream + BuildConfig from YAML..."
                    oc apply -f openshift/bc-flask-app.yaml
                  else
                    echo "BuildConfig ${APP_NAME} already exists, updating if needed..."
                    oc apply -f openshift/bc-flask-app.yaml
                  fi
                '''
            }
        }

        stage('Build Image in OpenShift') {
            steps {
                sh '''
                  oc project ${OC_PROJECT}
                  # Start binary build using the flask-app directory (contains Dockerfile)
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

