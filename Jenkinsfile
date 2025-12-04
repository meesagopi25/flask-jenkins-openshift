pipeline {

    agent any

    environment {
        APP_NAME    = "flask-app"
        OC_PROJECT  = "mg1982-dev"
        OC_SERVER   = "https://kubernetes.default.svc"   // always correct inside cluster
        // IMPORTANT: OC_TOKEN must be injected via DeploymentConfig env var
        // Example:
        // oc set env dc/jenkins OC_TOKEN=$(oc create token jenkins -n mg1982-dev)
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -R'
            }
        }

        stage('Quick Python Check (optional)') {
            steps {
                sh '''
                  if command -v python3 >/dev/null 2>&1; then
                      cd flask-app
                      python3 -m py_compile app.py || { echo "Python syntax error"; exit 1; }
                  else
                      echo "python3 not installed, skipping..."
                  fi
                '''
            }
        }

        stage('Login to OpenShift') {
            steps {
                sh '''
                    echo "Logging in using service account token injected via OC_TOKEN..."
                    
                    if [ -z "$OC_TOKEN" ]; then
                        echo "ERROR: OC_TOKEN is not set in Jenkins DeploymentConfig!"
                        exit 1
                    fi

                    oc login --token="$OC_TOKEN" --server=${OC_SERVER} --insecure-skip-tls-verify || { echo "LOGIN FAILED"; exit 1; }

                    oc project ${OC_PROJECT}
                    echo "Logged in as:"
                    oc whoami || true
                '''
            }
        }

        stage('Ensure BuildConfig Exists') {
            steps {
                sh '''
                    oc project ${OC_PROJECT}

                    if ! oc get bc/${APP_NAME} >/dev/null 2>&1; then
                        echo "BuildConfig does not exist — creating..."
                        oc apply -f openshift/bc-flask-app.yaml
                    else
                        echo "BuildConfig exists — updating..."
                        oc apply -f openshift/bc-flask-app.yaml
                    fi
                '''
            }
        }

        stage('Build Image in OpenShift') {
            steps {
                sh '''
                    oc project ${OC_PROJECT}
                    echo "Starting source-to-image build..."
                    oc start-build ${APP_NAME} --from-dir=flask-app --follow --wait || {
                        echo "Build failed"
                        exit 1
                    }
                '''
            }
        }

        stage('Deploy / Rollout') {
            steps {
                sh '''
                    oc project ${OC_PROJECT}
                    echo "Applying deployment..."
                    oc apply -f openshift/dc-flask-app.yaml

                    echo "Waiting for rollout..."
                    oc rollout status deployment/${APP_NAME} --timeout=180s || {
                        echo "Rollout failed"
                        exit 1
                    }
                '''
            }
        }
    }

    post {
        success {
            sh '''
                echo "Fetching the route for the application..."

                ROUTE=$(oc get route ${APP_NAME} -o jsonpath='{.spec.host}')
                printf "Application URL: https://%s\n" "$ROUTE"
            '''
        }

        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}
