pipeline {
    agent any

    stages {
        stage('Install Maven') {
            steps {
                script {
                    // Vérifie si Maven est installé, sinon l'installe
                    sh """
                    if ! command -v mvn &> /dev/null
                    then
                        echo "Maven n'est pas installé. Installation de Maven..."
                        curl -sL https://apache.claz.org/maven/maven-3/3.8.6/binaries/apache-maven-3.8.6-bin.tar.gz -o /tmp/maven.tar.gz
                        tar -xzvf /tmp/maven.tar.gz -C /opt
                        echo "export M2_HOME=/opt/apache-maven-3.8.6" >> ~/.bashrc
                        echo "export PATH=\$M2_HOME/bin:\$PATH" >> ~/.bashrc
                        source ~/.bashrc
                    else
                        echo "Maven est déjà installé."
                    fi
                    """
                }
            }
        }

        stage('Build App') {
            steps {
                git branch: 'main', url: 'https://github.com/kuldeepsingh99/openshift-jenkins-cicd.git'
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    version = pom.version
                }
                sh "mvn install"
            }
        }

        stage('Create Image Builder') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject() {
                            return !openshift.selector("bc", "sample-app-jenkins-new").exists();
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.newBuild("--name=sample-app-jenkins-new", "--image-stream=openjdk18-openshift:1.14-3", "--binary=true")
                        }
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh "rm -rf ocp && mkdir -p ocp/deployments"
                sh "pwd && ls -la target"
                sh "cp target/openshiftjenkins-0.0.1-SNAPSHOT.jar ocp/deployments"

                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            openshift.selector("bc", "sample-app-jenkins-new").startBuild("--from-dir=./ocp", "--follow", "--wait=true")
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject() {
                            return !openshift.selector('dc', 'sample-app-jenkins-new').exists()
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject() {
                            def app = openshift.newApp("sample-app-jenkins-new", "--as-deployment-config")
                            app.narrow("svc").expose();
                        }
                    }
                }
            }
        }
    }
}
