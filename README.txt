# vngo-backend
VNGo is a drive-booking apps system. This system has 3 apps in total, they are VNGo, VNGo Driver and VNGo Admin.

This is an end-term project of our 4-member team in Intro To Software Engineer course.

# How to Configure Jenkins

First, updated code is push directly to feature/map, there is no test requirement.
Then create PRs to merge to main, there are two check to pass:
- continuous-integration/jenkins/branch: The commit from feature/map must pass unit test
- continuous-integration/jenkins/pr-merge: The result of merging must pass unit test.

(image)

## Create worker node Jenkins to run the job
1) Download java
sudo add-apt-repository ppa:openjdk-r/ppa -y
sudo apt update
sudo apt install openjdk-21-jdk -y
sudo update-alternatives --config java

2) Create remote working dir for the node
create working dir
sudo mkdir /jenkins-worker

3) Create the node, worker will actively connect to Jenkins controller using JNSP by chosing Launch method: Lauch agent by connecting it to the controller.
(image)

4) Download and run agent.jar
Vì Jenkins đang chạy trên host windows, nên để giao tiếp với host windows
cat /etc/resolv.conf | grep nameserver

nhưng user chạy ứng dụng jenkins cần có quyền thực thi agent.jar trong thư mục đó
sudo chmod 777 /jenkins-worker

chạy agent.jar để agent kết nối đến Jenkins Controller.
java -jar agent.jar -url http://10.255.255.254:8080/ -secret <secret given by Jenkins after create the node> -name "first-node" -webSocket -workDir "/jenkins-worker"

(image)

The node is now online

(image)

## Configure Webhook
Download GitHub plugins
Download Ngrok from https://ngrok.com/downloads/linux
Use Ngrok to expose http://localhost:8080 of Jenkins to Internet
Add webhook to the URL Ngrok handed

(image)

## Configure ruleset
Apply to branch main require status check to pass like this Image

(image)

## Configure Jenkins to run test which branches
Discover branches: Exclude branches that also filed as PRs.
With this choice, Jenkins won't create continuous-integration/jenkins/pr-head, which is the test for the commit to merge to main.
We dont need that because we already have continuous-integration/jenkins/branch doing that job
The continuous-integration/jenkins/branch will automaticaly be created by Jenkins when there is any push to feature/map
Eventhough feature/map doesn't require any test to pass to push, continuous-integration/jenkins/branch just run by Jenkins and indicate the result as green tick or red cross like this image.
(image)

Discover the pull requests from origin: Merging the pull request with the current target branch revision
This will create continuous-integration/jenkins/pr-merge.

(image)

# How to Create Image of VNGo and push to DockerHhub

*Chưa hoàn thành, đến giai đoạn CD sẽ giấu các secret sau, hiện tại mới chỉ hoàn thiện CI, ở môi trường dev trong lúc test CI, các secret được giấu nhờ Jenkins Credentials*

```
docker build -t vngo:0.0.1 .
```

```docker run -d \
  -p 8080:8080 \
  -e SERVER_PORT=8080 \
  -e DB_URL="jdbc:mysql://host.docker.internal:3306/vngo" \
  -e DB_USERNAME="root" \
  -e DB_PASSWORD="your_secure_password" \
  -e JWT_SIGNER_KEY="your_secure_key_here" \
  --name vngo-app \
  vngo:0.0.1```

```docker build -t nguyenphuc4444/vngo:latest .```

```docker login```

```docker tag nguyenphuc4444/vngo:latest nguyenphuc4444/vngo:latest```

```docker push nguyenphuc4444/vngo:latest```

```.\mvnw spring-boot:run -D"spring-boot.run.profiles"="dev"```

docker exec -u root -it jenkins-blueocean bash

apt-get update && apt-get install -y default-mysql-client

----download trivy---
curl -LO https://github.com/aquasecurity/trivy/releases/download/v0.63.0/trivy_0.63.0_Linux-64bit.tar.gz
tar -xzf trivy_0.63.0_Linux-64bit.tar.gz
chmod +x trivy
sudo mv trivy /usr/local/bin/



docker run -d --name mysql-service --network jenkins \
  -e MYSQL_ROOT_PASSWORD=${MYSQL_PASSWORD} \
  -e MYSQL_DATABASE=${MYSQL_DB} \
  -p 3308:3306 \
  mysql:8.0

-- chạy với profile defaul, trong application.yaml (default, các biển DB_URL mới có)
docker run -d --name vngo-app --network jenkins \
  -e DB_URL=jdbc:mysql://mysql-service:3306/vngo \
  -e DB_USERNAME=${MYSQL_USERNAME} \
  -e DB_PASSWORD=${MYSQL_PASSWORD} \
  -e JWT_SIGNER_KEY=daylakeydanhchodev \
  -e JWT_VALID_DURATION=3600 \
  -e JWT_REFRESHABLE_DURATION=86400 \
  -p 8080:8080 \
  vngo:2

-- trong application-dev.yaml, không có biến DB_URL nên phải dùng SPRING_DATASOURCE_URL
docker run -d --name vngo-app --network jenkins \
  -e SPRING_PROFILES_ACTIVE=dev \
  -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-service:3306/vngo \
  -e SPRING_DATASOURCE_USERNAME=${MYSQL_USERNAME} \
  -e SPRING_DATASOURCE_PASSWORD=${MYSQL_PASSWORD} \
  -e JWT_SIGNERKEY=daylakeydanhchodev \
  -e JWT_VALID_DURATION=3600 \
  -e JWT_REFRESHABLE_DURATION=86400 \
  -p 8080:8080 \
  vngo:2