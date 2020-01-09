#!groovy

def label = "jenkins"
env.DOCKERHUB_IMAGE = "devops53/hello-world"

podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-slave
  namespace: jenkins
  labels:
    component: ci
    jenkins: jenkins-agent
spec:
  serviceAccountName: jenkins
  volumes:
  - name: dind-storage
    emptyDir: {}
  containers:
  - name: git
    image: alpine/git
    command:
    - cat
    tty: true
  - name: nodejs
    image: node:latest
    command:
    - cat
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
  - name: docker
    image: docker:19.03.3-git
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_HOST
      value: tcp://docker-dind:2375
    volumeMounts:
    - mountPath: /var/lib/docker
      name: dind-storage 
  - name: helm
    image: lachlanevenson/k8s-helm:v2.16.1
    command:
    - cat
    tty: true
"""
  )

        {

            node(label) {
                stage('Clone another repo master') {
                    checkout ( [$class: 'GitSCM',
                                branches: [[name: ${params.BRANCH_NAME} ]],
                                userRemoteConfigs: [[
                                                            credentialsId: 'gitHub_key',
                                                            url: "https://github.com/empikls/node.is"]]])
                }
        }
}


                def tagDockerImage
                def nameStage
                def hostname
                def job

                if ( isMaster() ) {
                    stage('Deploy dev version') {
                        nameStage = "app-dev"
                        namespace = "dev"
                        tagDockerImage = readFile('GIT_COMMIT').take(7)
                        hostname = "dev-184-173-46-252.nip.io"
                        deploy(nameStage, namespace, tagDockerImage, hostname)
                    }
                }
                def isMaster() {
                    return (env.BRANCH_NAME == "master" )
                }


def deploy( appName, namespace, tagName, hostName ) {
    container('helm') {
        echo "Release image: ${shortCommit}"
        echo "Deploy app name: $appName"
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh """
         helm upgrade --install $appName --debug --force ./app \
            --namespace=$namespace \
            --set image.tag="$tagName" \
            --set ingress.hostName=$hostName \
            --set-string ingress.tls[0].hosts[0]="$hostName" \
            --set-string ingress.tls[0].secretName=acme-$appName-tls 
          """
        }
    }
}
