# Demo Project 1

Install Jenkins on DigitalOcean

## Technologies Used

Jenkins, Docker, DigitalOcean, Linux

## Project Description

- Create an Ubuntu server on DigitalOcean
- Set up and run Jenkins as Docker container
- Initialize Jenkins

### Details of project

- Create new server on Cloud

  A new server was created with a minimum of 2 vCPUs and 4 GB of RAM to host Jenkins. In this case, I used an AWS EC2 instance. Port 8080 was opened in the firewall to allow access to 
  Jenkins via a browser.

- Install docker on server to run Jenkins

  The first step was to install Docker on this new server, and then run the following command:

  ```
  docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
  ```
  
  This command pulls Jenkins from Docker Hub, running on port 8080 and using a volume named jenkins_home. Port 50000 is used for the connection between the master and nodes. With this setup, 
  Jenkins can be accessed in the browser using the public IP of the EC2 instance.

- Initialize Jenkins

  The first screen displayed by Jenkins prompted for an initial password, which is located at: /var/jenkins_home/secrets/initialAdminPassword.

  ![Diagram](./images/jenkins_1st_password.png)

  After entering the password, the recommended plugins were installed, and a new user and password were configured. Jenkins is now ready to be used on this server.

  ![Diagram](./images/jenkins_dashboard.png)

# Demo Project 2

Create a CI Pipeline with Jenkinsfile (Freestyle, Pipeline, Multibranch Pipeline)

## Technologies Used

Jenkins, Docker, Linux, Git, Java, Maven

## Project Description

CI Pipeline for a Java Maven application to build and push to the repository
- Install Build Tools (Maven, Node) in Jenkins
- Make Docker available on Jenkins server
- Create Jenkins credentials for a git repository
- Create different Jenkins job types (Freestyle, Pipeline, Multibranch
  pipeline) for the Java Maven project with Jenkinsfile to:
    a. Connect to the application’s git repository
    b. Build Jar
    c. Build Docker Image
    d. Push to private DockerHub repository

### Details of project

- Install Build Tools
  Maven was installed using Jenkins plugins (version 3.9.2, named maven-3.9). Node was installed via curl: curl -sL https://deb.nodesource.com/setup_20.x -o nodesource_setup.sh

- Make Docker available on Jenkins server
  To enable Docker within the Jenkins server, a volume was mounted to /var/run/docker.sock. The updated docker run command is:
  
  ```
  docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins:lts
  ```
  After this, while inside the container as the root user, Docker was installed by running the following commands:

  ```
  curl https://get.docker.com/ > dockerinstall && chmod 777 dockerinstall && ./dockerinstall
  ```
  This script fetches the latest version of Docker and sets the appropriate permissions for the installation file. Additionally, to ensure Jenkins can access the Docker socket, the following   permission was applied:
  ```
  chmod 666 /var/run/docker.sock
  ```
  
- Create Jenkins credentials for a git repository
  I used a Git repository to store the files for this project. Credentials are required to allow the Jenkins pipeline to access external services such as GitHub, GitLab, and Docker Hub. The    Git and Docker Hub credentials were created through the Jenkins UI.

  From this point, different types of pipelines were created to build the JAR files and Docker images, which were then pushed to Docker Hub. These pipelines will be explained in the sections below.

  ## Freestyle Job

  The freestyle pipeline was created and configured with single tasks. One stage checks the npm version using a shell script, while another stage invokes top-level Maven targets to    verify the Maven version. After running successfully, the pipeline was connected to a GitHub repository to execute a script within a specific branch.

  ![Diagram](./images/freestyle.png)

  ## Complete Pipeline

  In this step, a pipeline was created and configured using a Jenkinsfile located in a branch named jenkins-jobs within the GitHub repository. This pipeline contains four stages:

  - Initialize: A Groovy script is initialized, containing the commands to be used in the following stages.    
  - Build: The Java application is built using mvn package with an installed tool.
  - Docker Image: The Docker image is built using stored credentials to log in to Docker Hub. The image is then pushed to a private Docker Hub repository with versioning set to jma-1.1.
  - Deploy: A stage simulates the deployment of the application by returning a message indicating the deployment.

  ![Diagram](./images/pipeline.png)

  ## Multibranch Pipeline

  In this configuration, the setup is almost identical to the previous pipeline. However, this time the pipeline is configured to match all branches of the repository. Each       
  branch has its own pipeline that runs independently, displaying results in separate consoles. For this run, three GitHub branches were used: main, jenkins-jobs, and 
  feat/payments.
  
  ![Diagram](./images/github.png)

  All branches contains identical Jenkins files, which were recognized and executed by the pipeline. The image stage has pull in docker hub with versioning set to jma-2.0.
  
  ![Diagram](./images/multibranchpipeline.png)

  Additionally, a condition was added to the build and deploy stages so that they only run in the main branch. The other branches correctly skipped these stages.

  ![Diagram](./images/payments.png)

# Demo Project 3

Create a Jenkins Shared Library

## Technologies Used

Jenkins, Groovy, Docker, Git, Java, Maven

## Project Description

Create a Jenkins Shared Library to extract common build logic:
- Create separate Git repository for Jenkins Shared Library project
- Create functions in the JSL to use in the Jenkins pipeline
- Integrate and use the JSL in Jenkins Pipeline (globally and for a specific project in Jenkinsfile)

### Details of project

  - Create separate repository and prepare the functions to be called in jenkinsfile
    
  At the beginning of the project, a separate repository was created on GitHub to host the Jenkins Shared Library. The Groovy scripts are stored inside a vars folder at the root of the 
  repository. The file names must match the function names used in the pipeline. One of the functions created was buildJar.
  
  ```
  #!/user/bin/env groovy

    def call() {
      echo "building the application..."
      sh 'mvn package'
    }
  ```
  The first line of the code ensures that the editor recognizes the Groovy script. A similar function, buildImage, was created in a separate file. Both functions need to be called in the 
  Jenkinsfile.
  
  ```
        stage("build jar") {
            steps {
                script {
                    buildJar()
                }
            }
        }

        stage("build image") {
            steps {
                script {
                    buildImage()

                }
            }
        }
  ```
   To run this pipeline, the Shared Library Repository must be available in Jenkins. This was configured under Manage Jenkins > System > Global Pipeline Libraries, where the repository URL, 
   the main branch for versioning, and the credentials for authentication were added.

   ![Diagram](./images/global-pipeline-library.png)

   This configuration allows Jenkins to use the Shared Library across all pipelines. However, further configuration will be done later to limit its scope. For now, the library must be 
   declared in the Jenkinsfile.
  
   ```
    @Library('jenkins-shared-library')
    def gv
   ```
  - Add variables in functions
    
    The pipeline ran successfully with the previous configurations. Additionally, improvements were made, one of which was adding environment variables to the Shared Library functions. For 
    example, the buildImage function was updated as shown below:
  
     ```
     def call(String imageName) {
        echo "building the docker image..."
        withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
        sh "docker build -t $imageName ."
        sh 'echo $PASS | docker login -u $USER --password-stdin'
        sh "docker push $imageName"
        }
      }
     ```

     A variable called imageName was passed as a parameter to the function, simplifying some Docker commands and avoiding code repetition. The Jenkinsfile contains the string that will be 
     passed to this function:
  
     ```
      stage("build image") {
              steps {
                  script {
                      buildImage 'mauriciocamilo/demo-app:jma-3.0'
                  }
              }
          }
     ```

  - Creating a Class
    
    Another improvement in this project was the creation of a class within the Shared Library. This class is stored in a separate file inside the src folder. It extracts all the commands 
    from the functions and acts as a bridge between the Jenkinsfile and the Shared Library.

     ```
      #!/user/bin/env groovy
      package com.example

      class Docker implements Serializable {

        def script

        Docker(script) {
          this.script = script
        }
      }
     ```
     The class structure includes the package name, the class declaration, and script configuration to enable the execution of commands in the pipeline. The Jenkinsfile still calls the     
     Groovy script functions, but these functions now invoke the class. Here’s an example of a Groovy function:

      ```
      #!/user/bin/env groovy
      
      import com.example.Docker
      def call(String imageName) {
          return new Docker(this).buildDockerImage(imageName)
      }
     ```
    By implementing this configuration, two additional Groovy script functions were created, each executing a Docker command. The Jenkinsfile now calls all of them within a single stage.
    
    ```
      stage("build and push image") {
        steps {
          script {
            buildImage 'mauriciocamilo/demo-app:jma-3.0'
            dockerLogin()
            dockerPush 'mauriciocamilo/demo-app:jma-3.0'
          }
        }
      }
    ```
    The result is a cleaner Jenkinsfile with functions that clearly describe what is executed at each stage, while all the logic is centralized in the class created in the Shared Library. 
    The pipeline also ran successfully with this configuration:
    
    ![Diagram](./images/shared-library-pipeline.png)

  - Configure JSL to be used only for this project

    In this step, the configuration made under Global Pipeline Libraries was removed, and the Jenkinsfile was updated to directly access the Shared Library. This way, only this specific 
    pipeline has access to the Shared Library:
    
    ```
    library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
    [$class: 'GitSCMSource',
    remote: 'https://github.com/Mauricio-Camilo/jenkins-shared-library.git',
    credentialsId: 'github-credentials'])
    ```
    
# Demo Project 4

Configure Webhook to trigger CI Pipeline automatically on every change

## Technologies Used

Jenkins, GitHub, Git, Docker, Java, Maven

## Project Description

- Since I am using GitHub for all my projects, I can achieve the same results for this project using GitHub as well.
- Configure GitHub webhook to access Jenkins pipelines
- Configure Jenkins to trigger the CI pipeline, whenever a change is pushed to GitHub

### Details of project

- Configure Github webhook to access a pipeline

  The first step of this process was to navigate to the GitHub repository, then go to Settings > Webhooks. A new webhook was set up using the Jenkins URL, as shown below:
  
   ```
     http://54.197.166.37:8080/github-webhook/ 
   ```
   ![Diagram](./images/github-webhook-config.png)

  GitHub automatically pings the Jenkins URL to validate the webhook.

   ![Diagram](./images/github-webhook-confirmation.png)

  In Jenkins, under the pipeline configuration dashboard, the Build Triggers section was updated by selecting the GitHub hook trigger for GITScm polling option. This enables the pipeline to receive requests from the GitHub webhook. As a result, when a change is pushed to the repository, the pipeline runs automatically without needing to manually click the Build button.

- Configuration for multibranch pipelines

  By default, multibranch pipelines do not have build triggers enabled. To resolve this, the Multibranch Scan Webhook Trigger plugin was installed. This added a new option called Scan Multibranch Pipeline Triggers in the pipeline settings. The configuration of this section is shown below:
  
   ![Diagram](./images/multibranch-pipeline-trigger.png)

  This trigger token, configured in Jenkins, also needs to be set in the GitHub webhook. This is done by adding a new URL specifically designed to connect with multibranch pipelines.
  
   ```
    http://54.197.166.37:8080/multibranch-webhook-trigger/invoke?token=githubtoken
   ```
  With this configuration in place, any changes made in any branch of the repository will automatically trigger the multibranch pipeline.

# Demo Project 5

Dynamically Increment Application version in Jenkins Pipeline

## Technologies Used

Jenkins, Docker, GitLab, Git, Java, Maven

## Project Description

- Configure CI step: Increment patch version
- Configure CI step: Build Java application and clean old artifacts
- Configure CI step: Build Image with dynamic Docker Image Tag
- Configure CI step: Push Image to private DockerHub repository
- Configure CI step: Commit version update of Jenkins back to Git repository
- Configure Jenkins pipeline to not trigger automatically on CI build commit to avoid commit loop

### Details of project

- Testing maven plugin

  At the beginning of the project, a Maven plugin was used to automatically update the application version. The command to execute this is described below:

  ```
  mvn build-helper:parse-version versions:set -DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextIncrementalVersion} versions:commit
  ```

  It searches for versioning tags inside the pom.xml file. The parsed version creates variables representing the code version, which can be used in the code as parameters. The version follows a three-number format, with only the last number being incremented. Finally, the pom.xml is updated to reflect the new version.The idea was to test this process using a plugin and then implement it in the Jenkinsfile.

- Increment patch version in Jenkinsfile

  When the mvn build command runs, it builds the JAR file using the current version from the pom.xml file. Therefore, the versioning needs to be done before the build. A new stage for version incrementing was added to the Jenkinsfile.

  ```
  stage('increment version') {
              steps {
                  script {
                      echo 'increment app version...'
                      sh 'mvn build-helper:parse-version versions:set \
                      -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                      versions:commit'
                      def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                      def version = matcher[0][1]
                      env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                  }
              }
          }    
  ```
  
  The variable $IMAGE_NAME contains the version tags from the pom.xml, which are captured in an array from the matcher variable, and then the correct value is assigned to the version variable. The build number provided by Jenkins is also used.

  ![Diagram](./images/version-update-log.png)

- Build Image with dynamic Docker Image Tag and push to Docker hub

  This pipeline was tested successfully, but some issues still need to be resolved. Currently, the Dockerfile contains a hardcoded version method, which does not match the version used by Jenkins when building the application. To fix this, some adjustments were made:

  In the Dockerfile, it was updated to handle any version of the JAR file using the command: COPY ./target/java-maven-app-*.jar /usr/app/.
  In the Jenkinsfile, the variable $IMAGE_NAME is used in the docker build and docker push commands.

- Build Java application and clean old artifacts
  
  In the build stage, the command mvn clean package is used to clean the target folder, ensuring that only one JAR file is available for the Dockerfile to read, as it now accepts all versions in the target folder.

  ![Diagram](./images/pipeline-version-complete.png)
 
  It can be seen that pipeline build #6 was executed, and the build number was successfully added to the image pushed to Docker Hub.

  ![Diagram](./images/dockerhub-version-complete.png)

- Commit version update of Jenkins back to Git repository

  Although the Jenkinsfile changes the image version, the updated pom.xml is not being committed to the Git repository. Without this step, the version increment will always remain the same. To address this, a new stage for committing the pom.xml was added after the deploy stage.

  ```
  stage('commit version update') {
              steps {
                  script {
                      withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                          sh 'git config --global user.email "jenkins@example.com"'
                          sh 'git config --global user.name "jenkins"'
                          
                          sh 'git status'
                          sh 'git branch'
                          sh 'git config --list'
  
                          sh "git remote set-url origin https://${USER}:${PASS}@github.com/Mauricio-Camilo/java-maven-app"
                          sh 'git add .'
                          sh 'git commit -m "ci: version bump"'
                          sh 'git push origin HEAD:jenkins-version'
                      }
                  }
              }
          }  
  ```
  All the necessary Git commands to push the updated file to the repository are included. I would like to highlight the Git remote setup, which requires the Git user and password to connect to the repository. The git push command needs to specify origin HEAD to push to the correct branch.

  ![Diagram](./images/pipeline-loop-version.png)

- Configure Jenkins pipeline to not trigger automatically on CI build commit to avoid commit loop

  With these changes, the pom.xml file is committed, completing the version incrementing process automatically. However, this method introduces an infinite loop. This occurs because committing a new file to the repository triggers the pipeline, which in turn triggers another commit. To prevent this loop, Jenkins needs to ignore commits coming from this specific pipeline stage. 
  A new plugin, called Ignore Committer StrategyVersion, was used. With this plugin, the pipeline recognizes when it was triggered by the Jenkins user, preventing it from running automatically. As a result, when the pom.xml file is committed by the user configured in the Jenkinsfile, the pipeline will not rerun automatically.

       



