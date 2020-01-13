#!groovy


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
                    def values = readYaml(file: 'values.yaml')
                    println "tag from yaml: ${values.image.tag}"
                    tagDockerImageFromFile = "${values.image.tag}"
                }

                stage('Clone another repo master') {
                    def values = readYaml(file: 'values.yaml')
                    println "tag from yaml: ${values.image.tag}"
                    if (isChangeSet()) {
                        checkout([$class           : 'GitSCM',
                                  branches         : [[name: "${values.image.tag}"]],
                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                    }
                        if (isBuildingTag() ) {
                        checkout([$class           : 'GitSCM',
                                  branches         : [[name: "${params.TAG}"]],
                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                        }
                    else {
                        checkout([$class           : 'GitSCM',
                                  branches         : [[name: "${params.COMMIT}"]],
                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                    }
                }
                echo "${params.TAG}"
                echo "${params.COMMIT}"


                stage('Deploy DEV release') {
                    if (isMaster()) {
                        container('helm') {
                            withKubeConfig([credentialsId: 'kubeconfig']) {
                                sh """
                         helm upgrade --install app-dev --debug --force ./App/app \
                            --namespace=dev \
                            --set image.tag="${params.COMMIT}" \
                            --set ingress.hostName="dev-173-193-112-65.nip.io" \
                            --set-string ingress.tls[0].hosts[0]="dev-173-193-112-65.nip.io" \
                            --set-string ingress.tls[0].secretName=acme-app-dev-tls
                          """
                            }
                        }
                    }
                }
                stage('Deploy QA release') {
                    if (isBuildingTag()) {
                        container('helm') {
                            withKubeConfig([credentialsId: 'kubeconfig']) {
                                sh """
                         helm upgrade --install app-qa --debug --force ./App/app \
                            --namespace=qa \
                            --set image.tag="${params.TAG}" \
                            --set ingress.hostName="qa-173-193-112-65.nip.io" \
                            --set-string ingress.tls[0].hosts[0]="qa-173-193-112-65.nip.io" \
                            --set-string ingress.tls[0].secretName=acme-app-qa-tls
                          """
                            }
                        }
                    }
                }
                stage('Deploy PROD release') {
                    if (isChangeSet()) {
                        container('helm') {
                            withKubeConfig([credentialsId: 'kubeconfig']) {
                                sh """
                            helm upgrade --install prod --debug ./App/app --values ./values.yaml
                            """
                            }
                        }
                    }
                }
            }
        }


                def tagDockerImage
                def nameStage
                def hostname


                boolean isMaster() {
                    return ("${params.TAG}" == "master" )
                }
                boolean isBuildingTag() {
                    return ("${params.TAG}" ==~ /^v\d.\d.\d$/ || "${params.TAG}" ==~ /^\d.\d.\d$/ )
                }

                def isChangeSet() {
//                currentBuild.changeSets.any { changeSet ->
//                    changeSet.items.any { entry ->
//                        entry.affectedFiles.any { file ->
//                            if (file.path.equals("values.yaml")) {
//                                }
//                            }
//                        }
//                    }
//                }
                    def changeLogSets = currentBuild.changeSets
                    for (int i = 0; i < changeLogSets.size(); i++) {
                        def entries = changeLogSets[i].items
                        for (int j = 0; j < entries.length; j++) {
                            def entry = entries[j]
                            def files = new ArrayList(entry.affectedFiles)
                            for (int k = 0; k < files.size(); k++) {
                                def file = files[k]
                                if (file.path.equals("values.yaml")) {
                                echo " ${file.editType.name} via $file.path"
                            }
                        }
                    }
                }
                }

//                def deploy( appName, namespace, tagName, hostName ) {
//                    container('helm') {
//                        withKubeConfig([credentialsId: 'kubeconfig']) {
//                            sh """
//                         helm upgrade --install $appName --debug --force ./App/app \
//                            --namespace=$namespace \
//                            --set image.tag="$tagName" \
//                            --set ingress.hostName=$hostName \
//                            --set-string ingress.tls[0].hosts[0]="$hostName" \
//                            --set-string ingress.tls[0].secretName=acme-$appName-tls
//                          """
//                        }
//                    }
//              }