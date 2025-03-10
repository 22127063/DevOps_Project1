pipeline {
    agent jenkins-node-ssh1

    // Get the list of changed files in the last commit
    environment {
        CHANGED_FILES = sh(script: "git diff --name-only $(git rev-parse HEAD~1 || echo HEAD)", returnStdout: true).trim()
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    // Define the microservices that should be checked
                    def services = ['customers-service', 'vets-service', 'visits-service']
                    def triggeredServices = []

                    // Check if any files were modified in the respective service directories
                    for (service in services) {
                        if (CHANGED_FILES.contains(service + "/")) {
                            triggeredServices.add(service)
                        }
                    }

                    // If no relevant files were changed, skip the build
                    if (triggeredServices.isEmpty()) {
                        echo "No relevant changes detected. Skipping build."
                        currentBuild.result = 'ABORTED'
                        error("No changes detected")
                    } else {
                        env.TRIGGERED_SERVICES = triggeredServices.join(',')
                    }
                }
            }
        }

        stage('Test & Collect Results') {
            steps {
                script {
                    // Run tests for each service that has changed
                    def services = env.TRIGGERED_SERVICES.split(',')
                    for (service in services) {
                        dir(service) {
                            sh './mvnw test' // Execute unit tests
                        }
                    }
                }
                junit '**/target/surefire-reports/*.xml' // Upload test results to Jenkins
            }
        }

        stage('Check Coverage') {
            steps {
                script {
                    def services = env.TRIGGERED_SERVICES.split(',')
                    for (service in services) {
                        dir(service) {
                            sh './mvnw verify'  // Run tests + generate code coverage report

                            // Extract test coverage percentage from JaCoCo report
                            def coverage = sh(script: "grep -oP '(?<=<line-rate>).*?(?=</line-rate>)' target/site/jacoco/jacoco.xml | awk '{s+=$1} END {print s/NR}'", returnStdout: true).trim().toFloat()

                            // If test coverage is below 70%, fail the pipeline
                            if (coverage < 0.7) {
                                error("Test coverage too low: ${coverage * 100}%")
                            }
                        }
                    }
                }
                publishCoverage adapters: [jacocoAdapter('**/target/site/jacoco/jacoco.xml')] // Upload coverage reports
            }
        }

        stage('Build') {
            steps {
                script {
                    def services = env.TRIGGERED_SERVICES.split(',')
                    def parallelJobs = [:]

                    // Package only the changed services in parallel for faster execution
                    for (service in services) {
                        parallelJobs[service] = {
                            dir(service) {
                                sh './mvnw package' // Build the service
                            }
                        }
                    }

                    parallel parallelJobs // Run the build tasks in parallel
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.' // Always print completion message, even if aborted
        }
    }
}
