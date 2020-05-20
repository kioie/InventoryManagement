# Spring GCP: Inventory Management Rest Service 
**About**  
  
This is a demo app built on Spring, to test out a CI/CD deployment plan on Google Cloud Platform. This demo is designed with the goal of integrating GCP services to ease deployment on a GCP Cloud build pipeline.

The project integrates Spring boot, Hibernate, GCP Cloud Sql, GCP Secret Manager, GCP App Engine, GCP Cloud Build and an option for a container deployment.

## Overview
This demo is built with Spring Cloud, Spring GCP and Maven. The app is a simple Inventory Manager API endpoint and should be accessible on 

    curl https://{url}/inventory/1
    curl https://{url}/inventory/2
    curl https://{url}/inventory/
The app will bootstrap a database using Spring Hibernate and should access the data over the rest API endpoint.

## Bill of Materials
**Dependencies**
| Dependency                             | Version |
|----------------------------------------|---------|
| spring-boot-starter-parent             | 2.2.6   |
| Java                                   | 1.8     |
| spring-boot-starter-data-rest          | 1.2.3   |
| spring-boot-starter-data-jpa           | 1.2.3   |
| mysql-connector-java                   | 1.2.3   |
| spring-cloud-gcp-starter               | 1.2.3   |
| spring-cloud-gcp-starter-sql-mysql     | 1.2.3   |
| spring-cloud-gcp-starter-secretmanager | 1.2.3   |

**Plugins**
| Plugin                   | Version |
|--------------------------|---------|
| spring-boot-maven-plugin |         |
| maven-dependency-plugin  | 2.0     |
| appengine-maven-plugin   | 2.2.0   |

## Services Breakdown
**Spring Hibernate with GCP Cloud Sql DataSource**

Lets start with defining the data source and configs

Basic configs for spring hibernate are defined under the [`application.properties`](https://github.com/kioie/InventoryManagement/blob/master/src/main/resources/application.properties) file. This file needs no further configuration.

    spring.jpa.hibernate.ddl-auto=none  
    spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl  
    spring.jpa.show-sql=true  
    spring.datasource.initialization-mode=always  
    spring.datasource.hikari.maximum-pool-size=1  
    management.contextPath=/_ah  
    spring.profiles.active=mysql
  
  
**Note**:
*`spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl`* Hibernate maps field names using a physical strategy and an implicit strategy. We want to use physical naming strategy just so the values we use for annotations `@Table` and `@Column’s` name attribute we use in the [inventory](https://github.com/kioie/InventoryManagement/blob/master/src/main/java/com/gcp/springboot/inventorymanagement/model/Inventory.java) model file among other files would remain as it is.

Linking it to GCP Cloud Sql as our MySql datasource we begin by adding this dependency

    <dependency>  
        <groupId>org.springframework.cloud</groupId>  
        <artifactId>spring-cloud-gcp-starter-sql-mysql</artifactId>  
    </dependency>

And then add another config file for cloud sql: [application-mysql.propeties](https://github.com/kioie/InventoryManagement/blob/master/src/main/resources/application-mysql.properties). Make sure that your GCP Cloud Sql service is setup and your secrets have been configured and ready for use with the correct permissions. You'll need to setup an instance-connection, an instance, a database and create credentials for your database. As you can see below, I'll be using a `root` login

    ##CLOUD-SQL-CONFIGURATIONS  
    spring.cloud.appId=sample-gcp-project  
    spring.cloud.gcp.sql.instance-connection-name=sample-gcp-project-xxxxx:us-central1:test-inventory-db  
    spring.cloud.gcp.sql.database-name=inventory
    #SQL DB USERNAME/PASSWORD  
    spring.datasource.username=root
    spring.datasource.password=xxxxx
Now, since we are hosting on a public repo, we'd need to take care of confidential configs (usernames, passwords, connection names etc).

So lets add [GCP Secret Manager](https://cloud.google.com/secret-manager) service for our credential manager.

Start by adding the `spring-cloud-gcp-starter-secretmanager` dependency

    <dependency>  
        <groupId>org.springframework.cloud</groupId>  
        <artifactId>spring-cloud-gcp-starter-secretmanager</artifactId>  
    </dependency>
And then configure the secrets on the [application-mysql.propeties](https://github.com/kioie/InventoryManagement/blob/master/src/main/resources/application-mysql.properties) . Make sure that you've setup your GCP Secret Manager service is setup and you secrets have been configured and ready for use with the correct permissions.

    ##CLOUD-SQL-CONFIGURATIONS  
    spring.cloud.appId=sample-gcp-project  
    spring.cloud.gcp.sql.instance-connection-name=${sm://projects/647366740951/secrets/spring_cloud_gcp_sql_instance_connection_name}  
    spring.cloud.gcp.sql.database-name=${sm://projects/647366740951/secrets/spring_cloud_gcp_sql_database_name}  
    #SQL DB USERNAME/PASSWORD  
    spring.datasource.username=${sm://projects/647366740951/secrets/spring_datasource_username}  
    spring.datasource.password=${sm://projects/647366740951/secrets/spring_datasource_password}

We will now bootstrap a simple database using a [data.sql](https://github.com/kioie/InventoryManagement/blob/master/src/main/resources/data.sql) and [schema.sql](https://github.com/kioie/InventoryManagement/blob/master/src/main/resources/schema.sql) file.


At this point, you should be able to run the app locally with a simple 

    mvn spring-boot:run

## **Spring GCP App Engine**

Now lets take this app to the "cloud". Well start by adding the app engine plugin in your `pom.xml`

        <plugin>  
            <groupId>com.google.cloud.tools</groupId>  
            <artifactId>appengine-maven-plugin</artifactId>  
            <version>2.2.0</version>  
            <configuration>  
                <deploy.projectId>sample-gcp-project-xxxx</deploy.projectId>  
                <deploy.version>1</deploy.version>  
            </configuration>  
        </plugin>

**My plugin file includes the `deploy.projectId` and the `deploy.version`, both of which you will need to reconfigure with your own values, especially the `deploy.projectId.`**

We'll start by configuring [app.yaml](https://github.com/kioie/InventoryManagement/blob/master/src/main/appengine/app.yaml) on `src/main/appengine/app.yaml`. Again, you will need to configure permissions and kickstart an app on appengine for this to work

    env: flex  
    runtime: java  
    runtime_config:  
      jdk: openjdk8  
      
    resources:  
      cpu: 1  
      memory_gb: 1  
      disk_size_gb: 10  
      volumes:  
        - name: ramdisk1  
          volume_type: tmpfs  
          size_gb: 0.5  
      
      
    handlers:  
      - url: /.*  
        script: this field is required, but ignored

This file is highly configurable and you can tweak it to adjust performance, however, I've kept it as simple as possible to simplify the deployment plan. If you plan to keep a simple deployment, this file should suffice as it is.

At this point, you should be able to directly deploy with 

    mvn -DskipTests=true appengine:deploy

## **GCP Cloud Build**

Moving on to Cloud build, you will need the [cloudbuild.yaml](https://github.com/kioie/InventoryManagement/blob/master/cloudbuild.yaml) file on the project root. Your cloud build process is broken down into two steps, test step and package and deploy step.

    steps:  
      - name: maven:3-jdk-8  
        entrypoint: mvn  
        args: ["test"]  
      - name: maven:3-jdk-8  
        entrypoint: mvn  
        args: ["package", "-Dmaven.test.skip=true","appengine:deploy"]
At this point, you should be able to push this spring app to a github repo and trigger a cloud build on GCP.

## Bonus

If you are interested, you can include a container build in your CI/CD pipeline. All you'll need to do is to setup a simple [Dockerfile](https://github.com/kioie/InventoryManagement/blob/master/Dockerfile)

    FROM openjdk:8-jdk-alpine  
    ARG JAR_FILE=target/inventorymanagement*.jar  
    COPY ${JAR_FILE} app.jar  
    ENTRYPOINT ["java", "-Djava.security.edg=file:/dev/./urandom","-jar","/app.jar"]

Then add a container generation step to your deployment on cloudbuild.yaml.
The final file would look like so:

    steps: 
      - name: maven:3-jdk-8  
        entrypoint: mvn  
        args: ["test"]  
      - name: maven:3-jdk-8  
        entrypoint: mvn  
        args: ["package", "-Dmaven.test.skip=true","appengine:deploy"]  
      - name: gcr.io/cloud-builders/docker  
        args: ["build", "-t", "gcr.io/$PROJECT_ID/inventorymanagement", "--build-arg=JAR_FILE=target/inventorymanagement-1.0.0.0.jar", "."]  
    images: ["gcr.io/$PROJECT_ID/inventorymanagement"]
