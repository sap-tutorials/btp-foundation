---
parser: v2
author_name: Gergana Tsakova
author_profile: https://github.com/Joysie
title: Create an Application with SAP Java Buildpack 2
description: Create a simple Spring Boot application and enable services for it, by using SAP Java Buildpack 2 and Cloud Foundry Command Line Interface (cf CLI).
auto_validation: true
time: 45
tags: [ tutorial>beginner, software-product>sap-btp--cloud-foundry-environment, software-product-function>sap-btp-cockpit]
primary_tag: programming-tool>java
---


## You will learn
  - How to create a Spring Boot Java project
  - How to create and deploy a simple "Hello World" application
  - How to run authentication and authorization checks via the Authorization and Trust Management (XSUAA) service
 

## Prerequisites
 - You have a trial or productive account for SAP Business Technology Platform (SAP BTP). If you don't have such yet, you can create one so you can [try out services for free] (https://developers.sap.com/tutorials/btp-free-tier-account.html).
 - You have created a subaccount and a space on Cloud Foundry Environment.
 - [cf CLI] (https://help.sap.com/products/BTP/65de2977205c403bbc107264b8eccf4b/4ef907afb1254e8286882a2bdef0edf4.html) is installed locally.
 - You have downloaded [`JDK for SapMachine 17`] (https://sap.github.io/SapMachine/) and [installed] (https://github.com/SAP/SapMachine/wiki/Installation) it locally, configuring your `PATH` and `JAVA_HOME` environment variables.
 - You have [Apache Maven] (https://maven.apache.org/download.cgi) downloaded. To do that, go to **Files** and choose the `Binary zip archive` link. For this tutorial, we use version `3.9.8`.
 - [Install Maven] (https://maven.apache.org/install.html) - similar to JDK, configure your `PATH` and `MAVEN_HOME` variables.
 - You have installed an integrated development environment. In this tutorial, we use [Visual Studio Code] (https://code.visualstudio.com/) but you can use a different one (Eclipse IDE or IntelliJ IDEA).


## Intro
This tutorial will guide you through creating and setting up a simple Java application by using cf CLI. You will start by creating a Java project via Spring Boot, and then creating a web application that returns simple data – a **Hello World!** message. This simple app will be invoked through a web microservice (application router). Finally, you will set authentication checks and an authorization role to properly access your web application.

---


### Log on to SAP BTP


First, you need to connect to the SAP BTP, Cloud Foundry environment with your trial or enterprise (productive) subaccount. Your Cloud Foundry URL depends on the region where the API endpoint belongs to. To find out which one is yours, see:  [Regions and API Endpoints Available for the CF Environment] (https://help.sap.com/products/BTP/65de2977205c403bbc107264b8eccf4b/f344a57233d34199b2123b9620d0bb41.html?version=Cloud)

In this tutorial, we use `eu20.hana.ondemand.com` as an **example**.

1. Open a command-line console.

2. Set the Cloud Foundry API endpoint for your subaccount. Run the following command  (using your **actual** region URL):

    ```Bash/Shell
    cf api https://api.cf.eu20.hana.ondemand.com
    ```
3. Log on to the SAP BTP, Cloud Foundry environment:

    ```Bash/Shell
    cf login
    ```

4. When prompted, enter your user credentials – the email and password you have used to register your productive SAP BTP account.

    > **IMPORTANT**: If the authentication fails, even though you've entered correct credentials, try [logging in via single sign-on] (https://help.sap.com/products/BTP/65de2977205c403bbc107264b8eccf4b/e1009b4aa486462a8951c4d499ce6d4c.html?version=Cloud).

5. Choose the org name and space where you want to create your application.
    
    > If you're using a trial account, you don't need to choose anything. You can use only one org name, and your default space is `dev`.

#### RESULT

Details about your personal SAP BTP subaccount are displayed (API endpoint, user, organization, space).




### Create a Java Project

Before creating an application, you need a Java project. For this tutorial, you can easily create one by using Spring Boot.

1. Open: `https://start.spring.io`

2. From the configuration screen, choose `Maven Project`, language `Java`, and Spring Boot version `3.3.1 `.

3. From `Project Metadata` section, you need to do the following settings:

    - **Group**: `com.example`

    - **Artifact**: `java-tutorial`

    - **Name**: `HelloWorld`

    - **Description**: `A simple HelloWorld Java project`

    - **Package name**: `com.example.java-tutorial`

    - **Packaging**: `Jar`

    - **Java**: `17`

4. Choose `Add Dependencies` and then select `Spring Web`.

5. Choose `Generate`.

6. A `java-tutorial.zip` file is generated. Save it on your local file system and then extract the `java-tutorial` folder.



#### RESULT

You have successfully created a basic Java project.




### Complete your Java Project


For this part, you need to configure your `HelloWorld` application, add an extra class, and a `manifest.yml`  file.

1. Open `java-tutorial` directory in a console client, and run:

    ```Bash/Shell
    mvn install
    ```

    This command builds your Java project (as a Maven one).

2. From your Visual Studio Code, open the `java-tutorial` folder and create a file `manifest.yml` with the following content (using your **actual** path):

    ```YAML
    ---
    applications:
    - name: helloworld
      random-route: true
      path: ./target/java-tutorial-0.0.1-SNAPSHOT.jar
      memory: 1024M
      buildpacks: 
      - sap_java_buildpack_jakarta
      env:
        TARGET_RUNTIME: tomcat
        JBP_CONFIG_COMPONENTS: "jres: ['com.sap.xs.java.buildpack.jdk.SAPMachineJDK']"
    ```

3. Navigate to `src\main\java\com.example.javatutorial` and open the `HelloWorldApplication.java` file.

4. In the `public static void main` class, add the following line: 	`System.out.println("Hello World!");`

    Your final Java code should look like this:

    ```Java

    package com.example.java_tutorial;

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;

    @SpringBootApplication
    public class HelloWorldApplication {

    	public static void main(String[] args) {
    		SpringApplication.run(HelloWorldApplication.class, args);
    		System.out.println("Hello World!");
    	}

    }
    ```

5. Navigate to `src/main/java` and go to the `com.example.java_tutorial` package.

6. From its context menu, choose `Java File` > `Class`.

7.  Enter `MainController` and press the `ENTER` key. The new class appears in the project navigation.

8.  Replace its default content with the following code:

    ```Java

    package com.example.java_tutorial;

    import org.springframework.http.HttpStatus;
    import org.springframework.http.ResponseEntity;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    @RequestMapping(path = "")

    public class MainController {
       @GetMapping(path = "")
       public ResponseEntity<String> getDroneMedications() {
          return new ResponseEntity<String>("Hello World!", HttpStatus.OK);
       }

    }
    ```

#### RESULT

Your Java project is complete and your application is ready to be deployed.





### Deploy your Java application


1. Test your project locally first. To do that, in the Visual Studio Code, right-click on `HelloworldApplication.java` and choose `Run Java`.

    The final result displayed in the `Terminal` tab should be: **Hello World!**  

    > Same result will be displayed in a browser if you enter: `localhost:8080`

2. Now go to the `java-tutorial` directory from the command console, and build your project again by running:

    ```Bash/Shell
    mvn clean install
    ```

3. Then run:

    ```Bash/Shell
    cf push helloworld
    ```

    This command deploys your Java application.

    > Make sure you always run `cf push` in the folder where the `manifest.yml` file is located! In this case, that's `java-tutorial`.

4. When the staging and deployment steps are completed, the `helloworld` application should be successfully started and its details displayed in the command console.

5. Now open a browser window and enter the generated URL of the `helloworld` application (see `routes`).

  For example:  `https://helloworld-noway-panda.cfapps.eu20.hana.ondemand.com`

#### RESULT

Your Java application is successfully deployed and running on the SAP BTP, Cloud Foundry environment. A **Hello World!** message is displayed in the browser.





### Run an Authentication Check


Authentication in the SAP BTP, Cloud Foundry environment is provided by the Authorization and Trust Management (XSUAA) service. In this example, OAuth 2.0 is used as the authentication mechanism. The simplest way to add authentication is to use the Node.js `@sap/approuter` package. To do that, a separate Node.js micro-service will be created, acting as an entry point for the application.

1. In the `java-tutorial` folder, create an `xs-security.json` file for your application with the following content:
    
    ```JSON
    {
      "xsappname" : "helloworld",
      "tenant-mode" : "dedicated",
      "oauth2-configuration": {
        "redirect-uris": [
            "https://*.cfapps.eu20.hana.ondemand.com/**"
          ]
        }
    }
    ```

2. Create an `xsuaa` service instance named `javauaa` with plan `application`. To do that, run:

    ```Bash/Shell
    cf create-service xsuaa application javauaa -c xs-security.json
    ```

3. Add the `javauaa` service in `manifest.yml` so the file looks like this:

    ```YAML
    ---
    applications:
    - name: helloworld
      random-route: true
      path: ./target/java-tutorial-0.0.1-SNAPSHOT.jar
      memory: 1024M
      buildpacks: 
      - sap_java_buildpack_jakarta
      env:
        TARGET_RUNTIME: tomcat
        JBP_CONFIG_COMPONENTS: "jres: ['com.sap.xs.java.buildpack.jdk.SAPMachineJDK']"
      services:
      - javauaa
    ```

    The `javauaa` service instance will be bound to the `helloworld` application during deployment.

4. Now you have to create a microservice (the application router). To do that, in the `java-tutorial` folder create a subfolder `web`.

    > **IMPORTANT**: Make sure you don't have another application with the name `web` in your space! If you do, use a different name and adjust the rest of the tutorial according to it.

5. In the `web` folder, create a subfolder `resources`. This folder will provide the business application's static resources.

6. In the `resources` folder, create an `index.html` file with the following content:

    ```HTML
    <html>
    <head>
      <title>Java Tutorial</title>
    </head>
    <body>
      <h1>Java Tutorial</h1>
      <a href="/helloworld/">My Java Application</a>
    </body>
    </html>
    ```

    This will be the start page of the `helloworld` application.

7. In the `web` directory, run:

    ```Bash/Shell
    npm init
    ```

    Press **Enter** on every step. This process will walk you through creating a `package.json` file in the `web` folder. 

8. Now you need to create a directory `web/node_modules/@sap` and install an `approuter` package in it. To do that, in the `web` directory run:

    ```Bash/Shell
    npm install @sap/approuter --save
    ```

9. In the `web` folder, open the `package.json` file and replace the **scripts** section with the following:

    ```JSON
    "scripts": {
        "start": "node node_modules/@sap/approuter/approuter.js"
    },
    ```

10. Now you need to add the `web` application to your project and bind the XSUAA service instance (`javauaa`) to it. To do that, insert the following content **at the end** of your `manifest.yml` file.


    ```YAML
    - name: web
      random-route: true
      path: web
      memory: 1024M
      env:
        destinations: >
          [
            {
              "name":"helloworld",
              "url":"https://helloworld-noway-panda.cfapps.eu20.hana.ondemand.com/",
              "forwardAuthToken": true
            }
          ]
      services:
      - javauaa
    ```

    > For the `url` value, enter **your** generated URL for the `myapp` application

11. In the `web` folder, create an `xs-app.json` file with the following content:

    ```JSON
    {
      "routes": [
        {
          "source": "^/helloworld/(.*)$",
          "target": "$1",
          "destination": "helloworld"
        }
      ]
    }
    ```

    With this configuration, the incoming request is forwarded to the `helloworld` application, configured as a destination. By default, every route requires OAuth authentication, so the requests to this path will require an authenticated user.


12. Open your `pom.xml` file and replace the entire `<dependencies>` block with the following:

    ```XML
    <dependencies>
        <!-- Spring Boot starter packages -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cloud-connectors</artifactId>
            <version>2.2.13.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring and XSUAA Security -->
        <dependency>
            <groupId>com.sap.cloud.security.xsuaa</groupId>
            <artifactId>xsuaa-spring-boot-starter</artifactId>
            <version>3.5.0</version>
        </dependency>

        <!-- dependencies for test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    ```


13. Now go to the `java-tutorial` directory and run:

    ```Bash/Shell
    mvn clean install
    ```

14. Then run:

    ```Bash/Shell
    cf push
    ```

    This command will update the `helloworld` application and deploy the `web` application.

    > ### What's going on?

    >As of this point of the tutorial, the URL of the `web` application will be requested instead of the `helloworld` URL. It will then forward the requests to the `helloworld` application.


15. When the staging and deployment steps are completed, the `web` application should be successfully started and its details displayed in the command console.

16. Open a new browser tab or window, and enter the generated URL of the `web` application.

    For example:   `https://web-thankfully-fox.cfapps.eu20.hana.ondemand.com`

17. Enter the credentials for your SAP BTP user and choose the default identity provider.


#### RESULT

- A simple page with title `Java Tutorial` is displayed. When you click the `My Application` link, the output of your `helloworld` application is displayed.

- Check that the `helloworld` application is not directly accessible without authentication. To do that, refresh its previously loaded URL in a web browser – you should get a response `401 Unauthorized`.





### Run an Authorization Check


Authorization in the SAP BTP, Cloud Foundry environment is also provided by the Authorization and Trust Management (XSUAA) service. In the previous example, the `@sap/approuter` package was added to provide a central entry point for the business application and to enable authentication. Now to extend the example, authorization will be added.

1. Navigate to `src/main/java` and go to the `com.example.java_tutorial` package.

2. From its context menu, choose `Java File` > `Class`.

3. Enter `WebSecurityConfig.java` and press the `ENTER` key. The new class appears in the project navigation.

4. Replace its default content with the following code:

    ```Java
    package com.example.java_tutorial;

    import com.sap.cloud.security.xsuaa.XsuaaServiceConfiguration;
    import com.sap.cloud.security.xsuaa.token.TokenAuthenticationConverter;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.core.convert.converter.Converter;
    import org.springframework.security.authentication.AbstractAuthenticationToken;

    import org.springframework.security.config.annotation.web.builders.HttpSecurity;

    import org.springframework.security.config.http.SessionCreationPolicy;
    import org.springframework.security.oauth2.jwt.Jwt;
    import org.springframework.security.web.SecurityFilterChain;

    @Configuration

    public class WebSecurityConfig {

    	@Autowired
    	XsuaaServiceConfiguration xsuaaServiceConfiguration;

    	@Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

            http
            .sessionManagement()
            // session is created by approuter
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
                // demand specific scopes depending on intended request
                .authorizeRequests()

                .requestMatchers("/**").authenticated()
                .anyRequest().denyAll() // deny anything not configured above
            .and()
                .oauth2ResourceServer().jwt()
    				.jwtAuthenticationConverter(getJwtAuthoritiesConverter());

            return http.build();
        }

    	/**
    	 * Customizes how GrantedAuthority are derived from a Jwt
    	 *
    	 * @returns jwt converter
    	 */
    	Converter<Jwt, AbstractAuthenticationToken> getJwtAuthoritiesConverter() {
    		TokenAuthenticationConverter converter = new TokenAuthenticationConverter(xsuaaServiceConfiguration);
    		converter.setLocalScopeAsAuthorities(true);
    		return converter;
    	}

    }
    ```

     > Ignore the yellow warnings (due to recently deprecated methods)!

5. In the same way, create another Java class, named `NotAuthorizedException.java`, and replace its default content with:

    ```Java
    package com.example.java_tutorial;

    import org.springframework.http.HttpStatus;
    import org.springframework.web.bind.annotation.ResponseStatus;

    @SuppressWarnings("serial")
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public class NotAuthorizedException extends RuntimeException {
        public NotAuthorizedException(String message) {
            super(message);
        }
    }
    ```

6. Open the `MainController.java` file and replace its content with the following:

    ```Java
    package com.example.java_tutorial;

    import org.springframework.http.HttpStatus;
    import org.springframework.http.ResponseEntity;
    import org.springframework.security.core.authority.SimpleGrantedAuthority;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    import org.springframework.security.core.annotation.AuthenticationPrincipal;
    import com.sap.cloud.security.xsuaa.token.Token;

    @RestController
    @RequestMapping(path = "")

    public class MainController {

       @GetMapping(path = "")
       public ResponseEntity<String> readAll(@AuthenticationPrincipal Token token) {
           if (!token.getAuthorities().contains(new SimpleGrantedAuthority("Display"))) {
               throw new NotAuthorizedException("This operation requires \"Display\" scope");
           }

           return new ResponseEntity<String>("Hello World!", HttpStatus.OK);
       }
    }
    ```

7. To introduce an application role, open the `xs-security.json` in the `java-tutorial` folder, and add the necessary scope `Display` and role template `Viewer`, as follows:

    ```JSON
    {
        "xsappname" : "helloworld",
        "tenant-mode" : "dedicated",
        "scopes": [
            {
            "name": "$XSAPPNAME.Display",
            "description": "Display content"
            }
        ],
        "role-templates": [
            {
            "name": "Viewer",
            "description": "View content",
            "scope-references": [
                "$XSAPPNAME.Display"
            ]
            }
        ],
        "oauth2-configuration": {
            "redirect-uris": [
                "https://*.cfapps.eu20.hana.ondemand.com/**"
        ]
      }
   }

    ```


8. Update the XSUAA service. To do that, in the `java-tutorial` directory run:

    ```Bash/Shell
    cf update-service javauaa -c xs-security.json
    ```

9. Then build your project again, by running:

    ```Bash/Shell
    mvn clean install
    ```

10. And finally, run:

    ```Bash/Shell
    cf push helloworld
    ```

    This command will redeploy only the `helloworld` application. No changes have been made in `web` so no need to redeploy it.

11. Try to access `helloworld` again (in a browser) in both ways – directly, and through the `web` application router.


#### RESULT

- If you try to access it directly, a `401 Unauthorized` response is still displayed due to lack of authorization token (expected behavior).

- If you try to access it through the app router, it results in a `403 Forbidden` response due to missing permissions. To get these permissions, you need to create a role collection containing the role `Viewer` and assign this role to your user. You can do this only from the SAP BTP cockpit.




### Assigning Roles to a User in SAP BTP Cockpit


1. Open the SAP BTP cockpit and go to your subaccount.

2. From the left-side menu, navigate to `Security` > `Role Collections`.

3. Create a new role collection. For example, `MyJavaAppRC`.

4. Click this role collection and then choose `Edit`.

5. In the `Roles` tab, click the `Role Name` field.

6. Type **Viewer**. From the displayed results, select the `Viewer` role that corresponds to your application, and choose `Add`.

7. Now go to the `Users` tab, and in the `ID` field, enter your e-mail. Then enter the same e-mail in the `E-Mail` field.

8. Save your changes.

    > Your role collection is assigned to your user and contains the role you need to view the content of your application.

    Now you need to apply these changes to the `web` application by building and redeploying it again.

9. Go back to the command line, and in the `java-tutorial` directory, run:

    ```Bash/Shell
    mvn clean install
    ```

10. And finally, run:

    ```Bash/Shell
    cf push web
    ```

#### RESULT

When you try to access again the `helloworld` application through the app router, it will successfully display the **Hello World!** message.

> **Tip**: For the new result to take effect immediately, you might need to clear the cache of your browser. Or just open the `web` application URL in a private/incognito browser tab.

---
