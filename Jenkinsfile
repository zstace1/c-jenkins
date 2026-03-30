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
                        apt-get update && apt-get install -y build-essential cmake cppcheck clang-format git coreutils python3-pyparsing python3-junit.xml
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
                            sh 'cppcheck --enable=all --error-exitcode=1 --suppress=missingIncludeSystem src/ 2>&1 | grep -E "(error|warning):" || echo "cppcheck: no issues found"'
                            sh '''
                                echo "Checking code formatting..."
                                UNFORMATTED=$(find src/ include/ tests/ \\( -name "*.c" -o -name "*.h" \\) -exec clang-format --dry-run -Werror {} \\; 2>&1 | wc -l)
                                if [ "$UNFORMATTED" -eq "0" ]; then
                                    echo "✓ All files are properly formatted"
                                else
                                    echo "⚠ Found $UNFORMATTED formatting issues (run format-code.sh to fix)"
                                fi
                            '''
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

        stage('Discover Egress IPs') {
            when {
                branch 'main'
            }
            steps {
                container('build') {
                    sh '''
                        apt-get update && apt-get install -y curl dnsutils

                        echo "=== Pod IP Information ==="
                        hostname -i || echo "hostname -i not available"

                        echo ""
                        echo "=== Public Egress IP (what EC2 will see) ==="
                        curl -s https://checkip.amazonaws.com || echo "Could not reach AWS checkip"
                        curl -s https://ifconfig.me || echo "Could not reach ifconfig.me"
                        curl -s https://api.ipify.org || echo "Could not reach ipify"

                        echo ""
                        echo "=== All Network Interfaces ==="
                        apt-get install -y net-tools
                        ifconfig || ip addr show

                        echo ""
                        echo "=== Route Information ==="
                        ip route show || route -n

                        echo ""
                        echo "=== Node Information (if available) ==="
                        echo "NODE_NAME: ${NODE_NAME:-not set}"
                        echo "POD_NAME: ${POD_NAME:-not set}"
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                branch 'main'
            }
            steps {
                container('build') {
                    script {
                        withCredentials([sshUserPrivateKey(credentialsId: 'ec2key', keyFileVariable: 'SSH_KEY')]) {
                            sh """
                                # Ensure ssh is available
                                apt-get update && apt-get install -y openssh-client

                                echo "=== Deploying to EC2 ==="
                                ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no -o ConnectTimeout=10 ec2-user@ec2-3-81-36-238.compute-1.amazonaws.com 'echo "SSH connection successful"'

                                # Stop any running firmware instance
                                echo "Stopping any existing firmware..."
                                ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@ec2-3-81-36-238.compute-1.amazonaws.com 'bash -c "pkill -SIGTERM -f demo-firmware; sleep 2; echo Stop complete"' || true

                                # Copy artifact to EC2
                                echo "Copying artifact..."
                                scp -i \${SSH_KEY} -o StrictHostKeyChecking=no demo-firmware-${env.VERSION}.tar.gz ec2-user@ec2-3-81-36-238.compute-1.amazonaws.com:/tmp/

                                # Extract and deploy
                                echo "Deploying and starting firmware..."
                                ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@ec2-3-81-36-238.compute-1.amazonaws.com "bash -c 'mkdir -p ~/demo-firmware && cd ~/demo-firmware && tar -xzf /tmp/demo-firmware-${env.VERSION}.tar.gz && chmod +x demo-firmware && nohup ./demo-firmware > firmware.log 2>&1 & echo \$! > firmware.pid && sleep 2 && if pgrep -f demo-firmware >/dev/null; then echo Firmware started successfully with PID \$(cat firmware.pid); ls -lh demo-firmware firmware.log 2>/dev/null || true; else echo ERROR: Firmware failed to start; cat firmware.log 2>/dev/null || echo No log file found; exit 1; fi'"
                            """
                        }

                        // Register deployment with CloudBees Unify
                        echo "Registering deployment with CloudBees Unify..."
                        echo "ARTIFACT_ID: ${env.ARTIFACT_ID}"

                        registerDeployedArtifactMetadata(
                            //id: env.ARTIFACT_ID,
                            targetEnvironment: 'Development',
                            url: "${env.BUILD_URL}artifact/demo-firmware-${env.VERSION}.tar.gz"
                            //label: "ec2,aws"
                        )
                        echo "Deployment registered successfully"
                    }
                }
            }
        }
    }
}
