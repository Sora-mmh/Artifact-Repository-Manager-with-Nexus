# Nexus Repository Manager on a Cloud Server 

## 1. Installing and Running Nexus on a Cloud Server

### Prerequisites:
- A DigitalOcean Droplet with at least **4 GB RAM** (8 GB recommended).
- Firewall allowing access to **SSH port 22**.

### Steps:

1. **Log in and create Droplet**  
   Use your DigitalOcean account to create a new Droplet with sufficient memory and a Linux distribution.

2. **SSH into the server**  
   ```bash
   ssh root@x.x.x.x
   ```

3. **Install Java 8 runtime** (required for Nexus)  
   ```bash
   apt update
   apt install openjdk-8-jre-headless
   ```

4. **Download and extract Nexus**  
   ```bash
   cd /opt
   wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
   tar -zxvf latest-unix.tar.gz
   ```
   This creates two directories:  
   - `nexus-3.46.0-01` (Nexus application files)  
   - `sonatype-work` (Nexus data storage)

5. **Create a dedicated user to run Nexus**  
   ```bash
   adduser nexus
   ```

6. **Set ownership of Nexus directories**  
   ```bash
   chown -R nexus:nexus nexus-3.46.0-01
   chown -R nexus:nexus sonatype-work
   ```

7. **Configure Nexus to run as the nexus user**  
   Edit the file:  
   `/opt/nexus-3.46.0-01/bin/nexus.rc`  
   Add the line:  
   ```bash
   run_as_user="nexus"
   ```

8. **Start Nexus as the nexus user**  
   ```bash
   su - nexus
   /opt/nexus-3.46.0-01/bin/nexus start
   ```

9. **Verify Nexus is running**  
   - Check Nexus process ID:  
     ```bash
     ps aux | grep nexus
     ```
   - Check listening ports (requires net-tools):  
     ```bash
     netstat -tlnp
     ```
   Nexus usually listens on port **8081**.

10. **Open firewall port 8081**  
    In DigitalOcean control panel, add a firewall rule to allow inbound TCP traffic on port **8081** from all IP addresses.

11. **Access Nexus Web Interface**  
    Open a browser and go to:  
    ```
    http://:8081
    ```

## 2. Nexus Initial Access and Admin User

- A predefined **admin** user exists.
- The initial admin password is stored in the file:  
  ```
  /opt/sonatype-work/nexus3/admin.password
  ```
- On first login with this password, you will be prompted to change it.
- After changing the password, the `admin.password` file is automatically deleted for security.

## 3. Nexus Repository Types

Within the Nexus UI (under Repositories):

- **Proxy Repository**  
  Acts as a cache for remote repositories (e.g., Maven Central).  
  Clients retrieve artifacts from Nexus, which proxies the remote repository transparently.

- **Hosted Repository**  
  Primary storage for your artifacts (e.g., releases, snapshots).  
  Supports version policies (release-only, snapshot-only) and can store third-party binaries.

- **Group Repository**  
  Combines multiple repositories (proxy and hosted) into a single endpoint to simplify client configuration.

## 4. Publishing Artifacts to Nexus

### Create a Nexus User with Permissions

1. In Nexus UI, go to **Security > Users** and create a new local user.  
2. Initially assign the `nx-anonymous` role for browsing.  
3. Create a new **Role** (under **Security > Roles**) and add minimal required privileges:  
   - View, read, add, delete privileges for repositories.  
   - `nx-repository-view-maven2-*-*` for full control over Maven2 repositories.  
4. Assign this role to your new user and remove any unnecessary roles like `nx-anonymous`.

### Configure Gradle to Publish to Nexus

Add the following in your `build.gradle`:

```groovy
plugins {
    id 'java-library'
    id 'maven-publish'
}

version '1.0.0-SNAPSHOT'

publishing {
    publications {
        maven(MavenPublication) {
            artifact("build/libs/my-app-$version.jar") {
                extension 'jar'
            }
        }
    }
    repositories {
        maven {
            name 'nexus'
            def releasesRepoUrl = 'http://:8081/repository/maven-releases/'
            def snapshotsRepoUrl = 'http://:8081/repository/maven-snapshots/'
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl
            allowInsecureProtocol = true
            credentials {
                username project.repoUser       // defined in gradle.properties
                password project.repoPassword   // defined in gradle.properties
            }
        }
    }
}
```

To publish after building:

```bash
./gradlew publish
```

### Configure Maven to Publish to Nexus

In your `pom.xml`, add:

```xml

    
        
            org.apache.maven.plugins
            maven-deploy-plugin
            2.7
        
    



    
        nexus-releases
        http://:8081/repository/maven-releases/
    
    
        nexus-snapshots
        http://:8081/repository/maven-snapshots/
    

```

In your Maven settings (`~/.m2/settings.xml`), configure credentials:

```xml

  
    
      nexus-releases
      YourNexusUsername
      YourNexusPassword
    
    
      nexus-snapshots
      YourNexusUsername
      YourNexusPassword
    
  

```

Deploy with:

```bash
mvn deploy
```

- Version naming determines deployment target:  
  - Versions ending with `SNAPSHOT` go to snapshots repo  
  - Other versions go to releases repo

### Using Nexus as npm Registry

- Nexus can host private npm packages.  
- Refer to [Deploying Private npm Packages to Nexus (Blog Post)](https://levelup.gitconnected.com/deploying-private-npm-packages-to-nexus-a16722cc8166) for detailed guide.

## 5. Nexus REST API

- Nexus provides REST endpoints for scripting and automation.  
- All API calls require authentication of a user with necessary privileges.

Examples (replace `user:pwd` and URLs accordingly):

- List all repositories:  
  ```bash
  curl -u user:pwd -X GET 'http://:8081/service/rest/v1/repositories'
  ```

- List components (artifacts) in a repository:  
  ```bash
  curl -u user:pwd -X GET 'http://:8081/service/rest/v1/components?repository=maven-snapshots'
  ```

- Get assets of a component (replace ``):  
  ```bash
  curl -u user:pwd -X GET 'http://:8081/service/rest/v1/components/'
  ```

## 6. Blob Stores

- Blob stores are the storage backend of Nexus for all uploaded binaries.  
- Can be on local filesystem or cloud storage.  
- Default blob store is named **default**, located at:  
  ```
  /opt/sonatype-work/nexus3/blobs
  ```
- Each blob store can be assigned to one or more repositories or groups.  
- Important constraints:  
  - Blob stores cannot be modified (e.g., size changes).  
  - Cannot delete blob store if assigned to any repository.  
  - Repositories cannot span multiple blob stores or be reassigned once assigned.

- Plan blob stores thoughtfully based on storage and repository layout needs.

## 7. Components vs Assets

- A **component** (e.g. a Java app) consists of one or more **assets** (actual files):  
  - JAR files, POMs, checksum files, etc.

- In Maven/JAR repositories:  
  - Each component’s assets belong only to that component version.

- In Docker repositories:  
  - Assets are image layers shared across components (images).

## 8. Cleanup Policies and Scheduled Tasks

- **Cleanup Policies:** Define rules to delete unused components in repositories.  
- Can preview cleanup results before activation.

- **Associating Cleanup Policies:** Link them to repositories via repository configuration.

- **Cleanup Execution:**  
  - Scheduled cleanup tasks mark components for deletion — they become invisible but data remain on disk.  
  - Physical deletion requires running the "Admin - Compact blob store" scheduled task.

- **Testing Scheduled Tasks:**  
  - Can be run manually via Nexus UI for immediate effect.
