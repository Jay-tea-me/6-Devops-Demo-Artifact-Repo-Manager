# Description
This repo is a demo on how to publish artifacts to Nexus running on a droplet

## Technologies Used
- Nexus
- DigitalOcean
- Linux
- Java
- Gradle
- Maven

## Steps
### A. Install & Run Nexus

1. Select a droplet with enough storage
2. Add Firewall
3. Add Java 17 to remote server 
   1. ssh into remote server
  ```sh
    ssh root@<ip>
  ```
4. Install java
  ```sh 
    apt install openjdk-11-jre-headless 
  ```
5. ```cd /opt/```
6. Download nexus 
   1. get download link from https://help.sonatype.com/en/download.html
   2. Copy unix link 
   3. Untar
```sh 
  tar -zxvf nexus-3.82.0-08-linux-x86_64.tar.gz
```
FYI: 
- The nexus folder = bin
- Sona-work = configs and data, logs, last accessed, uploaded files + metadata, backups
7. Create nexus service user
   1.   create the user: ```adduser nexus```
   2. Add permissions for nexus folder (exe) and sonatype-work (r/w)
      1. ```chown -R nexus:nexus nexus-dir```
      2. ```chown -R nexus:nexus sonatype-work```
   3. Set nexus config so it runs as nexus user
      4. vim nexus-<version>/bin/nexus.rc
      5. ```run_as_user="nexus”```
8. Switch to nexus user
   1. ```su - nexus```
9. Start!
```shell
    /opt/nexus-<version>/bin/nexus start
```  
10. Check process running and port
    1. ```ps aux```
    2. ```netstat -lpnt```
11. Open port 8081 on firewall

### B. Create New Nexus User (Manually)
1. Login to nexus with admin:
   1. username = admin
   2. password = cat /opt/sonatype-work/nexus3/admin.password
2. Navigate to Security -> Users
3. Give user permission (least amount needed)
4. Create nexus role
   1. nx-repository-view-maven2-maven-snapshots-*


### C. Publish Artifact to Repo
GRADLE
1.  setup publications (using maven plugin)
``` 
publishing {
    publications {
        maven(MavenPublication) {
            artifact("build/libs/my-app-$version"+".jar"){
                extension 'jar'
            }
        }
    }

    repositories {
        maven {
            name 'nexus'
            url "http://xx.xx.xx.xx:xxxx/repository/maven-snapshots/" 
            allowInsecureProtocol = true
            credentials {
                username project.repoUser
                password project.repoPassword
            }
        }
    }
}
```
```allowInsureProtocol = true``` allows http fetch
2. Create gradle.properties file and add:
```
repoUser = username
repoPassword = password
```
3. ```gradle build```
4. ```gradle publish ```

---
MAVEN
1. Configure pom file -> add maven-deploy-plugin
2. And add distributionManagement:
```xml
    <distributionManagement>
        <snapshotRepository>
            <id>nexus-snapshots</id>
            <url>http://xx.xx.xx.xx:xxxx/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
```
3. Configure ~/.m2/settings.xml file
5. ls -a | grep .m2 (user home)
```xml
<settings>
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>nexus</username>
            <password>pwd</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>nexus</username>
            <password>pwd</password>
        </server>
    </servers>
</settings>
```
6. ```mvn package```
7. ```mvn deploy```

-----
### D Query Nexus with API
check repos available:
- ```curl -u user:pwd -X GET ‘http://http://46.101.244.20:8081/service/rest/v1/repositories'```

check components available in specific repo:
- ```curl -u user:pwd -X GET 'http://46.101.244.20:8081/service/rest/v1/components?repository=maven-snapshots'```

check specific component:
- ```curl -u user:pwd -X GET 'http://46.101.244.20:8081/service/rest/v1/components/<component_id>’```

---
### E Setup Cleanup Policies
Setup rules to remove old or unused components.
1. Create resource policy
2. Associate resource to repo
3. navigate repo -> maven-snapshots
4. and apply resource

Deleting is a soft delete ... i.e. it is marked for delete and you need to compact the blob storage to hard delete.

You can test manually under system->tasks.
