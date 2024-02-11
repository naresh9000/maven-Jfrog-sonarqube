# Sonar-qube installation and configuration for production grade working

create the t3.large or medium machine..

# java install
sudo apt update -y
sudo apt install openjdk-17-jdk -y
sudo apt install openjdk-17-jre -y

# Install maven latest stable version
cd /opt
sudo wget https://dlcdn.apache.org/maven/maven-3/3.8.8/binaries/apache-maven-3.8.8-bin.tar.gz
sudo tar xzvf apache-maven-3.8.8-bin.tar.gz
sudo mv apache-maven-3.8.8/ maven
echo 'export M2_HOME=/opt/maven' >> ~/.bashrc
echo 'export PATH=${M2_HOME}/bin:${PATH}' >> ~/.bashrc
source ~/.bashrc
mvn  –-version

# Install and Configure PostgreSQL
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl start postgresql

sudo passwd postgres
su - postgres
createuser sonar

-Login to PostgreSQL using command below
psql
1) ALTER USER sonar WITH ENCRYPTED password 'my_strong_password';

2) CREATE DATABASE sonarqube OWNER sonar;

3) GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;
4) \l - to check the created database

4) \du -To check the created database user

5) \q -exit




# Download and Install SonarQube


sudo apt-get install zip -y
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.0.0.68432.zip
sudo unzip sonarqube-10.0.0.68432.zip
sudo mv sonarqube-10.0.0.68432 /opt/sonarqube
**Add SonarQube Group and User**
sudo groupadd sonar
-Create a sonar user and set /opt/sonarqube as the home directory and assign to group.
sudo useradd -d /opt/sonarqube -g sonar sonar
-Grant the sonar user access to the /opt/sonarqube directory.
sudo chown sonar:sonar /opt/sonarqube -R

or use docker image
docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube


# Configure SonarQube

=> sudo nano /opt/sonarqube/conf/sonar.properties

Uncomment the lines, and add the database user and password you created earlier
 
``` 
sonar.jdbc.username=sonar
sonar.jdbc.password=my_strong_passowrd
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```


=> sudo nano /opt/sonarqube/bin/linux-x86-64/sonar.sh
Add this  line   "RUN_AS_USER=sonar"



**Setup Systemd service**

sudo nano /etc/systemd/system/sonar.service


```
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

sudo systemctl enable sonar
sudo systemctl start sonar
sudo systemctl status sonar

**Note::**
Modifying kernel limits allows SonarQube to access and utilize these resources effectively.Some parameters involved are.....
Resource Utilization,Concurrency and Scalability,File System Limits,Networking Limits,Optimizing Performance

Modify Kernel System Limits
=> sudo nano /etc/sysctl.conf
Add below lines
```
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```
Save and exit the file. Reboot the server.


Wait for 5 min and access the server on port 9000 as below
http://ec2-3-228-22-190.compute-1.amazonaws.com:9000
Default credentials admin/admin
Login and change the password. 


# Addtionall work for the production grade use
# Configuring the Reverse Proxy using Nginx

Create the hosted in AWS Route-53 and create the Records for routing .
sudo apt install nginx -y
=>Create a new Nginx configuration file for the site.
sudo nano /etc/nginx/sites-enabled/sonarqube
```
server {
listen 80;
server_name <route-53-domain hosta name>;
location / {
proxy_pass http://127.0.0.1:9000;
}
}
```

sudo nginx -t  => for checking the synatx errors
sudo systemctl enable nginx
sudo systemctl restart nginx
sudo systemctl status nginx


**But the domain is working on HTTP protocol and we need to make it secure by making it work with the HTTPS protocol with SSL certificate.**

sudo snap install --classic certbot
sudo certbot --nginx




# Scanning using the Sonar-Scanner

wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.7.0.2747-linux.zip
unzip sonar-scanner-cli-4.7.0.2747-linux.zip
mv sonar-scanner-4.7.0.2747-linux/ /opt/sonar-scanner
nano /opt/sonar-scanner/conf/sonar-scanner.properties
sonar.host.url=http://localhost:9000
sonar.sourceEncoding=UTF-8


nano /etc/profile.d/sonar-scanner.sh
#/bin/bash
export PATH="$PATH:/opt/sonar-scanner/bin"

reboot the server to effect the changes

source /etc/profile.d/sonar-scanner.sh

env | grep PATH

sonar-scanner -v

cd <project-location>
sonar-scanner \
  -Dsonar.projectKey=Terraform \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://ec2-35-153-213-51.compute-1.amazonaws.com:9000 \
  -Dsonar.login=<sqp_fefd12c5b6872932e12b32e13bf34dd7dad6791e(authentication-token)>

**without sonar-scanner**
mvn clean verify sonar:sonar \
  -Dsonar.projectKey=my-spring-boot-project \
  -Dsonar.host.url=http://ec2-18-60-112-184.ap-south-2.compute.amazonaws.com:9000 \
  -Dsonar.login=sqp_40e6b3ba9ddd71ea9209acbbc9b16a166283450d


**Reference::**
https://medium.com/@deshdeepakdhobi/how-to-install-and-configure-sonarqube-on-aws-ec2-ubuntu-22-04-c89a3f1c2447


NOTE::

quality gates focus on overall code quality thresholds and progression through the development pipeline, while quality profiles define coding standards and rules used during code analysis to assess code quality and identify areas for improvement.


**Code Coverage:** Ensure that a certain percentage of code is covered by automated tests, such as unit tests, integration tests typically ranges from 70% to 90%.

**Security Vulnerabilities:** SonarQube provides built-in security rules to detect common vulnerabilities like injection flaws, XSS, CSRF, etc.

**Code Smells and Maintainability:** complex code, duplicated code, long methods, and high cyclomatic complexity. Aim to keep the codebase clean, understandable, and maintainable.

**Performance:**potential issues like excessive database queries, inefficient algorithms, and memory leaks. 

**Dependency Management:** Monitor dependencies and third-party libraries for security vulnerabilities, licensing issues, and compatibility concerns. Regularly update dependencies to the latest stable versions and adhere to best practices for dependency management.

