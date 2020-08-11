# ToDo List Example Application

## Requires
- MySQL Database 8.x (Database Server)
- [WebLogic Server 12.2.1.4.0]([here](https://www.oracle.com/middleware/technologies/weblogic-server-downloads.html)) (Application Server)
- Maven (To build the ToDo application)
- WebLogic Deploy Tooling (WDT) (To convert the WebLogic domain to/from metadata) [GitHub](https://github.com/oracle/weblogic-deploy-tooling/releases)
- WebLogic Image Tool (WIT) to build the Docker image [GitHub](https://github.com/oracle/weblogic-image-tool/releases)

## Setup
Create a Domain that represents an on-premise WebLogic Server domain:
1. [Create a database](#create-a-database-using-mysql-called-tododb) in MySQL called `tododb`.
2. [Create a new WebLogic Server domain](#create-a-new-weblogic-server-domain) 
    - Create a new domain
    - Configure a datasource for the MySQL database
3. [Build and Deploy the To-Do application](#build-and-deploy-the-application)
    - Build `todo.war`
    - Deploy `todo.war` to your WebLogic Server domain
4. [Initialize the DB](#initialized-the-database) 
    - Create the database table for the ToDo application, and initialize the table with a few entries
    - Verify the configuration by using the application to see the ToDo list

Start the lift and shift process:
1. [Create a WDT Model](#create-a-wdt-model)
    - Run WDT's discoverDomain tool to create a model and application archive 
2. [Create a Docker Image](#create-a-docker-image)
    - Run WIT to create a new Docker image with the domain metadata and WebLogic Server binaries
    - Push the Docker image to your repository
3. [Uploading to Verrazzano](#uploading-to-verrazzano)
    - The database is assumed to be reachable from Kubernetes where Verrazzano will run the domain
    - Create the Kubernetes secrets for the domain
    - Create the Kubernetes secret for Verrazzano Docker pull 
    - Apply the model and binding resources for Verrazzano


### Create a database using MySQL called tododb
- Download the MySQL image from Dockerhub 
    ```
    docker pull mysql:latest
    ```
- Start the container database (and optionally mount a volume for data). 
    ```
    docker run --name tododb -p3306:3306 -e MYSQL_USER=derek -e MYSQL_PASSWORD=welcome1 -e MYSQL_DATABASE=tododb -e MYSQL_ROOT_PASSWORD=welcome1 -d mysql:latest
    ```

- Start a MySQL client to change the password algorithm to `mysql_native_password`.
    - Assuming the database server is still running from the previous step, start a database CLI client with:
        ```shell script
        docker exec -it tododb mysql -uroot -p
        ```
    - When prompted for the password, enter the password for the root user, `welcome1` if you used the command above.  
    Once connected, run the `ALTER` command at the MySQL prompt.
        ```mysql
        ALTER USER 'derek'@'%' IDENTIFIED WITH mysql_native_password BY 'welcome1';
        ```

### Create a new WebLogic Server Domain
If you do not have WebLogic Server 12.2.1.4.0 installed, you will need to install it now.  Choose the **GENERIC** installer
from WebLogic Server Downloads, [here](https://www.oracle.com/middleware/technologies/weblogic-server-downloads.html), 
and follow the install instructions documented there. Save the installer after you have finished this install.  You will
need this installer again to build the Docker image.  If you want a smaller image, you can use the **SLIM** installer to
build the image instead of the generic installer.  Since the slim installer does not have a WLS console installed, we 
will not use the slim installer for this step of the example.  To make cut-and-paste of the commands a little easier, 
you should define an environment variable for `ORACLE_HOME` to the folder where you installed WebLogic Server 12.2.1.4.0.

```shell script
export ORACLE_HOME=/install/directory
```

Using the Oracle WebLogic Server config tool, create a new domain called `tododomain`.  Add the password for the admin
user, and accept the defaults for everything else to create a simple domain with a single AdminServer.
Create a WebLogic Server domain using the WebLogic config wizard.
```shell script
$ORACLE_HOME/oracle_common/common/bin/config.sh
```

Start the newly created domain using the domain startWebLogic script.
Access the console of the newly started domain with your browser (e.g., [http://localhost:7001/console](http://localhost:7001/console)).
Using the WebLogic Server console, login and add a datasource configuration to access the MySQL database. 
During the data source configuration, you can accept the defaults for most fields. But, the following fields are 
required to match the application and database settings from above when you started and created the MySQL database:

    - Navigate to Services->Data Sources on the WebLogic console
    - New -> Generic Data Source
    - First page:
        - JDNI Name: jdbc/ToDoDB
        - Database Type: MySQL
    - Fourth page:
        - Database Name: tododb
        - Database Port 3306
        - Database User Name: derek
        - Password: welcome1
        - Confirm Password: welcome1   
     - Click Next and then Finish to complete the configuration.  

### Build and Deploy the Application
Using Maven, build this project to produce the artifact `todo.war`
```shell script
mvn clean package
```

Using the WebLogic Server console, deploy the ToDo application.  Accepting all the default options is fine.  NOTE: The 
remaining steps assume that the application context is `todo`.

**Alternatively**, you can create the domain and deploy the application using WebLogic Deploy Tooling (WDT).  After 
building the project with Maven, use `setup/makeArchive.sh` to create a WDT archive using the application WAR.
```shell script
${WDT_HOME}/bin/createDomain.sh -oracle_home ~/Oracle/Middleware12.2.1.4.0 -domain_parent ./ -model_file ./setup/wdt_domain.yaml -archive_file ./setup/wdt_archive.zip
```

### Initialized the database
After the application is deployed and running in WebLogic Server, access the `http://localhost:7001/todo/rest/items/init` 
REST service to create the database table used by the application. In addition to creating the application table, the 
`init` service will, also, load 4 sample items into the table.

### Access the application
You should be able to access the application at http://localhost:7001/todo/index.html.  Add a few entries or delete some.
After verifying the application and database, you may shutdown the local WebLogic Server domain.

### Create a WDT Model
If you have not already done so, download WebLogic Deploy Tooling (WDT) from [GitHub](https://github.com/oracle/weblogic-deploy-tooling/releases).
Unzip the installer `weblogic-deploy.zip` so that you can access `bin/discoverDomain.sh`. To make cut-and-paste of the 
commands a little easier, you can define an environment variable for `WDT_HOME` to point to the folder where you installed 
WebLogic Deploy Tooling.
```shell script
export WDT_HOME=/install/directory
```

To create a reusable model of the application and domain, use WebLogic Deploy Tooling (WDT) to create a metadata model 
of the domain.  First, create an output directory to hold the generated scripts and models.  Then, run WDT `discoverDomain`.
```shell script
mkdir v8o

${WDT_HOME}/bin/discoverDomain.sh -oracle_home $ORACLE_HOME -domain_home /path/to/domain/dir -model_file ./v8o/wdt-model.yaml -archive_file ./v8o/wdt-archive.zip -target vz -output_dir v8o
```

You should find the following files in `./v8o`:
- **binding.yaml** - Verrazzano binding file
- **model.yaml** - Verrazzano model template
- **wdt-archive.zip** - The WDT archive containing the ToDo Application WAR
- **wdt-model.yaml** - The WDT model of the WebLogic Server domain
- **vz_variable.properties** - A set of properties extracted from the WDT domain model.
- **create_k8s_secrets.sh** - A helper script with `kubectl` commands to apply the Kubernetes secrets needed for this domain.

If you chose to skip the "Access the application" step and did not verify that the ToDo application was deployed, you should
verify that you see the todo.war file inside the wdt-archive.zip.  If you do not see the WAR, there was something wrong 
in your deployment of the application on the WebLogic Server that will require additional troubleshooting in your domain. 

### Create a Docker Image
At this point, the Verrazzano model is still just a template for the real model.  The WebLogic Image Tool (WIT) will
fill in the placeholders for you, or you can edit the model manually to set the image name and domain home directory.

If you have not already done so, download WebLogic Image Tool (WIT) from [GitHub](https://github.com/oracle/weblogic-image-tool/releases).
Unzip the installer `imagetool.zip` so that you can access `bin/imagetool.sh`. To make cut-and-paste of the commands a 
little easier, you can define an environment variable for `WIT_HOME` to the folder where you installed WebLogic Image Tool.
```shell script
export WIT_HOME=/install/directory
```

You will need a Docker image to run your WebLogic Server domain in Kubernetes.  To use the WebLogic Image Tool (WIT) to 
create the Docker image, run `imagetool create`.  While WIT will download patches and PSUs for you, it does not yet
download installers.  Until that point, you must download the WebLogic Server and Java Development Kit installer manually,
and let WIT know where to find them using the `imagetool cache addInstaller` command. 

```shell script
${WIT_HOME}/bin/imagetool.sh cache addInstaller --path /path/to/intaller/jdk-8u231-linux-x64.tar.gz --type jdk --version 8u231 

# The installer filename may be slightly different depending on which version of the 12.2.1.4.0 installer that you downloaded, slim or generic
${WIT_HOME}/bin/imagetool.sh cache addInstaller --path /path/to/intaller/fmw_12.2.1.4.0_wls_Disk1_1of1.zip --type wls --version 12.2.1.4.0

${WIT_HOME}/bin/imagetool.sh cache addInstaller --path /path/to/intaller/weblogic-deploy.zip --type wdt --version latest

# Paths for the files in this command assume that you are running this command from the v8o directory created during the discoverDomain step above
${WIT_HOME}/bin/imagetool.sh create --tag your/repo/todo:1 --version 12.2.1.4.0 --jdkVersion 8u231 --wdtModel ./wdt-model.yaml --wdtArchive ./wdt-archive.zip --wdtVariables ./vz_variable.properties  --vzModel ./model.yaml --wdtModelOnly
```

The `imagetool create` command should have created a local Docker image and updated the Verrazzano model with the domain home
and image name.  Check your Docker images for the tag that you used in the create command using `docker images` from the Docker 
CLI.  If everything worked correctly, it is time to push that image to the repository that Verrrazzano will use to access
the image from Kubernetes.
**NOTE:** The image name must be the same as what is in the Verrazzano model.yaml under 
`spec->weblogicDomains->domainCRValues->image`.

```shell script
docker push your/repo/todo:1
```

### Uploading to Verrazzano
The following steps assume that you have a Kubernets cluster and that Verrazzano is already deployed.

If you haven't already, edit and run the `create_k8s_secrets.sh` to generate the Kubernetes secrets. 
WDT does not discover passwords from your existing domain.  Before running the create secrets script, you will need to 
edit `create_k8s_secrets.sh` to set the passwords for the WebLogic Server domain and the Datasource.  In this domain, 
there are only three passwords that you need to enter: administrator credentials (like weblogic/welcome1), domain credential,
and the ToDo database credential (like derek/welcome1).

For example:
```shell script
# Update <admin-user> and <admin-password> for weblogic-credentials
create_paired_k8s_secret weblogic-credentials weblogic welcome1

# Update <password> for securityconfig
create_k8s_secret securityconfig somedomainpassword

# Update <user> and <password> for jdbc-todo-datasource
create_paired_k8s_secret jdbc-todo-datasource derek welcome1
```

Verrazzano will need a credential to pull the image that you just created. So, you need to create one more secret.
The name for this credential can be changed in `model.yaml` to anything you like, but is defaulted to `ocir`.  
Assuming that you left the name as `ocir`, you will need to run a `kubectl create secret` similar to the follow:
```shell script
kubectl create secret docker-registry ocir --docker-server=phx.ocir.io --docker-email=your.name@company.com --docker-username=tenancy/username --docker-password='passwordForUsername'
```

And finally, run `kubectl apply` to upload the Verrazzano binding and model files to let Verrazzano know to start your pod.
```shell script
kubectl apply -f model.yaml
kubectl apply -f binding.yaml
```

## Copyright
Copyright (c) 2020, Oracle and/or its affiliates.