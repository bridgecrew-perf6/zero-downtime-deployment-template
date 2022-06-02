# zero-downtime-deployment-template

`Codedeploy`, `Nginx`, `Ec2`, `Github Action`, `Codedeploy Agent` 를 사용한 **서버 무중단 배포**에 대한 템플릿입니다.


## 전체 흐름도

![image](https://user-images.githubusercontent.com/83503188/171546060-ab0aff64-f076-4d07-806d-330d2e91ec77.png)


## Github Actions

### workflow

```yaml
# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.
# This workflow will build a Java project with Gradle and cache/restore any dependencies to improve the workflow execution time
# For more information see: https://help.github.com/actions/language-and-framework-guides/building-and-testing-java-with-gradle

permissions:
  id-token: write
  contents: read
  
on:
  workflow_dispatch:
  
env: 
  S3_BUCKET_NAME: logging-system-deploy2
  PROJECT_NAME: playground-logging
  
jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'
        
    - name: Grant execute permission for gradlew
      run: chmod +x gradlew 
      
    - name: Build with Gradle
      uses: gradle/gradle-build-action@0d13054264b0bb894ded474f08ebb30921341cee
      with:
        arguments: build
        
    - name: Make zip file
      run: zip -r ./$GITHUB_SHA.zip .
      shell: bash
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}
    - name: Upload to S3
      run: aws s3 cp --region ap-northeast-2 ./$GITHUB_SHA.zip s3://$S3_BUCKET_NAME/$PROJECT_NAME/$GITHUB_SHA.zip
    - name: Code Deploy
      run: aws deploy create-deployment --application-name logging-system-deploy --deployment-config-name CodeDeployDefault.AllAtOnce --deployment-group-name develop --s3-location bucket=$S3_BUCKET_NAME,bundleType=zip,key=$PROJECT_NAME/$GITHUB_SHA.zip

```

## CodeDeploy

CodeDeploy를 이용하여 S3에 올라간 jar파일을 EC2에 배포

### CodeDeploy Agent 설치

```shell
# 패키지 매니저 업데이트, ruby 설치
sudo apt update
sudo apt install ruby
sudo apt install wget

# 서울 리전에 있는 CodeDeploy 리소스 키트 파일 다운로드
cd /home/ec2-user
wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install

# 설치 파일에 실행 권한 부여
chmod +x ./install

# 설치 진행 및 Agent 상태 확인
sudo ./install auto
sudo service codedeploy-agent status

sudo service codedeploy-agent start
```

### EC2에 IAM 역할 부여
![image](https://user-images.githubusercontent.com/83503188/171547139-aaec53af-8249-463b-8aeb-0038dc576c21.png)
![image](https://user-images.githubusercontent.com/83503188/171547159-591db4f5-bdd4-41ed-afa1-b6f0de7ed043.png)
![image](https://user-images.githubusercontent.com/83503188/171547176-af61bb2b-5e60-403b-b38f-1d356bcaeb16.png)
![image](https://user-images.githubusercontent.com/83503188/171547202-d10c4340-7f59-42f5-ae58-50982dd6f9b1.png)
![image](https://user-images.githubusercontent.com/83503188/171547209-d1510930-6fc2-4c3f-9447-dc72bab89822.png)

### CodeDeploy IAM 역할 생성


![image](https://user-images.githubusercontent.com/83503188/171547266-43908295-ad29-404e-b7ef-59eb434a9952.png)
![image](https://user-images.githubusercontent.com/83503188/171547276-8cae58f8-d2af-433a-ba57-eb0ee3b71874.png)
![image](https://user-images.githubusercontent.com/83503188/171547286-375a91f5-2c70-4964-91a3-978bb2448234.png)
![image](https://user-images.githubusercontent.com/83503188/171547293-f1d6dcba-f360-48a9-8586-4b8a592521c6.png)
![image](https://user-images.githubusercontent.com/83503188/171547302-41ebff40-d225-4836-9fb0-2c1518c3b943.png)
![image](https://user-images.githubusercontent.com/83503188/171547314-73b703d6-23c5-4217-abb3-8b742b949248.png)
![image](https://user-images.githubusercontent.com/83503188/171547321-3e6c90c0-20aa-4a75-aa56-8a3dea5ff9e1.png)

### 스크립트 추가

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/playground-logging/
    overwrite: yes

permissions:
  - object: /
    pattern: "**"
    owner: root
    group: root

hooks:
  ApplicationStart:
    - location: scripts/run_new_was.sh
      timeout: 180
      runas: root
    - location: scripts/health_check.sh
      timeout: 180
      runas: root
    - location: scripts/switch.sh
      timeout: 180
      runas: root
```

## Nginx

### Nginx 설치와 설정

```shell
sudo amazon-linux-extras install nginx1

cd /etc/nginx/sites-available/test.conf


```

```
server {
    listen 80;
    listen [::]:80;
    server_name _;

    include /home/ec2-user/service_url.inc; // 1)

    location / {
        proxy_set_header    X-Forwarded-For $remote_addr;
        proxy_set_header    Host $http_Host;
        proxy_pass          $service_url; // 2)
        }
}

```

```shell
sudo ln -s /etc/nginx/sites-available/test.conf /etc/nginx/sites-enabled/
```

1. 다른 곳에 존재하는 설정 파일을 불러온다.
2. $service_url로 요청을 보낼 수 있도록 하는 프록시 설정

```shell
sudo vi /home/ec2-user/service_url.inc
```

```
# service_url.inc

set $service_url http://127.0.0.1:8081;
```

```shell
sudo service nginx start
sudo service nginx status
```