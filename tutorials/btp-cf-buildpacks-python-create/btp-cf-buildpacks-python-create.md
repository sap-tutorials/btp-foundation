---
parser: v2
author_name: Gergana Tsakova
author_profile: https://github.com/Joysie
title: Create an Application with Cloud Foundry Python Buildpack 
description: Create a simple Python application and enable services for it, by using  the Cloud Foundry Python Buildpack and Cloud Foundry Command Line Interface (cf CLI).
auto_validation: true
time: 45
tags: [ tutorial>beginner, software-product>sap-btp--cloud-foundry-environment, software-product>sap-hana, software-product-function>sap-btp-cockpit]
primary_tag: programming-tool>python
---

## You will learn
  - How to create a simple "Hello World" application in Python
  - How to consume an SAP BTP service from it
  - How to run authentication and authorization checks via the Authorization and Trust Management (XSUAA) service

## Prerequisites
 - You have a [trial](https://help.sap.com/docs/btp/sap-business-technology-platform/trial-accounts-and-free-tier) or an [enterprise](https://help.sap.com/docs/btp/sap-business-technology-platform/enterprise-accounts) (productive) account for SAP Business Technology Platform (SAP BTP). 
 - [Python] (https://www.python.org/downloads/) is installed locally. To check which Python versions are supported in the current buildpack, see: [Developing Python in the Cloud Foundry Environment](https://help.sap.com/docs/btp/sap-business-technology-platform/developing-python-in-cloud-foundry-environment#buildpack-versioning). In this tutorial, we use Python version **3.13.x**.
 - [cf CLI] (https://help.sap.com/docs/btp/sap-business-technology-platform/download-and-install-cloud-foundry-command-line-interface) is installed locally.
 - [npm] (https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) is installed locally.
 - You have installed an integrated development environment, for example [Visual Studio Code] (https://code.visualstudio.com/).
 - You have installed the `virtualenv` tool. It creates a folder, which contains all the necessary executables to use the packages that your Python project would need. To install it locally, run the following command in the Python installation path:   
&nbsp;
 `<Python_installation_path>\Python311\Scripts>pip install virtualenv`



## Intro
This tutorial will guide you through creating and setting up a simple Python application by using cf CLI. You will start by building and deploying a web application that returns simple data – a **Hello World!** message. This simple app will consume the SAP HANA Cloud service, and then will be invoked through a web microservice (application router). Finally, you will set authentication and authorization checks to properly access your web application.

---

### Create an SAP HANA Cloud instance

You need to fulfill this prerequisite step first in order to create an SAP HANA Cloud service instance later.

1. Open the SAP BTP cockpit.

2. Navigate to your global account. If you're using a trial SAP BTP account, just go the main screen.

3. From the left-side menu, choose `Boosters`. 

4. Find the **Set Up SAP HANA Cloud Administration Tools** tile and choose `Start`. 
   
    > Your SAP HANA Cloud database is created for you.

5. Now go to your subaccount. If you're using a trial SAP BTP account, choose `trial`.

6. From the left-side menu, choose `Services` -> `Instances and Subscriptions`. 
   
    > The top table shows that you're subscribed for the `SAP HANA Cloud` application with plan `tools`.

7. Choose `Go to Application`. 
   
    > The `SAP HANA Cloud Central` portal is opened. 

8. Now you need to create an SAP HANA Cloud instance. Choose `Create Instance`.

9. Choose `SAP HANA Cloud, SAP HANA Database` and then `Next Step`.

10. Enter an instance name and a password for your `DBADMIN` user. Choose `Next Step`.

11. For the next two screens of the wizard, keep choosing `Next Step` without making any changes.

12. On the `SAP HANA Database Advanced Settings` wizard page:
    
    * Select `Allow all IP addresses`. 
    
    * In the `Instance Mapping` section, choose `Add Mapping`.
    
    * For `Environment Instance ID`, enter your org ID. You can find it on your subaccount (or `trial`) page in the cockpit.

    * Choose `Next Step`. 

14. Skip `Data Lake` and choose `Review and Create`.

14. If everything looks fine to you, choose `Create Instance`.

    Your new SAP HANA Cloud instance is in status `Starting`. Wait until it changes to `Running`.


> **Next Steps**  You can move on with the rest of the tutorial. From now on, you will only need a command-line console and IDE. 


### Log on to SAP BTP


First, you need to connect to the SAP BTP, Cloud Foundry environment with your trial or enterprise (productive) subaccount. Your Cloud Foundry URL depends on the region where the API endpoint belongs to. To find out which one is yours, see:  [Regions and API Endpoints Available for the CF Environment] (https://help.sap.com/docs/btp/sap-business-technology-platform/regions-and-api-endpoints-available-for-cloud-foundry-environment)

In this tutorial, we use `eu20` as an **example**.

1. Open a command-line console.

2. Set the Cloud Foundry API endpoint for your subaccount. Run the following command (using your actual region URL):

    ```Bash/Shell
    cf api https://api.cf.eu20.hana.ondemand.com
    ```
3. Log on to the SAP BTP, Cloud Foundry environment:

    ```Bash/Shell
    cf login
    ```

4. When prompted, enter your user credentials. These are the email and password you have used to register your trial or productive SAP BTP account.
 
    > **IMPORTANT**: If the authentication fails, even though you've entered correct credentials, try [logging in via single sign-on] (https://help.sap.com/products/BTP/65de2977205c403bbc107264b8eccf4b/e1009b4aa486462a8951c4d499ce6d4c.html?version=Cloud).


5. Choose the org name and space where you want to create your application.
    
    > If you're using a trial account, you don't need to choose anything. You can use only one org name, and your default space is `dev`.


#### RESULT

Details about your personal SAP BTP subaccount are displayed (API endpoint, user, organization, space).


### Create a Python application

You're going to create a simple Python application.

1. In your local file system, create a new folder. For example: `python-tutorial`

2. From your Visual Studio Code, open the `python-tutorial` folder.

3. Create a file `manifest.yml` with the following content:

    ```YAML
    ---
    applications:
    - name: myapp
      random-route: true
      path: ./
      memory: 128M
      buildpacks: 
      - python_buildpack
      command: python server.py
    ```

    The `manifest.yml` file represents the configuration describing your application and how it will be deployed to Cloud Foundry.

    > **IMPORTANT**: Make sure you don't have another application with the name `myapp` in your space. If you do, use a different name and adjust the whole tutorial according to it.



4. Specify the Python runtime version that your application will run on. To do that, create a `runtime.txt` file with the following content:

    ```TXT
    python-3.13.x
    ```

5. This application will be a web server utilizing the Flask web framework. To specify Flask as an application dependency, create a `requirements.txt` file with the following content:

    ```TXT
    Flask==2.3.*
    ```

6. Create a `server.py` file with the following application logic:

    ```Python
    import os
    from flask import Flask
    app = Flask(__name__)
    port = int(os.environ.get('PORT', 3000))
    @app.route('/')
    def hello():
        return "Hello World!"
    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=port)
    ```

    This is a simple server, which will return a **Hello World!** message when requested.

7.	Deploy the application on Cloud Foundry. To do that, in the `python-tutorial` directory, run:

    ```Bash/Shell
    cf push
    ```

    > Make sure you always run `cf push` in the directory where the `manifest.yml` file is located! In this case, that's `python-tutorial`.

8.  When the staging and deployment steps are completed, the `myapp` application should be successfully started and its details displayed in the command console.

9.	Open a browser window and enter the generated URL of the application (see `routes`).

    For example:  `https://myapp-grouchy-rabbit.cfapps.eu20.hana.ondemand.com`

#### RESULT

Your Python application is successfully deployed and running on the SAP BTP, Cloud Foundry environment. A **Hello World!** message is displayed in the browser.

 

### Consume SAP BTP services


You have created a service instance for SAP HANA Cloud (see **STEP 1**). Now you're going to make a connection to your SAP HANA database from SAP HANA Schemas & HDI Containers - a service that runs on the SAP BTP, Cloud Foundry environment - and consume this service in your application.

1.	Create a `hana` service instance named `pyhana` with service plan `hdi-shared`. Run:

    ```Bash/Shell
    cf create-service hana hdi-shared pyhana
    ```

2.	Bind this service instance to the application. Add `pyhana` in the `manifest.yml` file so that its content looks like this:

    ```YAML
    ---
    applications:
    - name: myapp
      random-route: true
      path: ./
      memory: 128M
      buildpacks: 
      - python_buildpack
      command: python server.py
      services:
      - pyhana
    ```

3. To consume the service inside the application, you need to read the service settings and credentials from the application. To do that, you need to use the `cfenv` Python module. Add two more lines to the `requirements.txt` file so that its content looks like this:

    ```TXT
    Flask==2.3.*
    cfenv==0.5.3
    hdbcli==2.17.*
    ```


4.	Modify the `server.py` file to include additional lines of code - for reading the service information from the environment, and for executing a query with the `hdbcli` driver. After being requested, the application will now return an SAP HANA Cloud related result. Replace the current content with the following:

    ```Python
    import os
    from flask import Flask
    from cfenv import AppEnv
    from hdbcli import dbapi

    app = Flask(__name__)
    env = AppEnv()

    hana_service = 'hana'
    hana = env.get_service(label=hana_service)

    port = int(os.environ.get('PORT', 3000))
    @app.route('/')
    def hello():
        if hana is None:
            return "Can't connect to HANA service '{}' – check service name?".format(hana_service)
        else:
            conn = dbapi.connect(address=hana.credentials['host'],
                                 port=int(hana.credentials['port']),
                                 user=hana.credentials['user'],
                                 password=hana.credentials['password'],
                                 encrypt='true',
                                 sslTrustStore=hana.credentials['certificate'])

            cursor = conn.cursor()
            cursor.execute("select CURRENT_UTCTIMESTAMP from DUMMY")
            ro = cursor.fetchone()
            cursor.close()
            conn.close()

            return "Current time is: " + str(ro["CURRENT_UTCTIMESTAMP"])

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=port)
    ```

5.	In the `python-tutorial` directory, run:

    ```Bash/Shell
    cf push
    ```

6.	Refresh the URL of the `myapp` application (previously loaded in a browser window).


#### RESULT

The current time is displayed, in the UTC time zone.


### Run an Authentication Check


Authentication in the SAP BTP, Cloud Foundry environment is provided by the Authorization and Trust Management (XSUAA) service. In this example, OAuth 2.0 is used as the authentication mechanism. The simplest way to add authentication is to use the Node.js `@sap/approuter` package. To do that, a separate Node.js microservice will be created, acting as an entry point for the application.

1.	In the `python-tutorial` folder, create an `xs-security.json` file for your application with the following content: 
    
    ```JSON
    {
      "xsappname" : "myapp",
      "tenant-mode" : "dedicated",
      "oauth2-configuration": {
        "redirect-uris": [
            "https://*.cfapps.eu20.hana.ondemand.com/**"
          ]
        }
    }
    ``` 
    > **NOTE:**  Instead of  `eu20`, use the technical key of your actual region. 

2.	Create an `xsuaa` service instance named `pyuaa` with plan `application`. To do that, run:

    ```Bash
    cf create-service xsuaa application pyuaa -c xs-security.json
    ```

3.	Add the `pyuaa` service in the `manifest.yml` file so that its content looks like this:

    ```YAML
    ---
    applications:
    - name: myapp
      random-route: true
      path: ./
      memory: 128M
      buildpacks: 
      - python_buildpack
      command: python server.py
      services:
      - pyhana
      - pyuaa
    ```

    The `pyuaa` service instance will be bound to the `myapp` application during deployment.

4.	Now you need to create a microservice (the application router). To do that, go to the `python-tutorial` folder and create a subfolder named  `web`.

    > **IMPORTANT**: Make sure you don't have another application with the name `web` in your space! If you do, use a different name and adjust the rest of the tutorial according to it.


5.	In the `web` folder, create a subfolder `resources`. This folder will provide the business application's static resources.

6.	In the `resources` folder, create an `index.html` file with the following content:

    ```HTML
    <html>
    <head>
    	<title>Python Tutorial</title>
    </head>
    <body>
      <h1>Python Tutorial</h1>
      <a href="/myapp/">My Python Application</a>
    </body>
    </html>
    ```

    This will be the `myapp` application start page.

7.	In the `web` directory, run:

    ```Bash/Shell
    npm init
    ```

    Press **Enter** on every step. This process will walk you through creating a `package.json` file in the `web` folder. 

8.	Now you need to create a directory `web/node_modules/@sap` and install an `approuter` package in it. To do that, in the `web` directory run:

    ```Bash/Shell
    npm install @sap/approuter --save
    ```

9.	In the `web` folder, open the `package.json` file and replace the **scripts** section with the following:

    ```JSON
    "scripts": {
        "start": "node node_modules/@sap/approuter/approuter.js"
    },
    ```

10.	Now you need to add the `web` application to your project and bind the XSUAA service instance (`pyuaa`) to it. To do that, insert the following content **at the end** of your `manifest.yml` file.

    ```YAML
    - name: web
      random-route: true
      path: web
      memory: 128M
      env:
        destinations: >
          [
            {
              "name":"myapp",
              "url":"https://myapp-grouchy-rabbit.cfapps.eu20.hana.ondemand.com/",
              "forwardAuthToken": true
            }
          ]
      services:
      - pyuaa
    ```

    > **NOTE**: For the `url` value, enter your **actual** generated URL for the `myapp` application. 

11.	In the `web` folder, create an `xs-app.json` file with the following content:

    ```JSON
    {
      "routes": [
        {
          "source": "^/myapp/(.*)$",
          "target": "$1",
          "destination": "myapp"
        }
      ]
    }
    ```

    With this configuration, the incoming request is forwarded to the `myapp` application, configured as a destination. By default, every route requires OAuth authentication, so the requests to this path will require an authenticated user.

12.	Go to the `python-tutorial` directory and run:

    ```Bash/Shell
    cf push
    ```
    This command will update the `myapp` application and deploy the `web` application.

    > ### What's going on?

    >At this point of the tutorial, the URL of the `web` application will be requested instead of the `myapp` URL. It will then forward the requests to the `myapp` application.


13.	When the staging and deployment steps are completed, the `web` application should be successfully started, and its details displayed in the command console.

14.	Open a new browser tab or window and enter the generated URL of the `web` application.

    For example:   `https://web-unexpected-cheetah.cfapps.eu20.hana.ondemand.com`

15.	Enter the credentials for your SAP BTP user.


> ### TIP:  
> If you experience an authentication issue, go back to your  `xs-security.json` file and double-check if all the data is correct. If you need to fix something, proceed as follows:
> 
>   1. Do the corrections in the file and save your changes.
>   2. Update your `pyuaa` service, by running:   `cf update-service pyuaa -c xs-security.json`
>   3. Deploy your applications again, by running:  `cf push` 



#### RESULT

- A simple application page with title **Python Tutorial** is displayed. When you click the `My Python Application` link, the current time is displayed, in the UTC time zone.
  
- If you directly access the `myapp` URL, it displays the same result - the current time in the UTC time zone. 


### Run an Authorization Check
Authorization in the SAP BTP, Cloud Foundry environment is also provided by the Authorization and Trust Management (XSUAA) service. In the previous example, the `@sap/approuter` package was added to provide a central entry point for the business application and to enable authentication. Now, to extend the example, authorization will be added.

1.	Add the `sap-xssec` security library to the `requirements.txt` file, to place restrictions on the content you serve. The file should look like this:

    ```TXT
    Flask==2.3.*
    cfenv==0.5.3
    hdbcli==2.17.*
    sap-xssec==4.*
    ```


2.	Modify the `server.py` file to use the SAP `xssec` library. Replace the current content with the following code. (For `port`, you can use 3000, 8000, or 8080).

    ```Python
    import os
    from flask import Flask
    from cfenv import AppEnv
    from flask import request
    from flask import abort

    from sap import xssec

    from hdbcli import dbapi

    app = Flask(__name__)
    env = AppEnv()

    port = int(os.environ.get('PORT', 3000))
    hana = env.get_service(label='hana')
    uaa_service = env.get_service(name='pyuaa').credentials

    @app.route('/')
    def hello():
         if 'authorization' not in request.headers:
             abort(403)
         access_token = request.headers.get('authorization')[7:]
         security_context = xssec.create_security_context(access_token, uaa_service)
         isAuthorized = security_context.check_scope('openid')
         if not isAuthorized:
             abort(403)

         conn = dbapi.connect(address=hana.credentials['host'],
                               port=int(hana.credentials['port']),
                               user=hana.credentials['user'],
                               password=hana.credentials['password'],
                               encrypt='true',
                               sslTrustStore=hana.credentials['certificate'])

         cursor = conn.cursor()
         cursor.execute("select CURRENT_UTCTIMESTAMP from DUMMY")
         ro = cursor.fetchone()
         cursor.close()
         conn.close()

         return "Current time is: " + str(ro["CURRENT_UTCTIMESTAMP"])

    if __name__ == '__main__':
      app.run(host='0.0.0.0', port=port)
    ```

3.	Go to the `python-tutorial` folder and run:

    ```Bash/Shell
    cf push
    ```

    This command will update both **myapp** and **web**.

4.	Try to access `myapp` again (in a browser) in both ways – directly, and through the `web` application router.



#### RESULT

 - If you try to access it directly, a `403 Forbidden` response is displayed due to lack of permissions (lack of authorization header). This is a correct and expected behavior.

 - If you try to access it through the `web` application router, the current time is displayed (in the UTC time zone) – provided that you have the `openid` scope assigned to your user. Since the OAuth 2.0 client is used, the `openid` scope is assigned to your user by default, the correct authorization header is declared, and thus you are allowed to access the `myapp` application.


> **Tip**: For the new result to take effect immediately, you might need to clear the cache of your browser. Or just open the `web` application URL in a private/incognito browser tab.

    