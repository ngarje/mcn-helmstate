def INSTALL = ""
pipeline {
    // If you are running jenkins in a container use "agent { docker { image 'dtzar/helm-kubectl:2.11.0' }}"
    agent {
        kubernetes {
          label 'deploy'
          defaultContainer 'jnlp'
          yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-deploy
spec:
  containers:
  - name: deploy
    image: dtzar/helm-kubectl:2.11.0
    command:
    - cat
    tty: true
"""
        }
    }

    environment {
        KUBECONFIG = 'kubeconfig'
        HELM_CHARTS_GIT_REPO = 'github.com/infracloudio/app-mono-helmcharts.git'
        HELM_CHARTS_REPO = 'app-mono-helmcharts'
        HELM_CHARTS_BRANCH = 'master'
        GITHUB_HOOK_SECRET = "github-webhook-token-app-mono-helmstate"
        //K8S_SERVER = credentials('k8s-server')
        //K8S_TILLER_TOKEN = credentials('k8s-tiller-token')
        //K8S_CA_BASE64 = credentials('k8s-ca-base84')
        //GIT = credentials('github-credentials')
        GIT_USR = ""
        GIT_PSW = ""
        K8S_SERVER = ""
        K8S_TILLER_TOKEN = ""
        K8S_CA_BASE64 = ""
    }

    stages {
        stage('configure webook') {
            steps {
                script {
                    setupWebhook()
	        }
	    }
        }

        stage('Create kubeconfig') {
            steps {
                container('deploy') {
                    createKubeconfig()
                }
            }
        }

        stage('Checkout scm') {
            steps {
                checkout scm
            }
        }

        stage('Find app name to deploy') {
            steps {
                script {
                    if (REF != "") {
                        env.APP = VALUESFILE.split('/')[0]
                    }
                }
                echo "application to deploy:${APP}"
            }
        }

        stage('Get helm-charts') {
            steps {
                getHelmCharts()
            }
        }

        stage('Deploy helm chart') {
            steps {
                container('deploy') {
                    script {
                        INSTALL = sh (
                            script: "helm status ${APP}",
                            returnStatus: true
                        )
                    }
                    echo "To install? ${INSTALL}"
                    helmInstall(INSTALL)
                }
            }
        }
    }
    post {
        cleanup {
            deleteDir()
        }
    }
}

def createKubeconfig() {
    sh '''
    kubectl config set-cluster k8s-cluster --server=$K8S_SERVER --certificate-authority=$K8S_CA_BASE64
    sed -i 's/certificate-authority/certificate-authority-data/g' $KUBECONFIG
    kubectl config set-credentials tiller --token=$K8S_TILLER_TOKEN
    kubectl config set-context tiller --cluster=k8s-cluster --user=tiller
    kubectl config use-context tiller
    '''
}

def getHelmCharts() {
    sh '''
    git clone https://${GIT_USR}:${GIT_PSW}@${HELM_CHARTS_GIT_REPO}
    cd ${HELM_CHARTS_REPO}
    git checkout ${HELM_CHARTS_BRANCH}
    '''
    }

def helmInstall(install) {
    sh 'echo deploying application:$APP'
    sh 'helm init --client-only --home /tmp'
    if (install) {
        sh 'helm install --home /tmp --name $APP -f $APP/values.yaml ${HELM_CHARTS_REPO}/charts/$APP/'
    } else {
        sh 'helm upgrade --home /tmp $APP -f $APP/values.yaml ${HELM_CHARTS_REPO}/charts/$APP/'
    }
}


def setupWebhook() {
    properties([
        pipelineTriggers([
            [$class: 'GenericTrigger',
                genericVariables: [
                    [key: 'REF', value: '$.ref'],
                    [key: 'VALUESFILE', value: '$.commits[0].modified[0]'],
                ],
                causeString: 'Triggered on github push',
                token: env.GITHUB_HOOK_SECRET,
                printContributedVariables: true,
                printPostContent: true,
                regexpFilterText: '$REF',
                regexpFilterExpression: 'refs/heads/master',
                regexpFilterText: '$VALUESFILE',
                regexpFilterExpression: '.*.yaml'
            ]
        ])
    ])
}
