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
  

