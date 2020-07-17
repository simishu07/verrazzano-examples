// Copyright (c) 2020, Oracle and/or its affiliates.
// Licensed under the Universal Permissive License v 1.0 as shown at https://oss.oracle.com/licenses/upl.

pipeline {
    options {
        skipDefaultCheckout()
    }

    agent {
        docker {
            image "${RUNNER_DOCKER_IMAGE}"
            args "${RUNNER_DOCKER_ARGS}"
            registryUrl "${RUNNER_DOCKER_REGISTRY_URL}"
            registryCredentialsId 'ocir-pull-and-push-account'
            label 'small'
        }
    }

    environment {
        DOCKER_CREDS = credentials('github-markxnelns-private-access-token')
        OCR_CREDS = credentials('ocr-pull-and-push-account')
        NETRC_FILE = credentials('netrc')

        OCI_CLI_TENANCY = credentials('oci-tenancy')
        OCI_CLI_USER = credentials('oci-user-ocid')
        OCI_CLI_FINGERPRINT = credentials('oci-api-key-fingerprint')
        OCI_CLI_KEY_FILE = credentials('oci-api-key')
        OCI_CLI_REGION = 'us-phoenix-1'

        // image names and tags are created from these variables:
        REPO = 'docker.pkg.github.com/verrazzano/examples'
        BOBBYS_HELIDON = 'bobbys-helidon-stock-application'
        ROBERTS_HELIDON = 'roberts-helidon-stock-application'
        BOBBYS_COHERENCE = 'bobbys-coherence'
        ROBERTS_COHERENCE = 'roberts-coherence'
        BOBBYS_WEBLOGIC = 'bobbys-front-end'
        BOBS_WEBLOGIC = 'bobs-bookstore-order-manager'
        VERSION = '0.1.6'

        // secrets used during build
        BOB_DB_PASSWORD = credentials('bobs-bookstore-db-password')
        BOB_ADMIN_PASSWORD = credentials('bobs-bookstore-admin-password')

        // access to GitHub Packages Maven Repository
        MAVEN_SETTINGS = credentials('oracle-maven-settings')

        BUCKET_NAME = "build-shared-files"
        GRAALVM_BUNDLE = "graalvm-ee-java11-linux-amd64-20.1.0.1.tar.gz"
        GRAALVM_JDK8_BUNDLE = "graalvm-ee-java8-linux-amd64-19.3.2.tar.gz"
        WEBLOGIC_BUNDLE = "fmw_12.2.1.4.0_wls.jar"
        IMAGETOOL_BUNDLE = "imagetool.zip"
    }

    stages {
        stage('Initialize') {
            steps {
                sh """
                    find $WORKSPACE -mindepth 1 -maxdepth 1 | xargs rm -rf
                """
                sh """
                    cp -f "${NETRC_FILE}" $HOME/.netrc
                    chmod 600 $HOME/.netrc
                """
            }
        }

        stage('Default checkout') {
            steps {
                script {
                    def scmURL = scm.getUserRemoteConfigs()[0].getUrl()
                    defaultCheckoutTargetDir = scmURL.replaceAll(/^.*\//,'').replaceAll(/\.git$/, '')
                }
                checkout([
                    $class: 'GitSCM',
                    branches: scm.branches,
                    extensions: scm.extensions + [[$class: 'RelativeTargetDirectory', relativeTargetDir: defaultCheckoutTargetDir]],
                    userRemoteConfigs: scm.userRemoteConfigs
                ])
            }
        }

        stage('Checkout source code') {
            when { expression { return fileExists("${WORKSPACE}/examples") == false } }
            steps {
                checkout poll: false, scm: [
                        $class                           : 'GitSCM',
                        branches                         : [[name: params.VERRAZZANO_BRANCH]],
                        browser: [$class: 'GithubWeb', repoUrl: 'https://github.com/verrazzano/examples'],
                        doGenerateSubmoduleConfigurations: false,
                        extensions                       : [
                                [$class: 'RelativeTargetDirectory', relativeTargetDir: 'examples'],
                                [$class: 'CleanBeforeCheckout'],
                        ],
                        submoduleCfg                     : [],
                        userRemoteConfigs                : [
                                [credentialsId: 'github-markxnelns-private-access-token', url: 'https://github.com/verrazzano/examples.git']
                        ],
                ]
            }
        }


        stage('Prepare Environment') {
            steps {
                getMavenSeedData '/build-shared-files'
                sh """
                    mkdir -p $HOME/.m2/repository/com
                    tar xz -C $HOME/.m2/repository/com -f /build-shared-files/oracle-maven.tar.gz
                    sudo yum -y install wget
                """
            }
        }

        stage('Copyright Compliance Check') {
            when { not { buildingTag() } }
            steps {
                copyrightScan "${WORKSPACE}/examples"
            }
        }

        stage('Parallel builds') {
            parallel {
                stage ('Bobbys Books') {
                    stages {
                        stage('Build Bobbys Coherence Application') {
                            steps {
                                sh """
                                    echo "${DOCKER_CREDS_PSW}" | docker login docker.pkg.github.com -u ${DOCKER_CREDS_USR} --password-stdin
                                    cd examples/bobs-books/bobbys-books/bobbys-coherence
                                    mvn -B -s $MAVEN_SETTINGS clean deploy
                                    oci os object get -bn ${BUCKET_NAME} --file ${GRAALVM_BUNDLE} --name ${GRAALVM_BUNDLE}
                                    docker build --build-arg GRAALVM_BINARY=${GRAALVM_BUNDLE} --force-rm=true -f Dockerfile -t ${env.REPO}/${env.BOBBYS_COHERENCE}:${env.VERSION} .
                                    docker push ${env.REPO}/${env.BOBBYS_COHERENCE}:${env.VERSION}
                                """
                            }
                        }

                        stage('Scan Bobbys Coherence Application') {
                            steps {
                                script {
                                    clairScanTemp "${env.REPO}/${env.BOBBYS_COHERENCE}:${env.VERSION}"
                                }
                                sh "mv scanning-report.json bobby_coherence.scanning-report.json"
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: '**/*scanning-report.json', allowEmptyArchive: true
                                }
                            }
                        }

                        stage('Build Bobbys Helidon Stock Application') {
                            steps {
                                sh """
                                    echo "${DOCKER_CREDS_PSW}" | docker login docker.pkg.github.com -u ${DOCKER_CREDS_USR} --password-stdin
                                    cd examples/bobs-books/bobbys-books/bobbys-helidon-stock-application
                                    mvn -B -s $MAVEN_SETTINGS clean deploy 
                                    oci os object get -bn ${BUCKET_NAME} --file ${GRAALVM_BUNDLE} --name ${GRAALVM_BUNDLE}
                                    docker build --build-arg GRAALVM_BINARY=${GRAALVM_BUNDLE} --force-rm=true -f Dockerfile -t ${env.REPO}/${env.BOBBYS_HELIDON}:${env.VERSION} .
                                    docker push ${env.REPO}/${env.BOBBYS_HELIDON}:${env.VERSION}
                                """
                            }
                        }

                        stage('Scan Bobbys Helidon Stock Application') {
                            steps {
                                script {
                                    clairScanTemp "${env.REPO}/${env.BOBBYS_HELIDON}:${env.VERSION}"
                                }
                                sh "mv scanning-report.json bobby_helidon.scanning-report.json"
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: '**/*scanning-report.json', allowEmptyArchive: true
                                }
                            }
                        }

                        stage('Build Bobbys Front-end WebLogic Application') {
                            steps {
                                sh """
                                    echo "${DOCKER_CREDS_PSW}" | docker login docker.pkg.github.com -u ${DOCKER_CREDS_USR} --password-stdin
                                    echo "${OCR_CREDS_PSW}" | docker login container-registry.oracle.com -u ${OCR_CREDS_USR} --password-stdin
                                    cd examples/bobs-books/bobbys-books/bobbys-front-end
                                    mvn -B -s $MAVEN_SETTINGS clean deploy
                                    cd deploy
                                    oci os object get -bn ${BUCKET_NAME} --file ${GRAALVM_JDK8_BUNDLE} --name ${GRAALVM_JDK8_BUNDLE}
                                    oci os object get -bn ${BUCKET_NAME} --file ${WEBLOGIC_BUNDLE} --name ${WEBLOGIC_BUNDLE}
                                    oci os object get -bn ${BUCKET_NAME} --file ${IMAGETOOL_BUNDLE} --name ${IMAGETOOL_BUNDLE}
                                    ./build.sh ${env.REPO}/${env.BOBBYS_WEBLOGIC}:${env.VERSION}
                                    docker push ${env.REPO}/${env.BOBBYS_WEBLOGIC}:${env.VERSION}
                                """
                            }
                        }

                        stage('Scan Bobbys Front-end WebLogic Application') {
                            steps {
                                script {
                                    clairScanTemp "${env.REPO}/${env.BOBBYS_WEBLOGIC}:${env.VERSION}"
                                }
                                sh "mv scanning-report.json bobby_weblogic.scanning-report.json"
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: '**/*scanning-report.json', allowEmptyArchive: true
                                }
                            }
                        }
                    }
                }

                stage('Bobs Backend') {
                    stages {
                        stage('Build Bobs Backend WebLogic Application') {
                            steps {
                                sh """
                                    echo "${DOCKER_CREDS_PSW}" | docker login docker.pkg.github.com -u ${DOCKER_CREDS_USR} --password-stdin
                                    echo "${OCR_CREDS_PSW}" | docker login container-registry.oracle.com -u ${OCR_CREDS_USR} --password-stdin
                                    cd examples/bobs-books/bobs-bookstore-order-manager
                                    mvn -B -s $MAVEN_SETTINGS clean deploy
                                    cd deploy
                                    echo 'Update passwords from Jenkins secrets'
                                    sed -i -e "s|XX_DB_PASSWORD_XX|${env.BOB_DB_PASSWORD}|g" properties/docker-build/bobs-bookstore-topology.properties.encoded
                                    sed -i -e "s|XX_ADMIN_PASSWORD_XX|${env.BOB_ADMIN_PASSWORD}|g" properties/docker-build/bobs-bookstore-topology.properties.encoded
                                    ./build.sh ${env.REPO}/${env.BOBS_WEBLOGIC}:${env.VERSION}
                                    docker push ${env.REPO}/${env.BOBS_WEBLOGIC}:${env.VERSION}
                                """
                            }
                        }

                        stage('Scan Bobs Backend WebLogic Application') {
                            steps {
                                script {
                                    clairScanTemp "${env.REPO}/${env.BOBS_WEBLOGIC}:${env.VERSION}"
                                }
                                sh "mv scanning-report.json bobs_weblogic.scanning-report.json"
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: '**/*scanning-report.json', allowEmptyArchive: true
                                }
                            }
                        }
                    }
                }

                stage('Roberts Books') {
                    stages {
                        stage('Build Roberts Coherence Application') {
                            steps {
                                sh """
                                    echo "${DOCKER_CREDS_PSW}" | docker login docker.pkg.github.com -u ${DOCKER_CREDS_USR} --password-stdin
                                    cd examples/bobs-books/roberts-books/roberts-coherence
                                    mvn -B -s $MAVEN_SETTINGS clean deploy
                                    oci os object get -bn ${BUCKET_NAME} --file ${GRAALVM_BUNDLE} --name ${GRAALVM_BUNDLE}
                                    docker build --build-arg GRAALVM_BINARY=${GRAALVM_BUNDLE} --force-rm=true -f Dockerfile -t ${env.REPO}/${env.ROBERTS_COHERENCE}:${env.VERSION} .
                                    docker push ${env.REPO}/${env.ROBERTS_COHERENCE}:${env.VERSION}
                                """
                            }
                        }

                        stage('Scan Roberts Coherence Application') {
                            steps {
                                script {
                                    sh "sleep 300 # workaround clair concurrency issue"
                                    clairScanTemp "${env.REPO}/${env.ROBERTS_COHERENCE}:${env.VERSION}"
                                }
                                sh "mv scanning-report.json roberts_coherence.scanning-report.json"
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: '**/*scanning-report.json', allowEmptyArchive: true
                                }
                            }
                        }

                        stage('Build Roberts Helidon Stock Application') {
                            steps {
                                sh """
                                    echo "${DOCKER_CREDS_PSW}" | docker login docker.pkg.github.com -u ${DOCKER_CREDS_USR} --password-stdin
                                    cd examples/bobs-books/roberts-books/roberts-helidon-stock-application/src/main/web
                                    npm install
                                    cd ../../..
                                    mvn -B -s $MAVEN_SETTINGS clean deploy
                                    oci os object get -bn ${BUCKET_NAME} --file ${GRAALVM_BUNDLE} --name ${GRAALVM_BUNDLE}
                                    docker build --build-arg GRAALVM_BINARY=${GRAALVM_BUNDLE} --force-rm=true -f Dockerfile -t ${env.REPO}/${env.ROBERTS_HELIDON}:${env.VERSION} .
                                    docker push ${env.REPO}/${env.ROBERTS_HELIDON}:${env.VERSION}
                                """
                            }
                        }

                        stage('Scan Roberts Helidon Stock Application') {
                            steps {
                                script {
                                    clairScanTemp "${env.REPO}/${env.ROBERTS_HELIDON}:${env.VERSION}"
                                }
                                sh "mv scanning-report.json roberts_helidon.scanning-report.json"
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: '**/*scanning-report.json', allowEmptyArchive: true
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

