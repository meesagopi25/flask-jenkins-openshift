pipeline {

    agent any

    environment {
        APP_NAME    = "flask-app"
        OC_PROJECT  = "mg1982-dev"
        OC_SERVER   = "https://172.30.0.1:443"
        IMAGE_TAG   = "build-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Login to OpenShift') {
            steps {
                sh '''
                    echo "Logging in with service account token..."
                    oc login --token=$OC_TOKEN --server=${OC_SERVER} --insecure-skip-tls-verify
                    oc project ${OC_PROJECT}
                '''
            }
        }

        stage('Ensure BuildConfig Exists') {
            steps {
                sh '''
                    oc apply -f openshift/bc-flask-app.yaml
                '''
            }
        }

        stage('Build Image in OpenShift') {
            steps {
                sh '''
                    echo "Starting build with tag ${IMAGE_TAG} ..."
                    oc start-build ${APP_NAME} \
                        --from-dir=flask-app \
                        --follow --wait \
                        --build-arg IMAGE_TAG=${IMAGE_TAG}
                '''
            }
        }

        stage('Update Deployment Image') {
            steps {
                sh '''
                    echo "Ensuring Deployment exists..."
                    oc apply -f openshift/dc-flask-app.yaml

                    echo "Updating deployment image to use the NEW build tag: ${IMAGE_TAG}"

                    oc set image deployment/${APP_NAME} ${APP_NAME}=image-registry.openshift-image-registry.svc:5000/${OC_PROJECT}/${APP_NAME}:${IMAGE_TAG}

                    echo "Triggering rollout..."
                    oc rollout status deployment/${APP_NAME} --timeout=300s
                '''
            }
        }
    }

    post {
        success {
            sh '''
                ROUTE=$(oc get route ${APP_NAME} -o jsonpath='{.spec.host}')
                echo "New Deployment Completed!"
                echo "URL: https://$ROUTE"
            '''
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
