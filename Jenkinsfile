pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: build
    image: ubuntu:24.04
    command:
    - sleep
    args:
    - 99d
'''
        }
    }

    environment {
        CMAKE_BUILD_TYPE = 'Release'
    }

    stages {
        stage('Checkout') {
            steps {
                container('build') {
                    checkout([
                        $class: 'GitSCM',
                        branches: scm.branches,
                        userRemoteConfigs: scm.userRemoteConfigs,
                        extensions: [[$class: 'SubmoduleOption', disableSubmodules: true]]
                    ])
                }
            }
        }

        stage('Setup Tools') {
            steps {
                container('build') {
                    sh '''
                        apt-get update && apt-get install -y build-essential cmake cppcheck clang-format git coreutils python3-pyparsing python3-junit.xml awscli

                        # Install the AWS SSM Session Manager Plugin - test
                        curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
                        dpkg -i session-manager-plugin.deb
                    '''
                }
            }
        }

        stage('Update Submodules') {
            steps {
                container('build') {
                    sh 'git config --global --add safe.directory ${WORKSPACE}'
                    sh 'git submodule update --init --recursive'
                }
            }
        }

        stage('CI') {
            parallel {
                stage('Test') {
                    steps {
                        container('build') {
                            sh 'make test-ci'
                            junit 'test-results.xml'
                        }
                    }
                }
                stage('Lint') {
                    steps {
                        container('build') {
                            sh 'cppcheck --enable=all --error-exitcode=1 --suppress=missingIncludeSystem src/ || true'
                            sh 'find src/ include/ tests/ -name "*.c" -o -name "*.h" | xargs clang-format --dry-run -Werror || true'
                        }
                    }
                }
                stage('Scan') {
                    steps {
                        container('build') {
                            // TODO: Add Black Duck SCA scan
                            // This will require credentials to be configured in Jenkins
                            echo 'TODO: Run Black Duck SCA scan'
                            script {
                                env.MY_VAR = "Hello from CI"
                                env.BUILD_NUMBER = env.BUILD_NUMBER
                            }
                        }
                    }
                }
            }
        }

        stage('Build and Deploy') {
            when {
                branch 'main'
            }
            steps {
                container('build') {
                    script {
                        def commitCount = sh(script: 'git rev-list --count HEAD', returnStdout: true).trim()
                        def shortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        env.VERSION = "1.0.0-${commitCount}-${shortSha}"
                        echo "Setting version to: ${env.VERSION}"
                    }
                    sh 'make clean'
                    sh 'make release'
                    sh """
                        mkdir -p dist
                        cp build/src/demo-firmware dist/
                        cp README.md dist/
                        tar -czf demo-firmware-${env.VERSION}.tar.gz -C dist .
                    """
                    archiveArtifacts artifacts: "demo-firmware-${env.VERSION}.tar.gz", fingerprint: true

                    script {
                        def digest = sh(script: "sha256sum demo-firmware-${env.VERSION}.tar.gz | awk '{print \$1}'", returnStdout: true).trim()

                        def buildArtifactId = registerBuildArtifactMetadata(
                            name: env.JOB_NAME,
                            version: env.VERSION,
                            type: 'Binary',
                            label: 'main,c,cmake,firmware',
                            url: "${env.BUILD_URL}artifact/demo-firmware-${env.VERSION}.tar.gz",
                            digest: digest
                        )
                        env.ARTIFACT_ID = buildArtifactId
                        echo "Build artifact registered with CloudBees Unify"
                        echo "Artifact ID: ${env.ARTIFACT_ID}"
                    }

                    // TODO: Add CloudBees Platform evidence publishing
                    echo 'TODO: Publish Build Evidence'
                }
            }
        }

//        stage('Deploy to EC2') {
//            when {
//                branch 'main'
//            }
//            steps {
//                container('build') {
//                    script {
//                        withCredentials([sshUserPrivateKey(credentialsId: 'ec2key', keyFileVariable: 'SSH_KEY')]) {
//                            sh """
//                                # Ensure ssh is available
//                                apt-get update && apt-get install -y openssh-client
//
//                                # Stop any running firmware instance
//                                ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@ec2-54-198-197-52.compute-1.amazonaws.com '
//                                    if pgrep -f demo-firmware > /dev/null; then
//                                        echo "Stopping existing firmware instance..."
//                                        pkill -SIGTERM -f demo-firmware || true
//                                        sleep 2
//                                    fi
//                                '
//
//                                # Copy artifact to EC2
//                                scp -i \${SSH_KEY} -o StrictHostKeyChecking=no demo-firmware-${env.VERSION}.tar.gz ec2-user@ec2-54-198-197-52.compute-1.amazonaws.com:/tmp/
//
//                                # Extract and deploy
//                                ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@ec2-54-198-197-52.compute-1.amazonaws.com '
//                                    mkdir -p ~/demo-firmware
//                                    cd ~/demo-firmware
//                                    tar -xzf /tmp/demo-firmware-${env.VERSION}.tar.gz
//                                    chmod +x demo-firmware
//
//                                    # Start firmware in background
//                                    nohup ./demo-firmware > firmware.log 2>&1 &
//                                    echo \$! > firmware.pid
//
//                                    echo "Firmware deployed and started (PID: \$(cat firmware.pid))"
//                                    echo "Version: ${env.VERSION}"
//
//                                    # Wait a moment and check if it started successfully
//                                    sleep 2
//                                    if pgrep -f demo-firmware > /dev/null; then
//                                        echo "Firmware is running successfully"
//                                        tail -n 20 firmware.log
//                                    else
//                                        echo "ERROR: Firmware failed to start"
//                                        tail -n 50 firmware.log
//                                        exit 1
//                                    fi
//                                '
//                            """
//                        }
//                    }
//                }
//            }
//        }
stage('Deploy to EC2 via SSM') {
            when {
                branch 'main'
            }
            steps {
                container('build') {
                    script {
                        // IMPORTANT: You must inject AWS credentials here so Jenkins can call the SSM API!
                        // You can use a Jenkins credentials binding (like below) or environment variables.
                        withCredentials([
                            sshUserPrivateKey(credentialsId: 'ec2key', keyFileVariable: 'SSH_KEY'),
                            // Uncomment and configure this if you have AWS keys stored in Jenkins:
                            // usernamePassword(credentialsId: 'aws-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')
                        ]) {
                            
                            // Define your target instance ID
                            env.EC2_INSTANCE_ID = "i-08c8a16ee23becca4"
                            
                            sh """
                                # 1. Ensure SSH is available
                                apt-get update && apt-get install -y openssh-client

                                # 2. Configure SSH to tunnel through AWS SSM automatically for any EC2 Instance ID
                                mkdir -p ~/.ssh
                                echo "host i-* mi-*" > ~/.ssh/config
                                echo "    ProxyCommand sh -c \\"aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'\\"" >> ~/.ssh/config
                                chmod 600 ~/.ssh/config

                                # 3. Stop any running firmware instance (Notice we use EC2_INSTANCE_ID now!)
                                ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@\${EC2_INSTANCE_ID} '
                                    if pgrep -f demo-firmware > /dev/null; then
                                        echo "Stopping existing firmware instance..."
                                        pkill -SIGTERM -f demo-firmware || true
                                        sleep 2
                                    fi
                                '

                                # 4. Copy artifact to EC2 via SSM Tunnel!
                                scp -i \${SSH_KEY} -o StrictHostKeyChecking=no demo-firmware-${env.VERSION}.tar.gz ec2-user@\${EC2_INSTANCE_ID}:/tmp/

                                # 5. Extract and deploy
                                ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@\${EC2_INSTANCE_ID} '
                                    mkdir -p ~/demo-firmware
                                    cd ~/demo-firmware
                                    tar -xzf /tmp/demo-firmware-${env.VERSION}.tar.gz
                                    chmod +x demo-firmware

                                    # Start firmware in background
                                    nohup ./demo-firmware > firmware.log 2>&1 &
                                    echo \$! > firmware.pid

                                    echo "Firmware deployed and started (PID: \$(cat firmware.pid))"
                                    echo "Version: ${env.VERSION}"

                                    # Wait a moment and check if it started successfully
                                    sleep 2
                                    if pgrep -f demo-firmware > /dev/null; then
                                        echo "Firmware is running successfully"
                                    else
                                        echo "ERROR: Firmware failed to start"
                                        tail -n 50 firmware.log
                                        exit 1
                                    fi
                                '
                            """
                        }
                    }
                }
            }
        }
    }
}
