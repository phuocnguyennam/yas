pipeline {
    agent none
    triggers {
        githubPush()
    }
    environment {
        MAVEN_OPTS   = '-Xmx1536m -XX:+TieredCompilation -XX:TieredStopAtLevel=1'
        // Maven image — swap tag here to upgrade globally
        MAVEN_IMAGE  = 'maven:3.9-eclipse-temurin-25'
        // Local Maven repo cached on the host to avoid re-downloading deps
        MAVEN_CACHE  = '/tmp/jenkins-maven-cache'
        REVISION     = '1.0-SNAPSHOT'
        ALL_MODULES  = [
            'common-library',
            'backoffice-bff',
            'cart',
            'customer',
            'inventory',
            'location',
            'media',
            'order',
            'payment-paypal',
            'payment',
            'product',
            'promotion',
            'rating',
            'search',
            'storefront-bff',
            'tax',
            'webhook',
            'sampledata',
            'recommendation',
            'delivery'
        ].join(',')
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 1, unit: 'HOURS')
    }
    stages {
        stage('Checkout') {
            agent any
            steps {
                checkout scm
                stash name: 'source-code', includes: '**/*', excludes: '.git/**'
            }
        }
        stage('Change Detection') {
            agent any
            steps {
                script {
                    echo "Check changes in modules"
                    def basebranch = env.CHANGE_TARGET ?: env.BRANCH_NAME
                    def changedFiles = sh(
                        script: "git diff --name-only origin/${basebranch}...HEAD || true",
                        returnStdout: true
                    ).trim().split('\n')
                    def changedModules = []
                    boolean ForceBuildAll = false
                    for (file in changedFiles) {
                        if (file == 'pom.xml' || file.startsWith('docker-compose') || file.startsWith('Jenkinsfile') || file.startsWith('common-library/')) {
                            ForceBuildAll = true
                            break
                        }
                        def allModules = env.ALL_MODULES.split(',')
                        for (module in allModules) {
                            if (file.startsWith(module + '/')) {
                                changedModules.add(module)
                                break
                            }
                        }
                    }
                    if (env.BRANCH_NAME == 'main') {
                        ForceBuildAll = true
                    }
                    if (ForceBuildAll || changedModules.isEmpty()) {
                        env.MODULES_TO_BUILD = env.ALL_MODULES
                        echo "Changes detected in critical files or no module-specific changes. Building all modules: ${env.MODULES_TO_BUILD}"
                    } else {
                        env.MODULES_TO_BUILD = changedModules.unique().join(',')
                        echo "Modules with changes detected: ${env.MODULES_TO_BUILD}"
                    }
                    if (env.MODULES_TO_BUILD && !env.MODULES_TO_BUILD.contains('common-library')) {
                        env.MODULES_TO_BUILD = "common-library,${env.MODULES_TO_BUILD}"
                    }
                }
            }
        }
        stage('Build') {
            agent {
                docker {
                    image "${env.MAVEN_IMAGE}"
                    args "-v ${env.MAVEN_CACHE}:/root/.m2/repository"
                    reuseNode true
                }
            }
            steps {
                unstash 'source-code'
                script {
                    def modules = env.MODULES_TO_BUILD.split(',').toList()
                    if (modules.contains('common-library')) {
                        echo "Building common-library first"
                        sh """
                            mvn -pl common-library -am\
                                install -DskipTests -B -V \
                                --no-transfer-progress\
                                -T 1\
                                -Drevision=${env.REVISION}
                        """
                        modules.remove('common-library')
                    }
                    if (modules.contains('payment') && modules.contains('payment-paypal')) {
                        // Build payment-paypal trước, rồi mới để payment build song song với các module khác
                        sh """
                            mvn -pl payment-paypal -am\
                                install -DskipTests -B -V \
                                --no-transfer-progress \
                                -T 1 \
                                -Drevision=${env.REVISION}
                        """
                        modules.remove('payment-paypal')
                    }
                    def BuildTask = [:]
                    modules.each { module -> 
                        def mod = module
                        BuildTask["Build: ${mod}"] = {
                            sh """
                                mvn -pl ${mod}\
                                    package -DskipTests -B -V \
                                    --no-transfer-progress\
                                    -T 1\
                                    -Drevision=${env.REVISION}
                            """
                            stash name: "build-${mod}", includes: "${mod}/target/**"
                        }
                    }
                    if (BuildTask.size() > 0) {
                        parallel BuildTask
                    } else {
                        echo "No modules to build"
                    }
                }

            }
        }
        stage('Test') {
            agent {
                docker {
                    image "${env.MAVEN_IMAGE}"
                    args "-v ${env.MAVEN_CACHE}:/root/.m2/repository"
                    reuseNode true
                }
            }
            steps {
                unstash 'source-code'
                script {
                    def modules = env.MODULES_TO_BUILD.split(',')
                    def TestTask = [:]
                    modules.each { module -> 
                        def mod = module
                        TestTask["Test: ${mod}"] = {
                            try {
                                unstash "build-${mod}"
                            } catch (e) {
                                echo "No build artifacts found for ${mod}, skipping tests"
                                return
                            }
                            sh """
                                mvn -pl ${mod}\
                                    verify -B -V \
                                    --no-transfer-progress\
                                    -T 1\
                                    -Drevision=${env.REVISION}
                            """
                            def checkCoverageScript = """
                                REPORT_FILE="${mod}/target/site/jacoco/jacoco.csv"
                                
                                if [ -f "\$REPORT_FILE" ]; then
                                    # Cột 4: INSTRUCTION_MISSED, Cột 5: INSTRUCTION_COVERED
                                    COVERAGE=\$(awk -F"," 'NR>1 { missed += \$4; covered += \$5 } END { if ((missed+covered) > 0) print int((covered/(missed+covered))*100); else print 0 }' "\$REPORT_FILE")
                                    
                                    echo "Code Coverage for module '\${mod}': \${COVERAGE}%"
                                    
                                    if [ "\$COVERAGE" -lt 70 ]; then
                                        echo "FAILED: Coverage (\${COVERAGE}%) is below the minimum threshold of 70%!"
                                        exit 1
                                    else
                                        echo "PASSED: Coverage meets the requirement."
                                    fi
                                else
                                    echo "WARNING: No Jacoco coverage report found at \$REPORT_FILE."
                                    echo "Did the tests run for ${mod}? Assuming 0% coverage."
                                    exit 1
                                fi
                            """
                            
                            sh script: checkCoverageScript
                        }
                    }
                    if (TestTask.size() > 0) {
                        parallel TestTask
                    } else {
                        echo "No modules to test"
                    }
                }
            }
        }
    }
}