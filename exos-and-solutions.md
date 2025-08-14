# Artifact Repository Manager with Nexus — Exercises & Solutions

You and other teams in 2 different projects in your company see that they have many different small projects, including the NodeJS application you built in the previous step, the java-gradle helper application and so on. You discuss and decide it would be a good idea to be able to keep all these app artifacts in 1 place, where each team can keep their app artifacts and can access them when they need.

So they ask you to setup Nexus in the company and create repositories for 2 different projects.

***

## Exercise 1: Install Nexus on a server
**Task:**  
If you already followed the demo in the Nexus module for installing Nexus, then you can use that one.
If not, you can watch the module demo video to install Nexus.

**Solution:**  
If you need to install Nexus on a new droplet - follow the steps Outlined in [Lecture 2 - Install and Run Nexus on a cloud server](https://techworld-with-nana.teachable.com/courses/1108792/lectures/28658559)

***

## Exercise 2: Create npm hosted repository
**Task:**  
For a Node application you:
* Create a new npm hosted repository with a new blob store

**Solution:**  
- Create a new Blob store

![blob_store](Exercise2_1.png)

- Create a new **npm (hosted)** repository that uses the new store

![npm_hosted](Exercise2_2.png)

***

## Exercise 3: Create user for team 1
**Task:**  
* You create Nexus user for the project 1 team to have access to this npm repository

**Solution:**  
- Create a new role, which has `"nx-repository-admin-npm-repo1-*"` and `"nx-repository-view-npm-*-*"` Applied Privileges

![role_1_creation](Exercise3_1.png)

- Create a new user, and grant this new role to it

![user_1_creation](Exercise3_2.png)

***

## Exercise 4: Build and publish npm tar
**Task:**  
You want to test that the project 1 user has correct access configured. So you:
* Build and publish a nodejs tar package to the npm repo

*Use: Node application from Cloud & IaaS Basics exercises*

*Hint:*
```bash
# for publishing project tar file 
npm login --registry={npm-repo-url-in-nexus}
npm publish --registry={npm-repo-url-in-nexus} {package-name}
```

**Solution:**  
- Clone the repository [found here](https://gitlab.com/twn-devops-bootcamp/latest/04-build-tools/node-app)
- Run *npm pack* command
- Execute `npm login --registry=http://{nexus-ip}:8081/repository/{repo-name}/`
- Execute `npm publish --registry=http://{nexus-ip}:8081/repository/{repo-name}/ {package-name}.tgz`

***

## Exercise 5: Create maven hosted repository
**Task:**  
For a Java application you:
* Create a new maven hosted repository

**Solution:**  
- Create a new **maven2** repository which uses the blob store created in the earlier step

![maven_2_repo](Exercise5_1.png)

***

## Exercise 6: Create user for team 2
**Task:**  
* You create a Nexus user for project 2 team to have access to this maven repository

**Solution:**  
- Create a new role, which has `"nx-repository-admin-maven2-maven-central-*"` and `"nx-repository-view-maven2-*-*"` Applied Privileges
- Create a new user, and grant this new role to it

![user_2_creation](Exercise6_1.png)

***

## Exercise 7: Build and publish jar file
**Task:**  
You want to test that the project 2 user has the correct access configured and also upload the first version. So:
* Build and publish the jar file to the new repository using the team 2 user.

*Use the java-app application from the Build Tools module*

**Solution:**  
- Clone the repository [found here](https://gitlab.com/twn-devops-bootcamp/latest/04-build-tools/java-app)
- Your **build.gradle** file should contain the following, ensuring that the version is *not* a snapshot and that your credentials are defined in a **gradle.properties** file:

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.1.0-SNAPSHOT'
    id 'io.spring.dependency-management' version '1.1.0'
}

group 'com.example'
version '1.0.0'
sourceCompatibility = 17

apply plugin: 'maven-publish'

publishing {
    publications {
        mavenJava(MavenPublication) {
            artifact("build/libs/java-app-$version" + ".jar"){
              extension 'jar'
            }
        }
    }
    repositories {
        maven {
            name = "nexus"
            url = "http://{nexus-ip}:8081/repository/{repo-name}/"
            allowInsecureProtocol = true
            credentials {
                username project.repoUser
                password project.repoPassword
            }
        }
    }
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
    maven { url 'https://repo.spring.io/snapshot' }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation group: 'net.logstash.logback', name: 'logstash-logback-encoder', version: '7.3'
    testImplementation group: 'junit', name: 'junit', version: '4.13.2'
    implementation "javax.annotation:javax.annotation-api:1.3.2"
}
```

- Execute `gradle build` and then `gradle publish`

***

## Exercise 8: Download from Nexus and start application
**Task:**  
* Create new user for droplet server that has access to both repositories
* On a digital ocean droplet, using Nexus Rest API, fetch the download URL info for the latest NodeJS app artifact
* Execute a command to fetch the latest artifact itself with the download URL
* Run it on the server!

**Solution:**  
- Create a new user in the Nexus UI, and grant *both* of the roles previously created to it

![user_3_creation](Exercise8_1.png)

- Execute `curl -u {user}:{password} -X GET 'http://{nexus-ip}:8081/service/rest/v1/components?repository={repo-name}&sort=version'` on the DigitalOcean droplet
- Execute `wget` followed by the result of the previous command

![wget_response](Exercise8_2.png)

- Execute `java -jar java-app-1.0.jar`

***

## Exercise 9: Automate
**Task:**  
You decide to automate the fetching from Nexus and starting the application So you:
* Write a script that fetches the latest version from npm repository. Untar it and run on the server!
* Execute the script on the droplet

**Solution:**  
- Create a **.sh** file on the DigitalOcean droplet and ensure it has execute permissions
- The **.sh** file should contain the following:

```bash
#!/bin/bash

# save the artifact details in a json file
curl -u {user}:{password} -X GET 'http://{nexus-ip}:8081/service/rest/v1/components?repository={repo-name}&sort=version' | jq "." > artifact.json

# grab the download url from the saved artifact details using 'jq' json processor tool
artifactDownloadUrl=$(jq -r '.items[].assets[].downloadUrl | select(endswith(".jar"))' artifact.json)

# fetch the artifact with the extracted download url using 'wget' tool
wget --http-user={user} --http-password={password} "$artifactDownloadUrl" -O java-app.jar

# Run the Java application
java -jar java-app.jar
```

- Execute the shell script on the server

![shell_script](Exercise9_1.png)
