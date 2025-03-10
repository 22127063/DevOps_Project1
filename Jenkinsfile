pipeline {
    agent jenkins-node-ssh1
    environment {
        CHANGED_FILES = sh(script: "git diff --name-only $(git rev-parse HEAD~1 || echo HEAD)", returnStdout: true).trim()
    }
    stages {
        stage('Detect Changes') {
            steps {
                script {
                    def services = ['customers-service', 'vets-service', 'visits-service']
                    def triggeredServices = []

                    for (service in services) {
                        if (CHANGED_FILES.contains(service + "/")) {
                            triggeredServices.add(service)
                        }
                    }

                    if (triggeredServices.isEmpty()) {
                        error("No relevant changes detected. Skipping build.")
                    } else {
                        env.TRIGGERED_SERVICES = triggeredServices.join(',')
                    }
                }
            }
        }

        stage('Check Coverage') {
            steps {
                script {
                    def services = env.TRIGGERED_SERVICES.split(',')
                    for (service in services) {
                        dir(service) {
                            sh './mvnw verify'
                            def coverage = sh(script: "grep -oP '(?<=<line-rate>).*?(?=</line-rate>)' target/site/jacoco/jacoco.xml | awk '{s+=$1} END {print s/NR}'", returnStdout: true).trim().toFloat()

                            if (coverage < 0.7) {
                                error("Test coverage too low: ${coverage * 100}%")
                            }
                        }
                    }
                }
            }
        }

        stage('Test & Build') {
            steps {
                script {
                    def services = env.TRIGGERED_SERVICES.split(',')
                    def parallelJobs = [:]

                    for (service in services) {
                        parallelJobs[service] = {
                            dir(service) {
                                sh './mvnw test'
                                sh './mvnw package'
                            }
                        }
                    }

                    parallel parallelJobs
                }
            }
        }
    }
}
