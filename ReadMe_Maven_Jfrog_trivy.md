#                                                Maven - JFrog 


create the t3.large or medium machine..
# java install
sudo apt update -y
sudo apt install openjdk-17-jdk -y

# Install maven latest stable version
cd /opt
wget https://dlcdn.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz 
tar xzvf apache-maven-3.9.6-bin.tar.gz 
mv apache-maven-3.9.6-bin.tar.gz  maven
echo 'export M2_HOME=/opt/maven' >> ~/.bashrc
echo 'export PATH=${M2_HOME}/bin:${PATH}' >> ~/.bashrc
source ~/.bashrc
mvn  –version

**create the springboot project with build tool using maven**
git clone https://github.com/spring-projects/spring-petclinic
mvn validate
mvn compile
mvn test
mvn package

java -jar target/my-app-1.0-SNAPSHOT.jar ==> runs the spring-boot appilication as a process
nano /root/simple-java-maven-app/src/main/java/com/mycompany/app/App.java #Add Custom data


**mvn clean install does the below steps**
mvn clean
mvn clean compile
mvn clean compile test package
mvn clean compile test package verify
mvn clean compile test package verify install
mvn package -Dmaven.test.skip=true   => creates the jar without running the  test


mvn clean compile test package verify install deploy


# install Docker and then Jfrog if need to pull the jfrog image
$ sudo apt update -y
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
$ sudo add-apt-repository "deb [arch=$(dpkg --print-architecture)] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
$ sudo apt update -y
$ sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
$ sudo usermod -aG docker $USER
$ newgrp docker
$ docker pull docker.bintray.io/jfrog/artifactory-oss:latest

**Install Jfrog**
curl https://api.bintray.com/orgs/jfrog/keys/gpg/public.key
echo "deb https://jfrog.bintray.com/artifactory-debs bionic main" | tee /etc/apt/sources.list.d/jfrog.list
apt-get update -y
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 6B219DCCD7639232
apt update
sudo apt install jfrog-artifactory-oss -y
sudo systemctl start artifactory.service
sudo systemctl enable artifactory.service

Runs on **8081 and 8082** port
Default login 
Username: admin
Password: password

**Configure Nginx as a Reverse Proxy**
apt-get install nginx -y
nano /etc/nginx/sites-available/jfrog.conf
Copy the below lines
```
upstream jfrog {
  server <ip-adresss>:8082 weight=100 max_fails=5 fail_timeout=5;
}

server {
  listen          80;
  server_name     <server-public-dns-hostname/route53-dns-name>;

  location / {
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://jfrog/;
  }
}
```
Save and close the file then activate the Nginx virtual host with the following command:
ln -s /etc/nginx/sites-available/jfrog.conf /etc/nginx/sites-enabled/
verify the Nginx for any syntax error with the following command:
nginx -t
systemctl restart nginx

**It is recommended to secure JFrog with Let's Encrypt SSL. First, add the Certbot repository with the following command:**
apt-get install software-properties-common -y
add-apt-repository ppa:ahasenack/certbot-tlssni01-1875471
Next, update the repository and install the Certbot client with the following command:

apt-get update -y
apt-get install certbot python3-certbot-nginx -y
Once the Certbot client is installed, run the following command to download and install Let's Encrypt SSL for your website:

certbot --nginx -d jfrog.linuxbuz.com


First create   for authentication**SAML SSO/vault/OAuth SSO** and authtorization
**Jfrog Artifactory Basic workflow with maven repo and docker repo**

Create the Group
Create the User and add to the group 
Create the Repo
    Types of repo :  1) local
                    2) remote 
                    3) Vitual 

Create the Permissions

**How to upload ?**

1) edit the pom.xml and add reposiotory created in jfrog server
2) edit a settings.xml and add access of the repo
    Select libs-release-local and click on “Set Me Up” and scroll down and click on generate settings. Copy the contexts to /root/.m2/settings.xml
3) make sure element id of repository/snapshotRepository must be equal to  "server" element
4) mvn deploy



**How to download ?**

1) make sure th package is in artifactory
2) modify pom.xml witht eh dependencies element
3) make sure default registry is artifactory instaed of maven central.
        change the mirrors section in  settings.xml ,specify the repo url in jfrog server
4) mvn package   => call goes to artifactory rather maven central beacuse it caches the all dependencies earlier
                    => always dev team connecting to internet for dependencies is breach of security

**Reference**
https://www.howtoforge.com/tutorial/ubuntu-jfrog/


**TRIVY**

sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update -y
sudo apt-get install trivy -y


$docker run --rm -v $WORKSPACE:/root/.cache/ aquasec/trivy:0.17.2 -q image --exit-code 1 --severity CRITICAL --light openjdk:8-jdk-alpine 

Here’s a breakdown of the command and its components:

docker run: Executes a Docker container.
--rm: Automatically removes the container after it finishes running, ensuring a clean environment.
-v $WORKSPACE:/root/.cache/: Mounts the $WORKSPACE directory from the host into the container's cache directory, allowing Trivy to store and access cached data for faster subsequent scans.
aquasec/trivy:0.17.2: Specifies the Trivy container image and its version to run.
-q image: Performs an image vulnerability scan in a quiet mode, suppressing unnecessary output.
--exit-code 1: Sets the exit code to 1 if vulnerabilities are found, allowing you to programmatically determine if the scan detected critical vulnerabilities.
--severity CRITICAL: Specifies the severity level as CRITICAL, indicating that Trivy should focus on vulnerabilities with a critical severity level.
--light: Uses the light mode for vulnerability detection, which may reduce the scan time but might result in potential false negatives.