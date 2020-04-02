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
                        go build -o /home/jenkins/agent/src/github.com/eclipse/codewind-operator/build/_output/bin/codewind-operator -gcflags all=-trimpath=/home/jenkins/agent/src/github.com/eclipse -asmflags all=-trimpath=/home/jenkins/agent/src/github.com/eclipse github.com/eclipse/codewind-operator/cmd/manager

                        # clean up the cache directory
                        cd ../../
                        rm -rf .cache
                    '''
                }
            }
        }

        stage('Test') {
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
                echo 'Starting tests'
                script {
                    sh '''
                        set -x
                        // Just check the operator binary is still there.
                        ls -lrt /home/jenkins/agent/src/github.com/eclipse/codewind-operator/build/_output/bin/codewind-operator
                    '''
                }
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
                        // Just check the operator binary is still there.
                        ls -lrt /home/jenkins/agent/src/github.com/eclipse/codewind-operator/build/_output/bin/codewind-operator
                    //     # switch to the code go directory
                    //     cd ../../$CODE_DIRECTORY_FOR_GO
                    //     echo $(pwd)
                    //     if [ -d codewind-installer ]; then
                    //         rm -rf codewind-installer
                    //     fi
                    //     mkdir codewind-installer

                    //     TIMESTAMP="$(date +%F-%H%M)"
                    //     # WINDOWS EXE: Submit Windows unsigned.exe and save signed output to signed.exe

                    //     # only sign windows exe if not a pull request
                    //     if [ -z $CHANGE_ID ]; then
                    //         curl -o codewind-installer/cwctl-win-${TIMESTAMP}.exe  -F file=@cwctl-win.exe http://build.eclipse.org:31338/winsign.php
                    //         rm cwctl-win.exe
                    //     fi
                    //     # move other executable to codewind-installer directory and add timestamp to the name
                    //     for fileid in cwctl-*; do
                    //         mv -v $fileid codewind-installer/${fileid}-$TIMESTAMP
                    //     done

                    //     DEFAULT_WORKSPACE_DIR=$(cat $DEFAULT_WORKSPACE_DIR_FILE)
                    //     mkdir $DEFAULT_WORKSPACE_DIR/codewind-installer
                    //     cp -r codewind-installer/* $DEFAULT_WORKSPACE_DIR/codewind-installer
                    '''
                    // // stash the executables so they are avaialable outside of this agent
                    // dir('codewind-installer') {
                    //     sh 'echo "Stashing: $(ls -lA cwctl*)"'
                    //     stash includes: 'cwctl*', name: 'EXECUTABLES'
                    }
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
