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
                    sh 'apt-get update && apt-get install -y build-essential cmake cppcheck clang-format git coreutils python3-pyparsing python3-junit.xml'
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

                        def buildArtifact = registerBuildArtifactMetadata(
                            name: env.JOB_NAME,
                            version: env.VERSION,
                            type: 'Binary',
                            label: 'main,c,cmake,firmware',
                            url: "${env.BUILD_URL}artifact/demo-firmware-${env.VERSION}.tar.gz",
                            digest: digest
                        )
                        env.ARTIFACT_ID = sh(script: "echo \"${buildArtifact}\" | awk -F\"'\" '{print \$2}'", returnStdout: true).trim()
                        echo "Build artifact registered with CloudBees Unify"
                        echo "Artifact ID: ${env.ARTIFACT_ID}"
                    }

                    // TODO: Add CloudBees Platform evidence publishing
                    echo 'TODO: Publish Build Evidence'
                }
            }
        }
    }
}
