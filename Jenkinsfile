#!groovy
@Grab('org.yaml:snakeyaml:1.17')
import org.yaml.snakeyaml.Yaml

def label = "jenkins"


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

                stage('Clone config repo') {
                    checkout scm

                    Yaml parser = new Yaml()
                    def valuesStr = readFile("${env.WORKSPACE}/values.yaml")
                    println "values text: ${valuesStr?.isEmpty()}"
                    def values = parser.load(valuesStr)
                    values.each{println it.tag}
                }

                stage('Clone another repo master') {
                    checkout([$class           : 'GitSCM',
                              branches         : [[name: "${params.COMMIT}"]],
                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                    sh 'git rev-parse HEAD > GIT_COMMIT'
                    shortCommit = readFile('GIT_COMMIT').take(7)
                    echo "${shortCommit}"
                    echo "${params.TAG}"
                    echo "${params.COMMIT}"
                }

//                    Yaml parser = new Yaml()
//                    List values = parser.load(("values.yaml" as File).text)
//
//                    values.each{println it.tag}
//                stage('Deploy DEV release') {
//                    if (isMaster()) {
//                        nameStage = "app-dev"
//                        namespace = "dev"
//                        tagDockerImage = readFile('GIT_COMMIT').take(7)
//                        hostname = "dev-173-193-112-65.nip.io"
//                        deploy(nameStage, namespace, tagDockerImage, hostname)
//                    }
//                }
//                stage('Deploy QA release') {
//                    if (isBuildingTag()) {
//                        nameStage = "app-qa"
//                        namespace = "qa"
//                        tagDockerImage = "${params.TAG}"
//                        hostname = "qa-173-193-112-65.nip.io"
//                        deploy(nameStage, namespace, tagDockerImage, hostname)
//                    }
//                }
                stage('Deploy PROD release') {
                    container('helm') {
                        echo "Release image: ${shortCommit}"
//                        echo "Deploy app name: $appName"
                        withKubeConfig([credentialsId: 'kubeconfig']) {
                            sh """
                            helm upgrade --install prod --debug ./App/app --values ./values.yaml
                            """
                        }
                    }
                }
            }
        }

                def tagDockerImage
                def nameStage
                def hostname


//                if (isMaster()) {
//                    stage('Deploy dev version') {
//                        nameStage = "app-dev"
//                        namespace = "dev"
//                        tagDockerImage = readFile('GIT_COMMIT').take(7)
//                        hostname = "dev-173-193-112-65.nip.io"
//                        deploy(nameStage, namespace, tagDockerImage, hostname)
//                    }
//                }
//                if (isBuildingTag()) {
//                    stage('Deploy to QA stage') {
//                        nameStage = "app-qa"
//                        namespace = "qa"
//                        tagDockerImage = "${params.TAG}"
//                        hostname = "qa-173-193-112-65.nip.io"
//                        deploy(nameStage, namespace, tagDockerImage, hostname)
//                            }
//                        }
//                }
                boolean isMaster() {
                    return ("${params.TAG}" == "master" )
                }
                boolean isBuildingTag() {
                    return ("${params.TAG}" ==~ /^v\d.\d.\d$/ || "${params.TAG}" ==~ /^\d.\d.\d$/ )
                }

//                boolean isChangeSet() {

//                currentBuild.changeSets.any { changeSet ->
//                    changeSet.items.any { entry ->
//                        entry.affectedFiles.any { file ->
//                            if (file.path.equals("values.yaml")) {
//                                return true
//                                }
//                            }
//                        }
//                    }
//                }
//                def deploy( appName, namespace, tagName, hostName ) {
//                    container('helm') {
//                        echo "Release image: ${shortCommit}"
//                        echo "Deploy app name: $appName"
//                        withKubeConfig([credentialsId: 'kubeconfig']) {
//                            sh """
//                         helm upgrade --install $appName --debug --force ./app \
//                            --namespace=$namespace \
//                            --set image.tag="$tagName" \
//                            --set ingress.hostName=$hostName \
//                            --set-string ingress.tls[0].hosts[0]="$hostName" \
//                            --set-string ingress.tls[0].secretName=acme-$appName-tls
//                          """
//                        }
//                    }
//                }
