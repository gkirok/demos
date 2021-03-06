label = "${UUID.randomUUID().toString()}"
BUILD_FOLDER = "/go"
expired=240
git_project = "demos"
git_project_user = "v3io"
git_deploy_user_token = "iguazio-prod-git-user-token"
git_deploy_user_private_key = "iguazio-prod-git-user-private-key"

podTemplate(label: "${git_project}-${label}", yaml: """
apiVersion: v1
kind: Pod
metadata:
  name: "${git_project}-${label}"
  labels:
    jenkins/kube-default: "true"
    app: "jenkins"
    component: "agent"
spec:
  shareProcessNamespace: true
  containers:
    - name: jnlp
      image: jenkins/jnlp-slave
      resources:
        limits:
          cpu: 1
          memory: 2Gi
        requests:
          cpu: 1
          memory: 2Gi
      volumeMounts:
        - name: go-shared
          mountPath: /go
    - name: docker-cmd
      image: docker
      command: [ "/bin/sh", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run
        - name: go-shared
          mountPath: /go
  volumes:
    - name: docker-sock
      hostPath:
          path: /var/run
    - name: go-shared
      emptyDir: {}
"""
) {
    node("${git_project}-${label}") {
        withCredentials([
                string(credentialsId: git_deploy_user_token, variable: 'GIT_TOKEN')
        ]) {
            def TAG_VERSION
            def DOCKER_TAG_VERSION
            pipelinex = library(identifier: 'pipelinex@DEVOPS-204-pipelinex', retriever: modernSCM(
                    [$class: 'GitSCMSource',
                     credentialsId: git_deploy_user_private_key,
                     remote: "git@github.com:iguazio/pipelinex.git"])).com.iguazio.pipelinex
            multi_credentials=[pipelinex.DockerRepo.ARTIFACTORY_IGUAZIO, pipelinex.DockerRepo.DOCKER_HUB, pipelinex.DockerRepo.QUAY_IO]

            common.notify_slack {
                stage('get tag data') {
                    container('jnlp') {
                        TAG_VERSION = github.get_tag_version(TAG_NAME)
                        DOCKER_TAG_VERSION = github.get_docker_tag_version(TAG_NAME)
                        PUBLISHED_BEFORE = github.get_tag_published_before(git_project, git_project_user, "${TAG_VERSION}", GIT_TOKEN)

                        echo "$TAG_VERSION"
                        echo "$PUBLISHED_BEFORE"
                    }
                }

                if (TAG_VERSION != null && TAG_VERSION.length() > 0 && PUBLISHED_BEFORE < expired) {
                    stage('prepare sources') {
                        container('jnlp') {
                            dir("${BUILD_FOLDER}/src/github.com/v3io/${git_project}") {
                                git(changelog: false, credentialsId: git_deploy_user_private_key, poll: false, url: "git@github.com:${git_project_user}/${git_project}.git")
                                sh("git checkout ${TAG_VERSION}")
                            }
                        }
                    }

                    parallel(
                            'build netops-demo-golang': {
                                container('docker-cmd') {
                                    dir("${BUILD_FOLDER}/src/github.com/v3io/${git_project}/netops/golang/src/github.com/v3io/demos") {
                                        sh("docker build . --tag netops-demo-golang:${DOCKER_TAG_VERSION} --build-arg NUCLIO_BUILD_OFFLINE=true --build-arg NUCLIO_BUILD_IMAGE_HANDLER_DIR=github.com/v3io/demos")
                                    }
                                }

                                container('docker-cmd') {
                                    dockerx.images_push_multi_registries(["netops-demo-golang:${DOCKER_TAG_VERSION}"], multi_credentials)
                                }
                            },

                            'build netops-demo-py': {
                                container('docker-cmd') {
                                    dir("${BUILD_FOLDER}/src/github.com/v3io/${git_project}/netops/py") {
                                        sh("docker build . --tag netops-demo-py:${DOCKER_TAG_VERSION}")
                                    }
                                }

                                container('docker-cmd') {
                                    dockerx.images_push_multi_registries(["netops-demo-py:${DOCKER_TAG_VERSION}"], multi_credentials)
                                }
                            }
                    )

                    stage('update release status') {
                        container('jnlp') {
                            github.update_release_status(git_project, git_project_user, "${TAG_VERSION}", GIT_TOKEN)
                        }
                    }
                } else {
                    stage('warning') {
                        if (PUBLISHED_BEFORE >= expired) {
                            echo "Tag too old, published before $PUBLISHED_BEFORE minutes."
                        } else {
                            echo "${TAG_VERSION} is not release tag."
                        }
                    }
                }
            }
        }
    }
}
