pipeline {
    agent any

    environment {
        DB_HOST = "localhost"
        DB_PORT = "5432"
        DB_NAME = "appdb"
        DB_USER = "admin"
        DB_PASS = "admin"

        LIQUIBASE_VERSION = "4.27.0"
        WORKDIR = "${WORKSPACE}/liquibase-runtime"
    }

    stages {

        stage('Prepare Workspace') {
            steps {
                sh """
                rm -rf ${WORKDIR}
                mkdir -p ${WORKDIR}
                """
            }
        }

        stage('Download Liquibase at Runtime') {
            steps {
                sh """
                cd ${WORKDIR}

                echo "Downloading Liquibase..."
                wget -q https://github.com/liquibase/liquibase/releases/download/v${LIQUIBASE_VERSION}/liquibase-${LIQUIBASE_VERSION}.tar.gz

                echo "Extracting..."
                tar -xzf liquibase-${LIQUIBASE_VERSION}.tar.gz

                chmod +x liquibase
                ./liquibase --version
                """
            }
        }

        stage('Workspace Check') {
            steps {
                sh "ls -R"
            }
        }

        stage('DB Connectivity Check') {
            steps {
                sh """
                PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c '\\dt'
                """
            }
        }

        stage('Liquibase Validate') {
            steps {
                sh """
                ${WORKDIR}/liquibase validate --defaultsFile=liquibase.properties
                """
            }
        }

        stage('Check DB Lock') {
            steps {
                sh """
                ${WORKDIR}/liquibase list-locks --defaultsFile=liquibase.properties || true
                """
            }
        }

        stage('Release Lock') {
            steps {
                sh """
                ${WORKDIR}/liquibase release-locks --defaultsFile=liquibase.properties || true
                """
            }
        }

        stage('Deploy Schema') {
            steps {
                sh """
                ${WORKDIR}/liquibase update --defaultsFile=liquibase.properties
                """
            }
        }

        stage('Verify Schema') {
            steps {
                sh """
                PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c '\\d users'
                """
            }
        }

        stage('Audit Check') {
            steps {
                sh """
                ${WORKDIR}/liquibase history --defaultsFile=liquibase.properties
                """
            }
        }

        stage('Rollback (Disabled)') {
            when {
                expression { return false }
            }
            steps {
                sh """
                ${WORKDIR}/liquibase rollbackCount 1 --defaultsFile=liquibase.properties
                """
            }
        }
    }

    post {

        success {
            echo "SUCCESS: Runtime Liquibase execution completed"

            sh """
            PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c 'SELECT count(*) FROM databasechangelog;'
            """
        }

        failure {
            echo "FAILED: Debug mode"

            sh """
            ${WORKDIR}/liquibase status --defaultsFile=liquibase.properties || true
            ${WORKDIR}/liquibase list-locks --defaultsFile=liquibase.properties || true
            """

            sh """
            PGPASSWORD=${DB_PASS} psql -h ${DB_HOST} -U ${DB_USER} -d ${DB_NAME} -c '\\dt' || true
            """
        }

        always {
            echo "Pipeline finished (runtime mode)"
        }
    }
}
