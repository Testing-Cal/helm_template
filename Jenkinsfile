def parseJsonArray(jsonString){
  def datas = readJSON text: jsonString
  return datas
}

def parseJsonString(jsonString, key){
  def datas = readJSON text: jsonString
  String Values = writeJSON returnText: true, json: datas[key]
  return Values
}

def createYamlFile(data,filename) {
  writeFile file: filename, text: data
}

def parseYaml(jsonString, key) {
  def datas = readYaml text: jsonString
  String yml = writeYaml returnText: true, data: datas[key]
  return yml
}


pipeline {
    agent any
    environment {

        HELM_IMAGE_VERSION = "alpine/helm:3.8.1" //https://hub.docker.com/r/alpine/helm/tags 
        KUBECTL_IMAGE_VERSION = "bitnami/kubectl:1.24.9" //https://hub.docker.com/r/bitnami/kubectl/tags
        OC_IMAGE_VERSION = "quay.io/openshift/origin-cli:4.9.0" //https://quay.io/repository/openshift/origin-cli?tab=tags
        JENKINS_METADATA = "${JENKINS_METADATA}"
    }

    stages {
        stage('Initialization') {
            steps {
                script{
                    String generalProperties = parseJsonString(env.JENKINS_METADATA,'general')
                    generalPresent = parseJsonArray(generalProperties)

                    String kubeProperties = parseJsonString(env.JENKINS_METADATA,'kubernetes')
                    kubePresent = parseJsonArray(kubeProperties)

                    String helmProperties = parseJsonString(env.JENKINS_METADATA,'helm')
                    helmPresent = parseJsonArray(helmProperties)

                    echo generalProperties
                    echo helmProperties
                    echo generalPresent.artifactory

                    String helm_file = parseYaml(helmProperties,'values')
                    createYamlFile(helm_file,"Helm-temp.yaml")
                    sh 'sed "1d" Helm-temp.yaml > Helm.yaml'
                    sh 'cat Helm.yaml'

                    
                    env.RELEASE_NAME = generalPresent.helmReleaseName
                    env.ARTIFACTORY = generalPresent.artifactory
                    env.ACTION = generalPresent.deploymentAction
                    env.ARTIFACTORY_SECRET = generalPresent.artifactoryCredentialId
                    env.KUBERNETES_SECRET = generalPresent.kubernetesCredentialId

                    env.DEPLOYMENT_TYPE = generalPresent.deploymentType
                    env.ARTIFACTORY_URL = generalPresent.artifactoryURL
                    env.CONTEXT_PATH = generalPresent.contextPath


                    env.KUBERNETES_NAMESPACE = helmPresent.namespace
                    env.HELM_ARGUMENTS = helmPresent.additionalArguments
                    env.HELM_VALUES = helmPresent.additionalFiles
                    env.HELM_CHART_LOCATION = helmPresent.workingDirectory
                    env.HELM_CHART_TYPE = helmPresent.type
                    env.HELM_CHART_VERSION = helmPresent.version
                    
                    env.HELM_REPO_AUTHENTICATION = helmPresent.repoAuthentication

                    if(env.ACTION == 'ROLLBACK'){
                        env.HELM_ROLLBACK_VERSION = helmPresent.rollbackVersion
                    }

                     if(env.DEPLOYMENT_TYPE == 'OPENSHIFT' && env.HELM_CHART_TYPE == 'lazsa'){
                        env.HOST_NAME = kubePresent.oc.route.host
                        
                    }

                    if(env.HELM_REPO_AUTHENTICATION == 'true'){
                        env.HELM_REPO_URL = helmPresent.repoURL
                        env.HELM_REPO_USERNAME = helmPresent.repoUsername
                        env.HELM_REPO_PASSWORD = helmPresent.repoPassword
                    }
                    
                    sh '''
                        set +x
                        if [[ $RELEASE_NAME =~ ^[A-Za-z0-9-]+$ ]]; then
                            echo "Helm release name is valid"
                        else
                            echo "Helm release name is not valid"
                            exit 1;
                        fi
                        set -x
                    '''                   

                }
            }
        }
        stage('HELM Install') {
            when {
                    environment name: 'ACTION', value: 'DEPLOY'
            }
            steps {
                withCredentials([file(credentialsId: "$KUBERNETES_SECRET", variable: 'KUBECONFIG'), usernamePassword(credentialsId: "$ARTIFACTORY_SECRET", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                   script {
                       sh ''' echo "Helm Installation.." '''
                       sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" "$KUBECTL_IMAGE_VERSION"  create namespace "$KUBERNETES_NAMESPACE" || true '''
                       script {      
                              if (env.DEPLOYMENT_TYPE == 'OPENSHIFT') {
                                    sh '''
                                        set +x
                                        COUNT=$(grep 'serviceAccountName' Helm.yaml | wc -l)
                                        if [[ $COUNT -gt 0 ]]
                                        then
                                            ACCOUNT=$(grep 'serviceAccountName:' Helm.yaml | tail -n1 | awk '{ print $2}')
                                            echo $ACCOUNT
                                        else
                                            ACCOUNT='default'
                                        fi
                                        docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $OC_IMAGE_VERSION oc adm policy add-scc-to-user anyuid -z $ACCOUNT -n "$KUBERNETES_NAMESPACE"
                                        set -x
                                    '''
                                }
                       }
                       
                       if(env.ARTIFACTORY != 'ECR'){
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" "$KUBECTL_IMAGE_VERSION" -n "$KUBERNETES_NAMESPACE" delete secret regcred || true '''
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" "$KUBECTL_IMAGE_VERSION" -n "$KUBERNETES_NAMESPACE" create secret docker-registry regcred --docker-server="$ARTIFACTORY_URL" --docker-username="\"$USERNAME\"" --docker-password="\"$PASSWORD\"" || true '''
                       }
                       

                       if (env.HELM_CHART_TYPE == 'lazsa') {
                          sh ''' echo "actual helm installation for lazsa" '''
                          sh ''' cat Helm.yaml '''
                          if (env.DEPLOYMENT_TYPE == 'OPENSHIFT' && env.CONTEXT_PATH != '') {
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION template "$RELEASE_NAME" ./$HELM_CHART_LOCATION -n "$KUBERNETES_NAMESPACE" --values ./helm_chart/values.yaml --set ingress.enabled=false --set oc.route.enabled="true" --set oc.route.host=$HOST_NAME --set context=$CONTEXT_PATH $HELM_VALUES -f Helm.yaml'''
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION upgrade --install $HELM_ARGUMENTS "$RELEASE_NAME" ./$HELM_CHART_LOCATION -n "$KUBERNETES_NAMESPACE" --values ./helm_chart/values.yaml --set ingress.enabled="false" --set oc.route.enabled="true" --set oc.route.host=$HOST_NAME --set context=$CONTEXT_PATH $HELM_VALUES -f Helm.yaml'''
                          }
                          else if (env.DEPLOYMENT_TYPE == 'OPENSHIFT' && env.CONTEXT_PATH == ''){
                              // Openshift deployment  without context Path 
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION template "$RELEASE_NAME" ./$HELM_CHART_LOCATION -n "$KUBERNETES_NAMESPACE" --values ./helm_chart/values.yaml --set ingress.enabled=false --set oc.route.enabled="true" --set oc.route.host=$HOST_NAME $HELM_VALUES -f Helm.yaml'''
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION upgrade --install $HELM_ARGUMENTS "$RELEASE_NAME" ./$HELM_CHART_LOCATION -n "$KUBERNETES_NAMESPACE" --values ./helm_chart/values.yaml --set ingress.enabled="false" --set oc.route.enabled="true" --set oc.route.host=$HOST_NAME $HELM_VALUES -f Helm.yaml'''
                          }
                          else if (env.CONTEXT_PATH == '') {
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION upgrade --install $HELM_ARGUMENTS "$RELEASE_NAME" ./$HELM_CHART_LOCATION -n "$KUBERNETES_NAMESPACE" --values ./helm_chart/values.yaml $HELM_VALUES -f Helm.yaml'''
                          }
                          else{
                            sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION upgrade --install $HELM_ARGUMENTS "$RELEASE_NAME" ./$HELM_CHART_LOCATION -n "$KUBERNETES_NAMESPACE" --values ./helm_chart/values.yaml --set ingress.enabled="true" --set ingress.hosts[0].paths[0].path=$CONTEXT_PATH $HELM_VALUES -f Helm.yaml'''
                          }
                       }
                       else if (env.HELM_CHART_TYPE == 'external') {
                          sh ''' echo "actual helm installation for lazsa" '''
                          sh ''' cat Helm.yaml '''
                          sh ''' docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION upgrade --install $HELM_ARGUMENTS "$RELEASE_NAME" ./$HELM_CHART_LOCATION -n "$KUBERNETES_NAMESPACE" $HELM_VALUES -f Helm.yaml'''
                       }
                       else if (env.HELM_REPO_AUTHENTICATION == 'true' && env.HELM_CHART_TYPE == 'chart') {
                          sh ''' docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION repo add lazsa $HELM_REPO_URL --username $HELM_REPO_USERNAME --password $HELM_REPO_PASSWORD '''
                          sh ''' docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION repo update '''
                          sh ''' docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION upgrade --install $HELM_ARGUMENTS "$RELEASE_NAME" $HELM_CHART_LOCATION --version=$HELM_CHART_VERSION -n "$KUBERNETES_NAMESPACE" $HELM_VALUES -f Helm.yaml'''
                           
                       }
                       else if (env.HELM_CHART_TYPE == 'chart'){
                          sh ''' docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION repo add lazsa $HELM_REPO_URL '''
                          sh ''' docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION repo update '''
                          sh ''' docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION upgrade --install $HELM_ARGUMENTS "$RELEASE_NAME" $HELM_CHART_LOCATION --version=$HELM_CHART_VERSION -n "$KUBERNETES_NAMESPACE" $HELM_VALUES -f Helm.yaml'''                          
                       }                       
                        
                    }
  
                }
                    
            }
        }

        stage('HELM uninstall') {
            when {
                    environment name: 'ACTION', value: 'DESTROY'
            }
            steps {
                  withCredentials([file(credentialsId: "$KUBERNETES_SECRET", variable: 'KUBECONFIG'), usernamePassword(credentialsId: "$ARTIFACTORY_SECRET", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    script{
                            sh '''
                                set +x
                                export RELEASE_STATUS=$(docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -w /apps $HELM_IMAGE_VERSION status $RELEASE_NAME -n "$KUBERNETES_NAMESPACE")
                                echo $RELEASE_STATUS
                                if [[ "$RELEASE_STATUS" != "Error: release: not found" ]]
                                then
                                    echo "Release Present"
                                    docker run --rm --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -w /apps $HELM_IMAGE_VERSION uninstall $RELEASE_NAME -n "$KUBERNETES_NAMESPACE"
                                else
                                    echo "Release not found"
                                fi
                                set -x
                            '''
                }
              }
            }
        }


//         stage('HELM rollback') {
//             when {
//                     environment name: 'ACTION', value: 'ROLLBACK'
//             }
//             steps {
//                 withCredentials([file(credentialsId: "$KUBERNETES_SECRET", variable: 'KUBECONFIG'), usernamePassword(credentialsId: "$ARTIFACTORY_SECRET", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
//                     script{
//                             sh '''
//                                 export RELEASE_STATUS=$(docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION status $RELEASE_NAME)
//                                 echo $RELEASE_STATUS
//                                 if [[ "$RELEASE_STATUS" != "Error: release: not found" ]]
//                                 then
//                                     echo "Release Present, going for Rollback"
//                                     docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION rollback $RELEASE_NAME $HELM_ROLLBACK_VERSION
//                                     docker run --rm  -v $PWD:/root --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $HELM_IMAGE_VERSION history $RELEASE_NAME
//                                 else
//                                     echo "Release not found"
//                                 fi
//                             '''

//             }
//         }

//     }
//  }
}
    post {
            always {
                echo 'Running Cleanup Step'
                echo 'Cleaning Jenkins Workspace'
                cleanWs()
                echo 'Removing all Docker Images'
                sh 'docker rmi -f $(docker image ls -q) || true'
                echo 'Removing all Docker Containers'
                sh 'docker rm -f $(docker ps -a -q) || true'
            }
        }
}

