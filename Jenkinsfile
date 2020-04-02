#!groovy

pipeline {

    agent {
        kubernetes {
              label 'go-pod-1-13-buster'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: go
    image: golang:1.13-buster
    tty: true
    command:
    - cat
    resources:
      limits:
        memory: "4Gi"
        cpu: "1"
      requests:
        memory: "4Gi"
        cpu: "1"
"""
        }
    }

    options {
        timestamps()
        skipStagesAfterUnstable()
    }

    environment {
        CODE_DIRECTORY_FOR_GO = 'src/github.com/eclipse/codewind-operator'
        DEFAULT_WORKSPACE_DIR_FILE = 'temp_default_dir'
        CODECOV_TOKEN = credentials('codecov-token')
    }

    stages {

        stage ('Build') {

            // This when clause disables Tagged build
            when {
                beforeAgent true
                not {
                    buildingTag()
                }
            }

            steps {
                container('go') {
                    sh '''#!/bin/bash

                        echo "Starting setup for build.....: GOPATH=$GOPATH"
                        set -x
                        whoami

                        # add the base directory to the gopath
                        DEFAULT_CODE_DIRECTORY=$PWD
                        cd ../..
                        export GOPATH=$GOPATH:$(pwd)

                        # create a new directory to store the code for go compile
                        if [ -d $CODE_DIRECTORY_FOR_GO ]; then
                            rm -rf $CODE_DIRECTORY_FOR_GO
                        fi
                        mkdir -p $CODE_DIRECTORY_FOR_GO
                        cd $CODE_DIRECTORY_FOR_GO

                        # copy the code into the new directory for go compile
                        cp -r $DEFAULT_CODE_DIRECTORY/* .
                        echo $DEFAULT_CODE_DIRECTORY >> $DEFAULT_WORKSPACE_DIR_FILE

                        # go cache setup
                        mkdir .cache
                        cd .cache
                        mkdir go-build
                        cd ../

                        # now compile the code
                        export HOME=$JENKINS_HOME
                        export GOCACHE=/home/jenkins/agent/$CODE_DIRECTORY_FOR_GO/.cache/go-build
                        export GOARCH=amd64
                        export GO111MODULE=on
                        go mod tidy

                        # This is copied from the operator SDK build command.
                        go build -o /home/jenkins/agent/$CODE_DIRECTORY_FOR_GO/build/_output/bin/codewind-operator -gcflags all=-trimpath=/home/jenkins/agent/src/github.com/eclipse -asmflags all=-trimpath=/home/jenkins/agent/src/github.com/eclipse github.com/eclipse/codewind-operator/cmd/manager

                        # clean up the cache directory
                        cd ../../
                        rm -rf .cache
                    '''
                    stash name: "operator-binary", includes: "/home/jenkins/agent/src/github.com/eclipse/codewind-operator/build/_output/bin/codewind-operator"
                }
            }
        }

        stage('Build Image') {
            agent {
                label "docker-build"
            }
            // This when clause disables Tagged build
            when {
                beforeAgent true
                not {
                    buildingTag()
                }
            }

            options {
                timeout(time: 10, unit: 'MINUTES')
                retry(3)
            }

            steps {
                echo 'Building Operator Docker Image'
                // withDockerRegistry([url: 'https://index.docker.io/v1/', credentialsId: 'docker.com-bot']) {
                    unstash "operator-binary"
                    script {
                        sh '''
                            set -x
                            # Just check the operator binary is still there.
                            ls -lrt /home/jenkins/agent/src/github.com/eclipse/codewind-operator/build/_output/bin/codewind-operator
                            ls -lrt /home/jenkins/agent/$CODE_DIRECTORY_FOR_GO/build/_output/bin/codewind-operator
                            cd /home/jenkins/agent/$CODE_DIRECTORY_FOR_GO
                            docker build -f build/Dockerfile -t eclipse/codewind-operator:latest .
                            # TODO Make this conditional so we don't push PR builds.
                            docker push eclipse/codewind-operator:latest
                        '''
                    }
                // }
                // container('go') {
                //    sh '''#!/bin/bash
                //         export GOPATH=/go:/home/jenkins/agent

                //         # go cache setup
                //         mkdir .cache
                //         cd .cache
                //         mkdir go-build
                //         cd ../
                //         export GOCACHE=/home/jenkins/agent/$CODE_DIRECTORY_FOR_GO/.cache/go-build
                //         cd ../../$CODE_DIRECTORY_FOR_GO
                //         go test ./... -short -coverprofile=coverage.txt -covermode=count
                //         TEST_RESULT=$?
                //         if [ $TEST_RESULT -ne 0 ]; then
                //             exit $TEST_RESULT
                //         fi

                //         # Report coverage
                //         if [ -n "$CODECOV_TOKEN" ]; then
                //             echo "Reporting coverage to codecov"
                //             bash <(curl -s https://codecov.io/bash) -f ./coverage.txt
                //         else
                //             echo "CODECOV_TOKEN not set, not reporting coverage"
                //         fi

                //         # clean up the cache directory
                //         rm -rf .cache
                //     '''
                // }
                echo 'End of test stage'
            }
        }

        stage('Upload') {
            // This when clause disables Tagged build
            when {
                beforeAgent true
                not {
                    buildingTag()
                }
            }

            steps {
                echo 'Starting Upload'
                script {
                    sh '''
                        set -x
                        # Just check the operator binary is still there.
                        ls -lrt /home/jenkins/agent/src/github.com/eclipse/codewind-operator/build/_output/bin/codewind-operator
                    '''
                    // // stash the executables so they are avaialable outside of this agent
                    // dir('codewind-installer') {
                    //     sh 'echo "Stashing: $(ls -lA cwctl*)"'
                    //     stash includes: 'cwctl*', name: 'EXECUTABLES'
                    // }
                }
            }
        }
        stage('Deploy') {
            // This when clause disables PR/Tag build uploads; you may comment this out if you want your build uploaded.
            when {
                beforeAgent true
                allOf {
                    not {
                        changeRequest()
                    }
                    not {
                        buildingTag()
                    }
                }
            }

            options {
                timeout(time: 120, unit: 'MINUTES')
                retry(3)
            }

            agent any
               steps {
                   sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {

                println("Deploying codewind-installer to download area...")

                // get the stashed executables
                unstash 'EXECUTABLES'

                sh '''

                  '''
               }
           }
        }

    }

    post {
        success {
            echo 'Build SUCCESS'
        }
    }
}
