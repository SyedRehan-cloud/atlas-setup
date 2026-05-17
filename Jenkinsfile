pipeline {
    agent any

    environment {
        DB_HOST = "localhost"
        DB_PORT = "5432"
        DB_NAME = "appdb"
        DB_USER = "admin"
        DB_PASS = "admin"

        REMOTE_HOST = "172.31.16.197"
        WORKDIR = "/home/ubuntu/liquibase-enterprise-poc"
    }

    stages {

        stage('Run Liquibase on Server') {
            steps {
                sshagent(['liquibase-ci-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${REMOTE_HOST} '

                        set -e

                        echo "Moving to Liquibase directory..."
                        cd ${WORKDIR}

                        echo "Checking Liquibase version..."
                        ./liquibase --version

                        echo "DB Connectivity Check..."
                        PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c "\\dt"

                        echo "Validate..."
                        ./liquibase validate --defaultsFile=liquibase.properties

                        echo "Check Locks..."
                        ./liquibase list-locks --defaultsFile=liquibase.properties || true

                        echo "Release Locks..."
                        ./liquibase release-locks --defaultsFile=liquibase.properties || true

                        echo "Deploy Schema..."
                        ./liquibase update --defaultsFile=liquibase.properties

                        echo "Verify Schema..."
                        PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c "\\d users"

                        echo "History..."
                        ./liquibase history --defaultsFile=liquibase.properties

                        echo "DONE"
                    '
                    """
                }
            }
        }
    }

    post {

        success {
            echo "SUCCESS: Liquibase deployment completed"

            sshagent(['liquibase-ci-key']) {
                sh """
                ssh -o StrictHostKeyChecking=no ubuntu@${REMOTE_HOST} '
                PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c "SELECT count(*) FROM databasechangelog;"
                '
                """
            }
        }

        failure {
            echo "FAILED: Debug mode"

            sshagent(['liquibase-ci-key']) {
                sh """
                ssh -o StrictHostKeyChecking=no ubuntu@${REMOTE_HOST} '
                cd ${WORKDIR}
                ./liquibase status --defaultsFile=liquibase.properties || true
                ./liquibase list-locks --defaultsFile=liquibase.properties || true
                PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c "\\dt" || true
                '
                """
            }
        }

        always {
            echo "Pipeline finished"
        }
    }
}
