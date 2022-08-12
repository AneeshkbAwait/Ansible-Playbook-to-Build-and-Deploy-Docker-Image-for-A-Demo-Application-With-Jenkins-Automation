## Ansible-Playbook-to-Build-and-Deploy-Docker-Image-for-A-Demo-Application
This is a Playbook to automate the build and deployment of a docker based application.

Here the Playbook has 2 plays; 

The First play will build docker images based on the application dockerfile available at GitHub and then pushes it to the DockerHub. Whenever there is a change in the application version/any updation, this play will build and push docker images based on the changes applied on it's next manual run. 

The Second play will always fetch the latest docker Image of the application and will recreate the application containers whenever a new docker image is pushed to the DockerHub.
