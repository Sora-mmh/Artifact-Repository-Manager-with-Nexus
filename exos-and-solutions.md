# Cloud & IaaS - DigitalOcean — Exercises & Solutions

This module guides you through deploying a simple NodeJS app on a DigitalOcean droplet. You will clone and prepare the app, create a cloud server, install NodeJS, deploy the app, and configure firewall access.

**Repository to use:**  
[https://gitlab.com/twn-devops-bootcamp/latest/05-cloud/cloud-basics-exercises](https://gitlab.com/twn-devops-bootcamp/latest/05-cloud/cloud-basics-exercises)

***

## Exercise 0: Clone Git Repository

**Task:**  
- Clone the provided repository locally.  
- Remove its Git history and initialize it as your own project repository.  
- Push it to your remote GitLab repository.

**Solution:**
```sh
# clone repository & change into project directory
git clone git@gitlab.com:twn-devops-bootcamp/latest/05-cloud/cloud-basics-exercises.git
cd node-project

# remove old Git history and initialize new repository
rm -rf .git
git init
git add .
git commit -m "initial commit"

# add your remote repository and push
git remote add origin git@gitlab.com:{gitlab-user}/{gitlab-repo}.git
git push -u origin master
```

***

## Exercise 1: Package NodeJS App

**Task:**  
- Package your NodeJS app into a single tarball artifact using `npm pack`.

**Solution:**
```sh
cd app
npm pack
```

***

## Exercise 2: Create a New Server on DigitalOcean

**Task:**  
- Create a new Droplet server on DigitalOcean to host your app.

**Summary of steps:**
- Log in to your DigitalOcean account.  
- Click **Create > Droplet**.  
- Choose your Droplet settings: OS (e.g., Ubuntu 22.04), region, size, authentication (SSH keys or password), and optional features like monitoring.  
- Click **Create Droplet**.  
- Wait for the Droplet to be ready and note its public IP address.

(For detailed guidance, you can refer to DigitalOcean tutorials on creating droplets.)[1][2][3]

***

## Exercise 3: Prepare Server to Run Node App

**Task:**  
- Connect to your new server via SSH.  
- Install NodeJS and npm on it.

**Solution:**
```sh
ssh root@{server-ip-address}

# update package lists
apt update

# install nodejs and npm
apt install -y nodejs npm
```

***

## Exercise 4: Copy App to Droplet

**Task:**  
- Copy the packaged NodeJS application tarball to the new droplet.

**Solution:**
```sh
# from your local project root folder
scp bootcamp-node-project-1.0.0.tgz root@{server-ip-address}:/root
```

***

## Exercise 5: Run Node App

**Task:**  
- SSH into the droplet.  
- Unpack the artifact.  
- Install the app dependencies.  
- Run the NodeJS app.

**Solution:**
```sh
ssh -i ~/id_rsa root@{server-ip-address}

# unpack the tarball
tar -zxvf bootcamp-node-project-1.0.0.tgz

# go into the unpacked folder ("package")
cd package

# install dependencies
npm install

# start the application
node server.js
```

***

## Exercise 6: Access from Browser - Configure Firewall

**Task:**  
- Open the port the NodeJS app listens on (typically port 3000) in the DigitalOcean droplet’s firewall.  
- Confirm the app is accessible from your browser using the droplet's public IP and the open port.

**Typical steps:**
- If using UFW firewall on the droplet:
```sh
ufw allow 3000/tcp
ufw reload
```
- Or configure the DigitalOcean Cloud Firewall via the control panel to allow inbound TCP traffic on port 3000.

- Then, from your local browser, navigate to:  
```
http://{server-ip-address}:3000
```
