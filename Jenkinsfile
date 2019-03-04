/*
    This is an example pipeline that implement full CI/CD for a simple static web site packed in a Docker image.

    The pipeline is made up of 6 main steps
    1. Git clone and setup
    2. Build and local tests
    3. Publish Docker and Helm
    4. Deploy to dev and test
    5. Deploy to staging and test
    6. Optionally deploy to production and test
 */

/*
    Create the kubernetes namespace
 */
def createNamespace (namespace) {
    echo "Creating namespace ${namespace} if needed"

    sh "[ ! -z \"\$(kubectl get ns ${namespace} -o name 2>/dev/null)\" ] || kubectl create ns ${namespace}"
}

/*
    Helm install
 */
def helmInstall (namespace, release) {
    echo "Installing ${release} in ${namespace}"

    script {
        release = "${release}-${namespace}"
        sh "helm repo add helm ${HELM_REPO}; helm repo update"
	
	sh "docker login ${DOCKER_REG} -u ${DOCKER_USR} -p ${DOCKER_PSW}"
	sh "docker pull 077776510793.dkr.ecr.ap-southeast-1.amazonaws.com/${IMAGE_NAME}:${DOCKER_TAG}"
	sh "docker tag 077776510793.dkr.ecr.ap-southeast-1.amazonaws.com/${IMAGE_NAME}:${DOCKER_TAG} ${IMAGE_NAME}:${DOCKER_TAG}"
	sh "helm upgrade --install --namespace ${namespace} ${release} helm/acme"
        sh "sleep 5"
    }
}

/*
    Helm delete (if exists)
 */
def helmDelete (namespace, release) {
    echo "Deleting ${release} in ${namespace} if deployed"

    script {
        release = "${release}-${namespace}"
        sh "[ -z \"\$(helm ls --short ${release} 2>/dev/null)\" ] || helm delete --purge ${release}"
    }
}

/*
    Run a curl against a given url
 */
def curlRun (url, out) {
    echo "Running curl on ${url}"

    script {
        if (out.equals('')) {
            out = 'http_code'
        }
        echo "Getting ${out}"
            def result = sh (
                returnStdout: true,
                script: "curl --output /dev/null --silent --connect-timeout 5 --max-time 5 --retry 5 --retry-delay 5 --retry-max-time 30 --write-out \"%{${out}}\" ${url}"
        )
        echo "Result (${out}): ${result}"
    }
}

/*
    Test with a simple curl and check we get 200 back
 */
def curlTest (namespace, out) {
    echo "Running tests in ${namespace}"

    script {
        if (out.equals('')) {
            out = 'http_code'
        }

        // Get deployment's service IP
        def svc_ip = sh (
                returnStdout: true,
                script: "kubectl get svc -n ${namespace} | grep ${ID} | awk '{print \$3}'"
        )

        if (svc_ip.equals('')) {
            echo "ERROR: Getting service IP failed"
            sh 'exit 1'
        }

        echo "svc_ip is ${svc_ip}"
        url = 'http://' + svc_ip

        curlRun (url, out)
    }
}

/*
    This is the main pipeline section with the stages of the CI/CD
 */
pipeline {

    options {
        // Build auto timeout
        timeout(time: 60, unit: 'MINUTES')
    }

    // Some global default variables
    environment {
        IMAGE_NAME = 'acme'
        TEST_LOCAL_PORT = 8817
        DEPLOY_PROD = false
        HELM_HOME = "/root/.helm"
	WORKSPACE = "/home/ec2-user/jenkins-pipeline-kubernetes"
        PARAMETERS_FILE = "${JENKINS_HOME}/parameters.groovy"
	DOCKER_REG="077776510793.dkr.ecr.ap-southeast-1.amazonaws.com"
	DOCKER_USR="AWS"
	DOCKER_PSW="eyJwYXlsb2FkIjoiREtZVEVoK3VPbWVPc0ZHbDBSblI0dTIwV2tmS3haNUpJUU9rZzlrRFdXSjVEdzhsRStrc3AvSWxXbThYUnk1eWE1d2p6UXB3Y0tYQ2pTU1Y4T2NKVjNrdVR0SFJjSmZvaVNVSVhrdk4xYloveUtaTW4rUVJNNndtZFZFQ1ZqOGNWM2xhaTRTNW9CR2ZzRVVPZi85bXZLcThWRUQrdm1YU3pvMUNUUE13VnpRTVlvMktxTWZTaGJhZVlacm5wMkgva2FUbldHeXY1ZE5lSjFwZFZSRmhtNkQ4dmpoVm8wcjZ0L2kvSWxiTHpZWVhjNU16ZmNXT3EzUEw1YzVEdUU1NFoxOVRrSTNHbDB0WDJVTHVwWkx3SEpDVkRlNkZ0S0ZXYjFGandDZFZlNE9wZGdueW1acHJSYkxEUDRBNUQ3d3dDMnNQSUx2L0RyV0E5Z1BjMXBvWkJqODF3OFdGYWk5WnRIeWk5NG11WTFiRlNVYWo2OHlSclhLdFpVWTlETWlPMm81alE0K0NBaU5Fckw4NGFNTmRWWGNCc3pRQkxURk9VOVBTTXBUUk53R0FJRHFpanlFdVZjVmRLV3JQN2lOazY4L2NuU1l1NkU3T05QWWdJekdvVG5CK245b0sxQk10emF2NDh5Z3VCMUJnK2xXTjBHUmxTdDlVS2l6cUxzd3VBTlRRTHZLMitxWmNmS1RkQ1Z1K2FwYTBRaXhrSEVMNEo3M3NiKzFKd1VQVFUrZHJCQ1JhVnN6V1E1TTlXUmdFQiswSVBqajIvRkdOaHNyQXZmOE50ZldNOVlMYkcwWUdUc1c2VW8wTndBcE85L093SXRjd0hOV2lRcXZabFNHVEd6ZVBmVlJTUGJlc25FL2dSbDJGOC9nMXhLSWpieDFYYklqN25GdHBzVnY3d3FiSmhUYncxSUxSTEJRMkZvdW05MHdyV1J6ajJRZHVrR2hiMWwzeUlBMFRLNkhMdjNEMDJXZFZGZUJuS1ZOZ2RuV0hiOGFtVDZzT0JuWk5WeG5zeWpjODNFRWZjRm1aODBKTlArdlBpaTRzbkZ1RSt5VlE5Q3J4QlEwMU02TDZjM080dmFYRmJyWmlWZC9abTVpZzBJNGUvWWhHNVJieW5NcmhwNHJ6ajJxd1dQdFVpUDFML0R4dWl0bjgvSStNT0lYMlg5VzlWbVNWUC9XMEZITCtWRlcvTzg4UnRyZE93VTJWamJVWHpuYTc0d2VTZ3ZTMytENXZvV2tmdkM1WmZ3b084aGl1a1ptWlZ0RllVem9HOGxuM1BsdUNXS0FjTjR5T0hDV0QzZHdleVNOZ0hMMk95NzNvZGtVbVFGUHp4R25NcG1sZThGMTRmbWNLamZ6eldSVGlYWjJoNjRYUHFDRUlxc3ByWFpjTnN2b2UxQXRKMWswaTRVSDBsSVpvMVRxbFFnWEVWVEhMWEZnYy9wY09tY3lTIiwiZGF0YWtleSI6IkFRRUJBSGlkRXJaQ2ZoS09lRE0wOCtjUDVmdHlqdlE5WE1NU1E0cEswRlpudkFaWEpnQUFBSDR3ZkFZSktvWklodmNOQVFjR29HOHdiUUlCQURCb0Jna3Foa2lHOXcwQkJ3RXdIZ1lKWUlaSUFXVURCQUV1TUJFRURFaUVUbUF6RGVXNU9FenRxQUlCRUlBN0tuZno2TTVSWE8vejRJdXdBNFBHRUdwVXQyd3k0U0ZxSWVRMWlXR2l6QkFXWnhseUNKNlJzY09ibXBoVWxydzU2MkJHa05LRlhia2szcWM9IiwidmVyc2lvbiI6IjIiLCJ0eXBlIjoiREFUQV9LRVkiLCJleHBpcmF0aW9uIjoxNTUxNzQ3MDYxfQ=="
	DOCKER_REPO="acme"
	DOCKER_TAG="dev"
	HELM_REPO="http://172.31.3.74:8081/artifactory/helm-repo"
	HELM_USR="admin"
	HELM_PSW="password"
	IMG_PULL_SECRET="regcred"
    }

    parameters {
        string (name: 'GIT_BRANCH',           defaultValue: 'master',  description: 'Git branch to build')
        booleanParam (name: 'DEPLOY_TO_PROD', defaultValue: false,     description: 'If build and tests are good, proceed and deploy to production without manual approval')


        // The commented out parameters are for optionally using them in the pipeline.
        // In this example, the parameters are loaded from file ${JENKINS_HOME}/parameters.groovy later in the pipeline.
        // The ${JENKINS_HOME}/parameters.groovy can be a mounted secrets file in your Jenkins container.
/*
        string (name: 'DOCKER_REG',       defaultValue: 'docker-artifactory.my',                   description: 'Docker registry')
        string (name: 'DOCKER_TAG',       defaultValue: 'dev',                                     description: 'Docker tag')
        string (name: 'DOCKER_USR',       defaultValue: 'admin',                                   description: 'Your helm repository user')
        string (name: 'DOCKER_PSW',       defaultValue: 'password',                                description: 'Your helm repository password')
        string (name: 'IMG_PULL_SECRET',  defaultValue: 'docker-reg-secret',                       description: 'The Kubernetes secret for the Docker registry (imagePullSecrets)')
        string (name: 'HELM_REPO',        defaultValue: 'https://artifactory.my/artifactory/helm', description: 'Your helm repository')
        string (name: 'HELM_USR',         defaultValue: 'admin',                                   description: 'Your helm repository user')
        string (name: 'HELM_PSW',         defaultValue: 'password',                                description: 'Your helm repository password')
*/
    }

    // In this example, all is built and run from the master
    agent { node { label 'master' } }

    // Pipeline stages
    stages {

        ////////// Step 1 //////////
        stage('Git clone and setup') {
            steps {
                echo "Check out acme code"
                git branch: "master",
                        credentialsId: 'eldada-bb',
                        url: 'https://github.com/nabarunsen/jenkins-pipeline-kubernetes.git'

                // Validate kubectl
                sh "kubectl cluster-info"

                // Init helm client
                sh "helm init"

                // Make sure parameters file exists
                script {
                    if (! fileExists("${PARAMETERS_FILE}")) {
                        echo "ERROR: ${PARAMETERS_FILE} is missing!"
                    }
                }

                // Load Docker registry and Helm repository configurations from file
                //load "${JENKINS_HOME}/parameters.groovy"

                echo "DOCKER_REG is ${DOCKER_REG}"
                echo "HELM_REPO  is ${HELM_REPO}"

                // Define a unique name for the tests container and helm release
                script {
                    branch = GIT_BRANCH.replaceAll('/', '-').replaceAll('\\*', '-')
                    ID = "${IMAGE_NAME}-${DOCKER_TAG}-${branch}"

                    echo "Global ID set to ${ID}"
                }
            }
        }

        ////////// Step 2 //////////
        stage('Build and tests') {
            steps {
                echo "Building application and Docker image"
                sh "${WORKSPACE}/build.sh --build --registry ${DOCKER_REG} --tag ${DOCKER_TAG} --docker_usr ${DOCKER_USR} --docker_psw ${DOCKER_PSW}"

                echo "Running tests"

                // Kill container in case there is a leftover
                sh "[ -z \"\$(docker ps -a | grep ${ID} 2>/dev/null)\" ] || docker rm -f ${ID}"

                echo "Starting ${IMAGE_NAME} container"
                sh "docker run --detach --name ${ID} --rm --publish ${TEST_LOCAL_PORT}:80 ${DOCKER_REG}/${IMAGE_NAME}:${DOCKER_TAG}"

                script {
                    //host_ip = sh(returnStdout: true, script: '/sbin/ip route | awk \'/default/ { print $3 ":${TEST_LOCAL_PORT}" }\'')
		    host_ip = "172.31.3.74:8817"
                }
            }
        }

        // Run the 3 tests on the currently running ACME Docker container
        stage('Local tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        curlRun ("http://${host_ip}", 'http_code')
                    }
                }
                stage('Curl total_time') {
                    steps {
                        curlRun ("http://${host_ip}", 'total_time')
                    }
                }
                stage('Curl size_download') {
                    steps {
                        curlRun ("http://${host_ip}", 'size_download')
                    }
                }
            }
        }

        ////////// Step 3 //////////
        stage('Publish Docker and Helm') {
            steps {
                echo "Stop and remove container"
                sh "docker stop ${ID}"

                echo "Pushing ${DOCKER_REG}/${IMAGE_NAME}:${DOCKER_TAG} image to registry"
                sh "${WORKSPACE}/build.sh --push --registry ${DOCKER_REG} --tag ${DOCKER_TAG} --docker_usr ${DOCKER_USR} --docker_psw ${DOCKER_PSW}"

                echo "Packing helm chart"
                sh "${WORKSPACE}/build.sh --pack_helm --push_helm --helm_repo ${HELM_REPO} --helm_usr ${HELM_USR} --helm_psw ${HELM_PSW}"
            }
        }

        ////////// Step 4 //////////
        stage('Deploy to dev') {
            steps {
                script {
                    namespace = 'development'

                    echo "Deploying application ${ID} to ${namespace} namespace"
                    createNamespace (namespace)

                    // Remove release if exists
                    helmDelete (namespace, "${ID}")

                    // Deploy with helm
                    echo "Deploying"
                    helmInstall(namespace, "${ID}")
                }
            }
        }

        // Run the 3 tests on the deployed Kubernetes pod and service
        stage('Dev tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        curlTest (namespace, 'http_code')
                    }
                }
                stage('Curl total_time') {
                    steps {
                        curlTest (namespace, 'time_total')
                    }
                }
                stage('Curl size_download') {
                    steps {
                        curlTest (namespace, 'size_download')
                    }
                }
            }
        }

        /*stage('Cleanup dev') {
            steps {
                script {
                    // Remove release if exists
                    helmDelete (namespace, "${ID}")
                }
            }
        }

        ////////// Step 5 //////////
        stage('Deploy to staging') {
            steps {
                script {
                    namespace = 'staging'

                    echo "Deploying application ${IMAGE_NAME}:${DOCKER_TAG} to ${namespace} namespace"
                    createNamespace (namespace)

                    // Remove release if exists
                    helmDelete (namespace, "${ID}")

                    // Deploy with helm
                    echo "Deploying"
                    helmInstall (namespace, "${ID}")
                }
            }
        }

        // Run the 3 tests on the deployed Kubernetes pod and service
        stage('Staging tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        curlTest (namespace, 'http_code')
                    }
                }
                stage('Curl total_time') {
                    steps {
                        curlTest (namespace, 'time_total')
                    }
                }
                stage('Curl size_download') {
                    steps {
                        curlTest (namespace, 'size_download')
                    }
                }
            }
        }

        stage('Cleanup staging') {
            steps {
                script {
                    // Remove release if exists
                    helmDelete (namespace, "${ID}")
                }
            }
        }

        ////////// Step 6 //////////
        // Waif for user manual approval, or proceed automatically if DEPLOY_TO_PROD is true
        stage('Go for Production?') {
            when {
                allOf {
                    environment name: 'GIT_BRANCH', value: 'master'
                    environment name: 'DEPLOY_TO_PROD', value: 'false'
                }
            }

            steps {
                // Prevent any older builds from deploying to production
                milestone(1)
                input 'Proceed and deploy to Production?'
                milestone(2)

                script {
                    DEPLOY_PROD = true
                }
            }
        }

        stage('Deploy to Production') {
            when {
                anyOf {
                    expression { DEPLOY_PROD == true }
                    environment name: 'DEPLOY_TO_PROD', value: 'true'
                }
            }

            steps {
                script {
                    DEPLOY_PROD = true
                    namespace = 'production'

                    echo "Deploying application ${IMAGE_NAME}:${DOCKER_TAG} to ${namespace} namespace"
                    createNamespace (namespace)

                    // Deploy with helm
                    echo "Deploying"
                    helmInstall (namespace, "${ID}")
                }
            }
        }

        // Run the 3 tests on the deployed Kubernetes pod and service
        stage('Production tests') {
            when {
                expression { DEPLOY_PROD == true }
            }

            parallel {
                stage('Curl http_code') {
                    steps {
                        curlTest (namespace, 'http_code')
                    }
                }
                stage('Curl total_time') {
                    steps {
                        curlTest (namespace, 'time_total')
                    }
                }
                stage('Curl size_download') {
                    steps {
                        curlTest (namespace, 'size_download')
                    }
                }
            }
        }*/
    }
}
