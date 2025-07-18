// Jenkinsfile for an Ansible Role Project
// This pipeline automates linting, syntax checking, and Molecule testing
// for an Ansible role.

// Ensure you have the necessary Jenkins plugins installed:
// - Git Plugin
// - Pipeline Plugin
// - Pipeline Utility Steps Plugin (for reading properties if needed)

// The entire pipeline definition must be enclosed within a 'pipeline' block.
pipeline {
    // Define agent for the pipeline.
    // 'any' means Jenkins will pick any available agent.
    // For production, you might want to specify a label for a specific agent
    // with Ansible, Python, Molecule, and ansible-lint installed.
    agent any

    // Define global environment variables for the pipeline
    environment {
        // Path to the Ansible role project root within the workspace
        // Assuming the Jenkinsfile is at the root of the Ansible role repo
        ANSIBLE_ROLE_PATH = '.'
        // Define Python virtual environment path if you're using one
        VENV_PATH = "/opt/ansible/venv"
    }

    // Define stages for the pipeline
    stages {
        // Stage 0: Explicit Git Checkout and Verification
        // This stage ensures the repository is checked out and provides Git information.
        stage('Git Checkout & Verify') {
            steps {
                script {
                    // Explicitly checkout the SCM. This often provides more verbose output.
                    // 'scm' refers to the SCM configuration of the job itself.
                    checkout scm

                    echo "--- Git Information ---"
                    // Print the Git commit ID that was checked out
                    sh 'echo "Current Git Commit ID: $(git rev-parse HEAD)"'
                    sh 'echo "Short Git Commit ID: $(git rev-parse --short HEAD)"'
                    sh 'echo "Git Branch: $(git rev-parse --abbrev-ref HEAD)"'
                    sh 'echo "Git Remote URL: $(git config --get remote.origin.url)"'
                    echo "--- End Git Information ---"

                    // Jenkins also sets environment variables for Git
                    echo "Jenkins GIT_COMMIT env var: ${env.GIT_COMMIT}"
                    echo "Jenkins GIT_BRANCH env var: ${env.GIT_BRANCH}"
                    // Note: GIT_URL is also often available
                    echo "Jenkins GIT_URL env var: ${env.GIT_URL}"
                }
            }
        }

        // Stage 1: Prepare Environment
        // Sets up a Python virtual environment and installs necessary tools.
        stage('Prepare Environment') {
            steps {
                script {
                    // Check if Python is available
                    sh 'python3 --version || python --version'
                    sh 'pip --version' // Check if pip is available

                    // Create and activate a Python virtual environment
                    // This isolates dependencies for the pipeline run.
                    echo "Setting up Python virtual environment at ${VENV_PATH}"
                    sh "python3 -m venv ${VENV_PATH}"
                    sh "source ${VENV_PATH}/bin/activate"

                    // Install ansible, ansible-lint, and molecule
                    // Ensure correct versions are specified if needed.
                    echo "Installing Python dependencies..."
                    sh "${VENV_PATH}/bin/pip install --upgrade pip"
                    sh "${VENV_PATH}/bin/pip install ansible ansible-lint molecule[docker] testinfra"

                    // If your role has dependencies from Ansible Galaxy, install them
                    // This step is crucial if your role uses other roles.
                    if (fileExists("${ANSIBLE_ROLE_PATH}/requirements.yml")) {
                        echo "Installing Ansible Galaxy role dependencies from requirements.yml..."
                        sh "${VENV_PATH}/bin/ansible-galaxy install -r ${ANSIBLE_ROLE_PATH}/requirements.yml"
                    } else {
                        echo "No requirements.yml found. Skipping Ansible Galaxy dependencies installation."
                    }
                }
            }
        }

        // Stage 2: Linting
        // Uses ansible-lint to check for common issues and best practices.
        stage('Linting') {
            steps {
                script {
                    echo "Debugging Linting Stage:"
                    // Print the current working directory
                    sh 'pwd'
                    // List contents of the current directory to verify file presence
                    sh 'ls -la'
                    echo "Value of ANSIBLE_ROLE_PATH: ${ANSIBLE_ROLE_PATH}"
                    echo "Attempting to run ansible-lint..."
                    // Use 'set -x' to print the exact command being executed by the shell
                    // This will help identify if the command itself is malformed.
                    sh '''
                        set -x
                        ${VENV_PATH}/bin/ansible-lint ${ANSIBLE_ROLE_PATH}
                    '''
                }
            }
        }

        // Stage 3: Syntax Check
        // Performs a syntax check of the role's main playbook.
        stage('Syntax Check') {
            steps {
                script {
                    echo "Performing Ansible syntax check..."
                    // Use the test.yml playbook in the tests directory for a quick syntax check.
                    // --syntax-check only parses the playbook and role, it doesn't execute anything.
                    // -i tests/inventory ensures a valid inventory is provided.
                    sh "${VENV_PATH}/bin/ansible-playbook ${ANSIBLE_ROLE_PATH}/tests/test.yml --syntax-check -i ${ANSIBLE_ROLE_PATH}/tests/inventory"
                }
            }
        }

        // Stage 4: Molecule Test
        // Executes Molecule tests for the role. This is the most comprehensive testing step.
        stage('Molecule Test') {
            steps {
                script {
                    echo "Running Molecule tests..."
                    // Navigate into the Ansible role directory where molecule.yml is located.
                    dir("${ANSIBLE_ROLE_PATH}") {
                        // Run the default Molecule scenario.
                        // 'molecule test' command performs create, converge, verify, and destroy.
                        // Ensure Docker daemon is running on the Jenkins agent if using Molecule with Docker driver.
                        sh "${VENV_PATH}/bin/molecule test"
                    }
                }
            }
        }

        // Stage 5: Cleanup (Optional but Recommended)
        // Destroys any remaining Molecule instances.
        // This is useful if a previous Molecule run failed before destruction.
        stage('Cleanup') {
            steps {
                script {
                    echo "Cleaning up Molecule instances..."
                    dir("${ANSIBLE_ROLE_PATH}") {
                        // Destroy all Molecule instances.
                        sh "${VENV_PATH}/bin/molecule destroy"
                    }
                    //echo "Removing Python virtual environment..."
                    //sh "rm -rf ${VENV_PATH}"
                }
            }
        }
    }

    // Post-build actions (executed after all stages, regardless of success/failure)
    post {
        // Always clean up the workspace to ensure a fresh start for the next build.
        always {
            echo "Cleaning up workspace..."
            deleteDir()
        }
        // Actions to perform only if the build succeeds
        success {
            echo 'Pipeline finished successfully!'
            // You could add notifications here, e.g., Slack, Email
        }
        // Actions to perform only if the build fails
        failure {
            echo 'Pipeline failed!'
            // You could add notifications here for failures
        }
    }
}
