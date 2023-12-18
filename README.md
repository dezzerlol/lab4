1. start jenkins on server
```
https://www.digitalocean.com/community/tutorials/how-to-install-jenkins-on-ubuntu-20-04
```

2. jenkins admin password
```
docker exec -i jenkins-ci cat /var/jenkins_home/secrets/initialAdminPassword
```

3. run nexus repository. wait for container to start (1 min)
```
docker run -d --name nexus_repo -p 8030:8081 sonatype/nexus3
```

4. nexus admin password
```
docker exec -i nexus_repo cat /nexus-data/admin.password
```

5. create maven nexus repo
```
type: maven2 (hosted)
version policy: mixed
deployment policy: allow redeploy
```

6. create maven user for nexus
```
id: jenkins-user
firstname: Jenkins
lastname: User
email: test@mail.com
password: jenkins
status: active
roles: nx-admin
```

7. install jenkins plugins
```
Nexus Artifact Uploader
Pipeline Utility Steps
```

8. install maven in jenkins

9. add nexus credentials to jenkins

10. create nexus pipeline
```
pipeline {
    agent {
        label "master"
    }
    tools {
        maven 'maven'
    }
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.16.26.131:8030"
        NEXUS_MVN_REPOSITORY = "maven-nexus-repo"
        NEXUS_DOCKER_REPO = "docker-nexus-repo"
        NEXUS_CREDENTIAL_ID = "nexus-user-credentials"
    }
    stages {
        stage("Clone code from VCS") {
            steps {
                script {
                    git 'https://github.com/dezzerlol/CreatePDF';
                }
            }
        }
        stage("Maven Build") {
            steps {
                script {
                    sh "mvn package -DskipTests=true"
                }
            }
        }
        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_MVN_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }

        stage('Docker Build') {
        
            steps { 
                    echo 'Building docker Image'
                    sh 'docker build -t createpdf:latest .'
                }
        }

       stage('Docker Login') {
            steps {
                echo 'Nexus Docker Repository Login'
                script{
                    withCredentials([usernamePassword(credentialsId: NEXUS_CREDENTIAL_ID, usernameVariable: 'USER', passwordVariable: 'PASS' )]){
                       sh 'echo $PASS | docker login -u $USER --password-stdin $NEXUS_DOCKER_REPO'
                    }
                   
                }
            }
        }

        stage('Docker Push') {
            steps {
                echo 'Pushing Imgaet to docker hub'
                sh 'docker push createpdf:latest'
            }
        }
    }
}
```


999. add jenkins to docker group:
```
https://stackoverflow.com/questions/47854463/docker-got-permission-denied-while-trying-to-connect-to-the-docker-daemon-socke
```

docker run -d --name jenkins-ci -p 8040:8080 -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins:lts