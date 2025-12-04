pipeline {

    agent any

    environment {
        APP_NAME   = "flask-app"
        OC_PROJECT = "mg1982-dev"
        OC_SERVER  = "https://kubernetes.default.svc"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                sh '''
                    echo "Cleaning workspace manually..."
                    find . -mindepth 1 -maxdepth 1 -exec rm -rf {} +
                '''
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
                sh 'ls -R'
            }
        }

        stage('Remove Python Cache') {
            steps {
                sh '''
                    echo "Removing __pycache__..."
                    find . -type d -name "__pycache__" -exec rm -rf {} +
                '''
            }
        }

        stage('Quick Python Syntax Check') {
            steps {
                sh '''
                    if command -v python3 >/dev/null 2>&1; then
                      cd flask-app
                      python3 -m py_compile app.py || { echo "ERROR: Python syntax error"; exit 1; }
                    else
                      echo "python3 not installed; skipping syntax check"
                    fi
                '''
            }
        }

        stage('Login to OpenShift') {
            steps {
                sh '''
                    echo "Logging in using service account token injected into Jenkins pod..."
                    oc login --token=$OC_TOKEN --server=${OC_SERVER}
                    oc project ${OC_PROJECT}

                    echo "Logged in as:"
                    oc whoami
                '''
            }
        }

        stage('Ensure BuildConfig Exists') {
            steps {
                sh '''
                    oc project ${OC_PROJECT}

                    echo "Checking BuildConfig..."
                    if ! oc get bc/${APP_NAME} >/dev/null 2>&1; then
                        echo "BuildConfig not found — creating..."
                        oc apply -f openshift/bc-flask-app.yaml
                    else
                        echo "BuildConfig exists — updating YAML..."
                        oc apply -f openshift/bc-flask-app.yaml
                    fi
                '''
            }
        }

        stage('Build Image in OpenShift') {
            steps {
                sh '''
                    echo "Starting OpenShift build..."
                    oc start-build ${APP_NAME} --from-dir=flask-app --follow --wait
                '''
            }
        }

        stage('Deploy / Rollout') {
            steps {
                sh '''
                    echo "Applying Deployment YAML..."
                    oc apply -f openshift/dc-flask-app.yaml

                    echo "Forcing new rollout..."
                    oc rollout restart deployment/${APP_NAME} || true

                    echo "Waiting for rollout..."
                    oc rollout status deployment/${APP_NAME} --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            sh '''
                echo "Application URL:"
                oc get route ${APP_NAME} -o jsonpath='{.spec.host}{"\\n"}'
            '''
        }
        failure {
            echo "Pipeline failed. Review error logs above."
        }
    }
}
