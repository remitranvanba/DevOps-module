# DevOps-module - Report

## TP 01 - Docker 🚀🚀🚀

### Part 1 🛫✈️🛸

- DockerFile 🐳:

``` yaml
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

```

- to build 🔧 :
```shell
docker build -t tp1/tp1 .
```

- to launch the run in detached mode : 👨‍🦽

```shell
docker run -d --name tp1 tp1/tp1
```

❓ Why should we run the container with a flag -e to give the environment variables? 🇺🇸

💡 	 You need to launch the container with the -e flag to specify the environment variables. For a database, it is mandatory to have a superuser with a password to protect sensitive data.

## Connection to a PostGres database

- add a network 🕸️
```shell
docker network create app-network
```
- Restart the tp1 container with the network ♻️

``` shell
docker run --net=app-network -d --name=tp1 tp1/tp1
```

- Adminer 🕵️‍♂️

```shell
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

### Browser connection on port 8090

- infos de logs 🧑‍💻

```shell
    System:	PostGreSQL
    Server: tp1
    Username: usr
    Password: pwd
    Database: db	
```
❗ Connection identifiers are either supplied on the command line with the `-e` flag or specified in the dockerfile.

### Launch a database with a data set:

All scripts copied to `docker-entrypoint-initdb.d` will be executed. We therefore add to the Dockerfile 📝: 

```shell
COPY CreateScheme.sql /docker-entrypoint-initdb.d/
COPY InsertData.sql /docker-entrypoint-initdb.d/
```
### Data persistence 

To persist the data, a volume must be used. To do this, use the `-v` flag. It takes 3 parameters, one of which is optional:
- Volume name
- Path where the volume will be mounted in the container
- Option list (optional) 

❓ Why do we need a volume to be attached to our postgres container?

💡 A volume must be attached to the container in order to persist the data. If the container is destroyed and has no volume attached, the database is reset the next time it is launched.

## Backend API

### Java Application

DockerFile 🐳:

```
FROM openjdk:11
COPY ./Main.java /usr/src/
WORKDIR /usr/src/
RUN javac Main.java
CMD ["java", "Main"]
```
Build 🔧:

```
docker build -t java/backend . 
```

Run 👨‍🦽 : 

```
docker run --name backend java/backend    
```
### Springboot Application

❗ Docker ports must be properly exposed, otherwise the springboot application will not be accessible on the browser. 

DockerFile 🐳:
```
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```
Build 🔧:
```
docker build -t api/simpleapi .
```

Run 👨‍🦽 : 
```
docker run --name=simpleapi -p 8080:8080 api/simpleapi
```

❓ Why do we need a multistage build? And explain each step of this dockerfile.

💡 A multistage build is one that uses the `FROM` instruction several times. Each `FROM` constitutes a build step. This allows us to use different bases.

In our case, we first build our java application by downloading the maven dependencies. We then run it with the amazon JDK.

### Backend API 

Files downloaded from the [git](https://github.com/takima-training/simple-api-student) made available.


The Dockerfile is identical to the one above.

However, the container must be launched in the network created earlier, where our PostGreSQL database is running.

Run 👨‍🦽 : 

```
docker run --name=simpleapi-student --net=app-network -p 8081:8080 api/simpleapi
```

Pre-build app configuration for db connection: 

Application.yaml: 
```
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://tp1:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops
```
Thanks to call mapping on port 8081, you can access the database. ✅

## HTTP Server

Basic configuration:
We create a new `public-html` folder where we'll store a classic index.html.

DockerFile 🐳:
```
FROM httpd:2.4
COPY ./public-html/ /usr/local/apache2/htdocs/
```
Build 🔧:

`docker build -t http/http .`

Run 👨‍🦽 :

`docker run -dit --name my-running-app -p 8080:80 http/http`

### Configuration 
Display global apache configuration:

`docker exec my-running-app cat /usr/local/apache2/conf/httpd.conf`

### Reverse proxy

We create a conf file from the basic one:

```
docker exec my-running-app cat /usr/local/apache2/conf/httpd.conf > httpd.conf
```

We modify the apache conf file from the Dockerfile by adding:
- The server name :
`ServerName localhost`
- Proxy configuration : 
```
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://simpleapi-student:8080/
ProxyPassReverse / http://simpleapi-student:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

❓ Why do we need a reverse proxy?

💡 So as not to directly expose our container port.

DockerFile 🐳:
```
FROM httpd:2.4
COPY ./public-html/ /usr/local/apache2/htdocs/
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

Build 🔧:

`docker build -t http/http .`

Run 👨‍🦽 :

`docker run -dit --net=app-network --name my-running-app -p 8088:80 http/http`

### Link application

Docker-compose :
```
version: '3.7'

services:
  db:
    build: database
    container_name: postgres-db
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    networks:
      - app-network

  springboot-app:
    build: simple-api-student
    container_name : springboot-app-container
    environment:
      - URL=db:5432
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    networks:
      - app-network
    depends_on:
      - db
      
  my-apache-server:
    build: http-server
    container_name: my-apache-server-container
    ports:
      - "8088:80"
    networks:
      - app-network
    depends_on:
      - springboot-app
networks:
  app-network:
    driver: bridge
```

We update our springboot's application.yaml to support environment variables: 
```
applications.yaml
  datasource:
    url: jdbc:postgresql://${URL}/${POSTGRES_DB}
    username: ${POSTGRES_USER} 
    password: ${POSTGRES_PASSWORD}
```

Similarly for the apache server, we update the httpd.conf file:
```
ProxyPass / http://springboot-app-container:8080/
ProxyPassReverse / http://springboot-app-container:8080/
```

❓ Why is docker-compose so important? 🇺🇸

💡 Because it allows you to launch and configure several dockers automatically.

### Publish

Realize mainly in TP 02 with continuous integration and deployment.

❓ Document your publication commands and published images in dockerhub. 🇺🇸

💡 We tag the images to identify them and version them. This will enable us to retrieve very specific versions of our applications when deploying them.

## TP 02 - Github Action 🚀🚀🚀

### Setup Github Actions

GitHub Action is used throughout this course. It lets you launch **`jobs`**, tasks to be carried out during a deployment. You can choose when to trigger these actions.

### First steps into the CI World

We need to create a workflow, a .github folder & a workflows subfolder, containing the routines that will be executed. This routine is contained in the `main .yml` file.

### Build and test your Application

```bash
mvn clean verify
```

Launches all tests present in the project specified in pom.xml.

The `clean` flag allows you to rebuild the entire application, even if no piece of code has been touched.

❓ What are testcontainers? 🇺🇸

💡 `testcontainers` allow you to run unit tests without having to launch the entire application. Their main advantage is that they are portable and lightweight (because they are containerized).

❓ Document your Github Actions configurations:

```yml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: master
```

- 💡 We specify when to launch the `jobs`. Here, when pushing the master branch on GitHub.

```yml
jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

     #finally build your app with the latest command
      - name: Build and test with Maven
        working-directory: TP1/simple-api-student
        run: mvn clean verify
```

- 💡 Here we have a single job, `test-backend`. We specify its environment, and the steps it must follow. There's just one job here, made up of three sub-sections. In the first, we retrieve the latest commit from the repo. In the second, we set up our java, indicating its version and distribution. Finally, we launch our unit tests with `mvn clean verify`. However, we first specify the location of the working directory, as the repo contains several other projects.

### First steps into the CD World

The aim of this part is to enable continuous code delivery. With each push, we launch our test battery & build docker images which are then published on dockerhub.

❓ Secured Variables, why? 🇺🇸

💡 In order not to make public sensitive data that could make our projects vulnerable (ssh key, github login, database login etc... ).

So we declare several secret variables that we can access in our public `main.yml` without writing them in plain text.

To build and push on our dockerhub repo, we add a `job` to our `main.yml`:

```yml
# define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP1/simple-api-student
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/simple-api-student:latest
          push: ${{ github.ref == 'refs/heads/master' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP1/database
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/database:latest
          push: ${{ github.ref == 'refs/heads/master' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./TP1/http-server
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/http-server:latest
          push: ${{ github.ref == 'refs/heads/master' }}
```

❓ Why did we put needs: build-and-test-backend on this job?  🇺🇸

💡 There's an order to this, because to build our docker image, our application must first be compiled.

❓ For what purpose do we need to push docker images? 🇺🇸

💡 By pushing docker images, they become retrievable from any docker. This can be very useful for easy & fast deployment.

### Setup Quality Gate

**Sonar** : Code monitoring platform for analysis purposes.

After configuring sonar by linking it to the GitHub repo, we add sonar to our `main.yml` file in the maven command line with sonar credentials: 

```yml
- name: Build and test with Maven
  working-directory: TP1/simple-api-student
  run: mvn -B clean verify sonar:sonar -Dsonar.projectKey=remitranvanba_DevOps-module -Dsonar.organization=remitranvanba -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}
```

❓ Document your quality gate configuration. 🇺🇸

💡 We use `sonar way`, the default quality gate. The code passes the quality test if :
- No new bugs
- No new vulnerabilities
- New code has limited technical debt
- All new security hotspots covered
- Enough testing to cover new code (**> 80%**)
- Little duplication in new code (**< 3%**) 

## TP 03 - Ansible 🛸🛸🛸

Install ansible locally and create an inventory for it in a `setup.yml` file:
```
mkdir ansible
cd ansible
mkdir inventories
cd inventories
touch setup.yml
```

setup.yml:
```yml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /home/remi/Downloads/id_rsa
 children:
   prod:
     hosts: remi.tran-van-ba.takima.cloud
```
We can now ping our server:
```
sudo ansible all -i ansible/inventories/setup.yml -m ping
```
Response:
```
remi.tran-van-ba.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
Check conf:
```
sudo ansible all -i ansible/inventories/setup.yml -m setup -a  "filter=ansible_distribution*"
```
Response:
```
remi.tran-van-ba.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "7",
        "ansible_distribution_release": "Core",
        "ansible_distribution_version": "7.9",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```

If you had httpd installed on the ansible server, you can uninstall it with the following command:
```
sudo ansible all -i ansible/inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```

❓ Document your inventory and base commands 🇺🇸

💡 Inventory:
- We provide in variable the path to our ssh key, and the user.
- We provide the address of our server

### Playbooks

playbook.yml:
```yml
- hosts: all
  gather_facts: false
  become: true

  tasks:
   - name: Test connection
     ping:
```
By running the playbook, you can execute the task assigned to it:
```
ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml
```

### Advanced Playbooks
Several tasks can be assigned, here we install docker and python3 in their respective order:
```yml
# Install Docker
- hosts: all
  gather_facts: false
  become: true
  tasks:
    - name: Install device-mapper-persistent-data
      yum:
        name: device-mapper-persistent-data
        state: latest

    - name: Install lvm2
      yum:
        name: lvm2
        state: latest

    - name: add repo docker
      command:
        cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

    - name: Install Docker
      yum:
        name: docker-ce
        state: present

    - name: Install python3
      yum:
        name: python3
        state: present

    - name: Install docker with Python 3
      pip:
        name: docker
        executable: pip3
      vars:
        ansible_python_interpreter: /usr/bin/python3

    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker
```

### Using Roles 

Create a role:
```
ansible-galaxy init roles/<ROLE_NAME>
```

We're going to create roles to configure and launch the project we published earlier on dockerhub. Each role enables specific tasks to be carried out from `playbook.yml`:
```yml
roles:
    - docker
    - network
    - database
    - app
    - proxy
```

### Deploy your App

Once the specified roles have been created, each one must perform a task. The execution order of the roles is important for the compilation of each docker container we retrieve, so care must be taken. Following on from our github secrets, we need to add environment variables for **urls**, **container names**, **network**, and **tokens**. To do this, we create a `group_vars\var.yml` file:
```
API_CONTAINER: "springboot-app-container"

POSTGRES_CONTAINER: "postgres-db"
POSTGRES_DB: "db"
POSTGRES_USER: "usr"
POSTGRES_PASSWORD: "pwd"

NETWORK: "app-network"
```
List of tasks for each role:
- docker
```yml
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Install python3
  yum:
    name: python3
    state: present

- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```
- network:
```yml
- name: Create a network
  community.docker.docker_network:
    name: "{{ NETWORK }}" 
  vars:
    ansible_python_interpreter: /usr/bin/python3
```
- database:
```yml
- name: Run Postgres
  docker_container:
    name: "{{ POSTGRES_CONTAINER }}" 
    image: remitranvanba/database:latest
    networks:
      - name: "{{ NETWORK }}" 
    env:
      POSTGRES_DB: "{{ POSTGRES_DB }}" 
      POSTGRES_USER: "{{ POSTGRES_USER }}" 
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}" 
    volumes:
      - myvolume:/var/lib/postgresql/data
    recreate: true
    pull: true
  vars:
    ansible_python_interpreter: /usr/bin/python3
```
- app:
```yml
- name: Run Backend API
  docker_container:
    name: "{{ API_CONTAINER }}" 
    image: remitranvanba/simple-api-student:latest
    networks:
      - name: "{{ NETWORK }}" 
    env:
      URL: "{{ POSTGRES_CONTAINER }}:5432"
      POSTGRES_DB: "{{ POSTGRES_DB }}" 
      POSTGRES_USER: "{{ POSTGRES_USER }}" 
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
    recreate: true
    pull: true
  vars:
    ansible_python_interpreter: /usr/bin/python3
```
- proxy:
```yml
- name: Run HTTPD
  docker_container:
    name: my-apache-server
    image: remitranvanba/http-server:latest
    networks:
      - name: "{{ NETWORK }}" 
    ports:
      - 80:80
    recreate: true
    pull: true
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

❗ Note that for some tasks, we had to specify ansible's python interpreter in version 3.

❓ Document your docker_container tasks configuration. 🇺🇸

- 💡 docker : Install all docker dependencies using yum.
- 💡 network : Create a docker network using the name contained in the `env.yml` and specifying the python version.
- 💡 database : Specify the python version. Retrieve the docker image of our database from dockerhub, and specify its network. Set the superuser log as an environment variable. Specify its volume location to maintain data persistence. Specify that the image will always be rebuilt & pulled from the docker repo.
- 💡 app: Same as above, we retrieve the docker image from dockerhub and instantiate it with its environment variables.
- 💡 proxy: Same as above, except that here we also specify its exposed port. 

## Authors

- [@Remi Tran Van Ba](https://www.github.com/remitranvanba)