pipeline {

    agent any

    options {
        cleanWs()   // Ensures old workspace files (e.g., __pycache__) are removed
    }

    environment {
        APP_NAME   = "flask-app"
        OC_PROJECT = "mg1982-dev"
        OC_SERVER  = "https://kubernetes.default.svc"

        // OC_TOKEN is injected into Jenkins pod as env variable
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
                sh 'ls -R'
            }
        }

        stage('Remove Python cache') {
            steps {
                sh '''
                    echo "Removing Python __pycache__ folders..."
                    find . -type d -name "__pycache__" -exec rm -rf {} +
                '''
            }
        }

        stage('Quick Python Syntax Check') {
            steps {
                sh '''
                  if command -v python3 >/dev/null 2>&1; then
                    echo "Running syntax check..."
                    cd flask-app
                    python3 -m py_compile app.py || { echo "Python syntax error"; exit 1; }
                  else
                    echo "python3 not installed — skipping check"
                  fi
                '''
            }
        }

        stage('Login to OpenShift') {
            steps {
                sh '''
                    echo "Logging into OpenShift..."
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

                    echo "Checking for BuildConfig..."
                    if ! oc get bc/${APP_NAME} >/dev/null 2>&1; then
                        echo "BuildConfig not found — creating..."
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
                    echo "Starting binary build with NEW source code..."
                    oc project ${OC_PROJECT}
                    oc start-build ${APP_NAME} --from-dir=flask-app --follow --wait
                '''
            }
        }

        stage('Deploy / Rollout') {
            steps {
                sh '''
                    echo "Applying DeploymentConfig..."
                    oc apply -f openshift/dc-flask-app.yaml

                    echo "Forcing rollout to apply latest image..."
                    oc rollout restart deployment/${APP_NAME} || true

                    echo "Waiting for rollout to complete..."
                    oc rollout status deployment/${APP_NAME} --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            sh '''
                echo "Build & Deploy SUCCESS!"
                echo "Your application URL is:"
                oc get route ${APP_NAME} -o jsonpath='{.spec.host}{"\\n"}'
            '''
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}
