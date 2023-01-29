def branch = 'master'
def commitMsg = 'CICD-pipeline'
def githubEmail = '75758585@naver.com'
def githubKey = 'github-key'
def githubSSHURL = 'git@github.com:leejunsu249/CICD.git'
def imageTag = 'blue'
def registry = '10.60.200.120:5000'

podTemplate(label: 'docker-build',
  containers: [
    containerTemplate(
      name: 'podman',
      image: '10.60.200.120:5000/c-podman',
      command: 'cat',
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'argo',
      image: 'argoproj/argo-cd-ci-builder:latest',
      command: 'cat',
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'trivy',
      image: 'aquasec/trivy',
      command: 'cat',
      ttyEnabled: true
    ),
  ],
  // volumes: [
  //   hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  // ]
) 
{
  node('docker-build') {
    stage('Checkout') {
      container('podman') {
        checkout scm
      }
    }

    stage('Podman Build') {
      dir(path: 'container') {
        container(name:'podman', shell:'/bin/bash') {
             sh """
                #!/bin/bash

                # Construct Image Name
                IMAGE=${registry}/test:${imageTag}

                podman build -t \${IMAGE} --build-arg COLOR=${imageTag} .
                """
        }
      }
    }

    stage('Image Scan') {
      container('trivy') {
          sh 'trivy --severity CRITICAL,HIGH --format json ${registry}/test:${imageTag}'
      }
    }

    stage('Push image') {
      container(name:'podman', shell:'/bin/bash') {
        withCredentials([usernamePassword(credentialsId: podman-key,
                                               usernameVariable: 'USERNAME',
                                               passwordVariable: 'PASSWORD')]) {
                sh """
                    #!/bin/bash

                    # Construct Image Name
                    IMAGE=${registry}/test:${imageTag}

                    podman login -u ${USERNAME} -p ${PASSWORD} ${registry} --tls-verify=false

                    podman push \${IMAGE} --tls-verify=false
                    """
        }
      }
    }

    stage('Deploy') {
      container('argo') {
        checkout(
                    [
                        $class: 'GitSCM',
                        extensions: scm.extensions,
                        branches: [
                            [
                                name: "*/${branch}"
                            ]
                        ],
                        userRemoteConfigs: [
                            [
                                url: "${githubSSHURL}",
                                credentialsId: "${githubKey}",
                            ]
                        ]
                    ]
                )
        sshagent(credentials: ["${githubKey}"]) {
          sh("""
                        #!/usr/bin/env bash
                        set +x
                        export GIT_SSH_COMMAND="ssh -oStrictHostKeyChecking=no"
                        git config --global user.email ${githubEmail}
                        git checkout ${branch}
                        cd  helm-charts
                        sed -i 's/tag:.*/tag: ${imageTag}/g' values.yaml
                        git commit -a -m ${commitMsg}
                        git remote set-url origin ${githubSSHURL}
                        git push -u origin master
                    """)
        }
      }
    }
  }
}
