1. start jenkins on server
```
https://www.digitalocean.com/community/tutorials/how-to-install-jenkins-on-ubuntu-20-04
```

2. jenkins admin password
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

3. get ip from `ip addr`
```
172.16.26.131 
```

4. run nexus repository. wait for container to start (1 min)
```
docker run -d --name nexus_repo -p 8030:8081 -p 8031:8082 -e REGISTRY_HTTP_HOST=https://172.16.26.131:8082 sonatype/nexus3
```

5. get nexus admin password
```
docker exec -i nexus_repo cat /nexus-data/admin.password
```

6. create maven repo inside nexus
```
type: maven2 (hosted)
version policy: mixed
deployment policy: allow redeploy
```

7. create docker repo inside nexus
```
name: docker-nexus-repo
type: hosted
http: 8082
```

8. create maven user for nexus
```
id: jenkins-user
firstname: Jenkins
lastname: User
email: test@mail.com
password: jenkins
status: active
roles: nx-admin
```

9. install jenkins plugins
```
Nexus Artifact Uploader
Pipeline Utility Steps
docker
```

10.  install maven in jenkins

11.   add nexus credentials to jenkins
    
12.   cat /etc/default/docker
```
DOCKER_OPTS="--config-file=/etc/docker/daemon.json"
```

13.   cat /etc/docker/daemon.json
```
{
"insecure-registries": ["172.16.26.131:8030", "172.16.26.131:8031", "172.16.26.131:8082"],
"registry-mirrors": ["http://172.16.26.131:8030"]
}
```

14. create nexus pipeline
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
        NEXUS_MVN_URL = "172.16.26.131:8030"
        NEXUS_MVN_REPOSITORY = "maven-nexus-repo"
        NEXUS_DOCKER_URL = "172.16.26.131:8031"
        NEXUS_DOCKER_REPO = "docker-nexus-repo"
        NEXUS_CREDENTIAL_ID = "nexus-user-credentials"
        CONTAINER_NAME = "java-app"
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
                            nexusUrl: NEXUS_MVN_URL,
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

        stage('Initialize'){
            steps {
                script {
                    def dockerHome = tool 'docker'
                    env.PATH = "${dockerHome}/bin:${env.PATH}"
                }
            }
        }

        stage('Docker Build') {
            steps { 
                    echo 'Building docker Image'
                    sh 'docker build -t createpdf:latest .'
                }
        }
        
        stage('Docker tag') {
            steps { 
                    echo 'Building docker Image'
                    sh 'docker tag createpdf:latest $NEXUS_DOCKER_URL/createpdf:latest'
                }
        }
        
       stage('Docker Login') {
            steps {
                echo 'Nexus Docker Repository Login'
                script{
                    withCredentials([usernamePassword(credentialsId: NEXUS_CREDENTIAL_ID, usernameVariable: 'USER', passwordVariable: 'PASS' )]){
                       sh 'docker login -u $USER -p $PASS $NEXUS_DOCKER_URL'
                    }
                   
                }
            }
        }

        stage('Docker Push') {
            steps {
                echo 'Pushing Image to docker hub'
                sh 'docker push $NEXUS_DOCKER_URL/createpdf:latest'
            }
        }
        
        stage('Docker Stop Application') {
            steps {
                echo 'Starting application'
                sh 'docker stop $CONTAINER_NAME || true && docker rm $CONTAINER_NAME || true'
            }
        }
        
        stage('Docker Run') {
            steps {
                echo 'Starting application'
                sh 'docker run -d -p 8888:8080 --name $CONTAINER_NAME $NEXUS_DOCKER_URL/createpdf:latest'
            }
        }
    }
}
```


999. add jenkins to docker group:
```
https://stackoverflow.com/questions/47854463/docker-got-permission-denied-while-trying-to-connect-to-the-docker-daemon-socke
```