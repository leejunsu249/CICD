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
      name: 'docker',
      image: 'docker',
      command: 'cat',
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'argo',
      image: 'argoproj/argo-cd-ci-builder:latest',
      command: 'cat',
      ttyEnabled: true
    ),
    // containerTemplate(
    //   name: 'trivy',
    //   image: 'aquasec/trivy',
    //   command: 'cat',
    //   ttyEnabled: true
    // ),
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]
) 
{
  node('docker-build') {
    stage('Checkout') {
      container('docker') {
        checkout scm
      }
    }

    stage('Docker Build') {
      dir(path: 'container') {
        container('docker') {
          image = docker.build("${registry}/test:${imageTag}", "--build-arg COLOR=${imageTag} .")
        }
      }
    }

    stage('Image Scan') {
      container('trivy') {
          sh 'trivy --severity CRITICAL,HIGH --format json ${registry}/test:${imageTag}'
      }
    }

    stage('Push image') {
      container('docker') {
        docker.withRegistry("https://${registry}", 'podman-key') {
          image.push()
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
