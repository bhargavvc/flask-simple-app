Step-by-step explanation of the entire process of building a Docker-Jenkins CI/CD pipeline for a simple Python application. The goal is that even if you’re completely new to DevOps concepts, Jenkins, Docker, or CI/CD pipelines, you should be able to understand what’s going on by the end of this explanation.

---

[Step by Step Guide](#Step-by-Step-Guide)
[Theory](#Notes-CI-CD-Pipeline)


# Step by Step Guide

#### 1. Installing/Updating Java

**Why do we need Java?**  
Jenkins is a Java-based application. It needs Java to run properly. We specifically need at least Java 11.

**How to check if Java is installed?**  
Run:
```bash
java -version
```
- If Java is not installed, the command will say so.
- If it is installed, it will show the version number.

**Install Java 11:**
```bash
sudo apt-get install -y openjdk-11-jre
```
After installation, verify again:
```bash
java -version
```
You should now see something like `openjdk 11.0.x ...` confirming that Java is installed.

#### 2. Installing Git

**What is Git?**  
Git is a tool for source code management. It helps you track changes to your code over time and collaborate with others. We’ll use it to manage our local code and push it to GitHub.

**Check if Git is installed:**
```bash
git --version
```
- If Git is not installed, run:
```bash
sudo apt-get install -y git
```

#### 3. Configuring Git and Creating a Local Repository

**Set up Git defaults:**
```bash
git config --global init.defaultBranch main
git config --global user.name "your_name"
git config --global user.email "your@email.com"
```
- `init.defaultBranch main` sets the default branch to `main` instead of the old `master` naming.
- `user.name` and `user.email` are used by Git to label your commits.

**Create a project folder and initialize a repo:**
```bash
mkdir pythonapp
cd pythonapp
git init
```
This `git init` command creates an empty Git repository inside `pythonapp`.

#### 4. Setting Up a GitHub Repository (Remote Repo)

- Go to GitHub, create a new repository named `pythonapp` (or any name you choose).
- Make sure the repo is Public for simplicity.

**Connecting Local Repo to Remote Repo:**
We want to connect our local code to GitHub. GitHub no longer allows simple password auth for Git, so we’ll use SSH keys.

**Create SSH keys if not already present:**
```bash
ssh-keygen
```
Press Enter a few times to accept defaults. This generates a key pair.

**Get the public key:**
```bash
cat ~/.ssh/id_rsa.pub
```
Copy the output of this command.

**Add this key to GitHub:**
- On GitHub, in your repository settings, go to "Deploy keys".
- Click "Add deploy key", paste your public key, and enable "Allow write access".
  
**Add the remote origin:**
```bash
git remote add origin git@github.com:your_username/pythonapp.git
```
Check:
```bash
git remote
```
You should see `origin`.

**Test the connection:**
```bash
ssh -T git@github.com
```
Type `yes` when prompted. You should see a success message.

Now your local Git repo is connected to your GitHub remote repo via SSH.

#### 5. Creating a Simple Python Application

**Why Python and Flask?**  
We’re using a simple Python app with Flask (a popular Python web framework) because it’s easy to understand. It will display "Hello World!" in the browser.

**Create the application structure:**
```bash
mkdir src
cd src
sudo nano hello_world.py
```

**hello_world.py content:**
```python
from flask import Flask, request
from flask_restful import Resource, Api

app = Flask(__name__)
api = Api(app)

class Greeting (Resource):
    def get(self):
        return 'Hello World!'

api.add_resource(Greeting, '/') # Root route

if __name__ == '__main__':
    app.run('0.0.0.0','3333')
```

This code sets up a simple web server that returns "Hello World!" when you navigate to your server’s IP on port 3333.

#### 6. Installing Jenkins

Jenkins is the automation server that will run the pipeline for us.

**Add Jenkins key and repository:**
```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

Add Jenkins to the sources list:
```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
```

**Install Jenkins:**
```bash
sudo apt-get install -y jenkins
```

Check the Jenkins service:
```bash
sudo systemctl status jenkins.service
```
If it’s running, perfect.

#### 7. Configuring Jenkins

**Access Jenkins via browser:**  
Open `http://localhost:8080` (or your server’s IP:8080).

You’ll see a page asking for an initial admin password. Get it with:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Copy and paste this into Jenkins.

**Install suggested plugins:**  
Follow the on-screen instructions and then continue.

**Create Admin user or skip:**  
You can skip and continue as admin, but best practice is to create a proper admin user.

#### 8. Installing Docker

Docker is needed to build and run container images of our Python app.

**Remove older Docker versions if any:**
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

**Install Docker prerequisites:**
```bash
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

**Add Docker GPG key and repository:**
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] 
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Install Docker engine:**
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

**Allow your user to run Docker without sudo:**
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

Check Docker version:
```bash
docker version
```
You should see client and server info.

#### 9. Creating a Dockerfile

**What is a Dockerfile?**  
A Dockerfile is a set of instructions that tell Docker how to build a Docker image for your application. For our Python app, the Dockerfile will:

- Start from a Python base image.
- Copy our code into the image.
- Install Flask and Flask-RESTful.
- Expose port 3333.
- Run our Python app.

**Create Dockerfile:**
```bash
cd ~/pythonapp
sudo nano Dockerfile
```

**Dockerfile content:**
```dockerfile
FROM python:3.8
WORKDIR /src
COPY . /src
RUN pip install flask
RUN pip install flask_restful
EXPOSE 3333
ENTRYPOINT ["python"]
CMD ["./src/hello_world.py"]
```

This tells Docker:  
- Use Python 3.8 as the base image.  
- Set `/src` as the working directory.  
- Copy all files into `/src`.  
- Install required Python libraries.  
- Expose port 3333 (the port Flask uses).  
- When the container runs, execute `python ./src/hello_world.py`.

#### 10. Building and Running the Docker Image

**Build the image:**
```bash
docker build -t hello_worldpython .
```
The `-t` flag names the image `hello_worldpython`.

**Run the container:**
```bash
docker run -p 3333:3333 hello_worldpython
```
Visit `http://localhost:3333` (or your server’s IP:3333) in your browser, and you should see "Hello World!".

#### 11. Creating a Jenkinsfile for CI/CD

**What is a Jenkinsfile?**  
A Jenkinsfile is a file that Jenkins uses to understand the stages of your pipeline. It describes what to do when code changes occur: cloning code, building images, pushing images, and deploying containers.

**Create Jenkinsfile in the project:**
```bash
sudo nano Jenkinsfile
```

**Jenkinsfile content:**
```groovy
node {
    def application = "flask-sample-app"
    def dockerhubaccountid = "dockerforb" // replace with your Docker Hub username

    stage('Clone repository') {
        checkout scm
    }

    stage('Build image') {
        app = docker.build("${dockerhubaccountid}/${application}:${BUILD_NUMBER}")
    }

    stage('Push image') {
        withDockerRegistry([ credentialsId: "dockerHub", url: "" ]) {
            app.push()
            app.push("latest")
        }
    }

    stage('Deploy') {
        sh ("docker run -d -p 3333:3333 ${dockerhubaccountid}/${application}:${BUILD_NUMBER}")
    }

    stage('Remove old images') {
        sh("docker rmi ${dockerhubaccountid}/${application}:latest -f")
    }
}
```

**What this pipeline does:**

- **Stage 1 - Clone repository**: Gets the code from GitHub.
- **Stage 2 - Build image**: Builds a Docker image from our Dockerfile and tags it using the Jenkins build number.
- **Stage 3 - Push image**: Pushes the newly built image to Docker Hub so it can be accessed anywhere.
- **Stage 4 - Deploy**: Runs the container from the Docker Hub image, making it live.
- **Stage 5 - Remove old images**: Cleans up old images to save space.

#### 12. Pushing Code to GitHub

Now that we have `src/hello_world.py`, `Dockerfile`, and `Jenkinsfile` in our folder, let’s push to GitHub.

```bash
cd ~/pythonapp
git add *
git commit -m "First push of the python app"
git push -u origin main
```

Check GitHub’s repository page and you’ll see your files there.

#### 13. Setting Up Jenkins Credentials for Docker Hub

Jenkins needs your Docker Hub credentials to push images there.

**In Jenkins:**
- Click on "Manage Jenkins".
- Go to "Manage Credentials".
- Under "System" -> "Global credentials", click "Add Credentials".

Select "Username with password" as the kind, and fill in:
- Username: Your Docker Hub username.
- Password: Your Docker Hub password or token.
- ID: `dockerHub`

Click "Create".

#### 14. Creating a Jenkins Job (Pipeline)

- In Jenkins, click "New Item".
- Enter a name, select "Pipeline", then click "OK".

**Configure the pipeline:**
- Under "Build Triggers", select "Poll SCM".
- In the "Schedule" box, type `* * * * *` (this checks the repo every minute for changes).
- Under "Pipeline" -> "Definition", choose "Pipeline script from SCM".
- "SCM" -> Git. Paste your GitHub repo’s HTTPS URL.
- Change branch to `main` (if your repo’s default is main).
- Jenkinsfile should be auto-detected if you named it `Jenkinsfile`.

Click "Save".

#### 15. Running and Verifying the Pipeline

**Trigger a build:**
- On the project page, click "Build Now".
- Jenkins will go through the stages. Check the console output for logs.

If everything is correct:
- Jenkins will have built and pushed an image to Docker Hub.
- Jenkins will have run the container.

Check Docker Hub – you should see your image with tags like `latest` and the build number.

On your server, run:
```bash
docker ps
```
You should see the running container.

**Check in the browser again:**  
`http://localhost:3333` should show "Hello World!".

#### 16. Testing the Full Flow

**Make a small code change:**
Change the message in `hello_world.py` from "Hello World!" to "Hello World! I am learning DevOps!".

Push the change to GitHub:
```bash
git add *
git commit -m "Updated greeting message"
git push
```

Jenkins will detect this change in about a minute (due to the Poll SCM setting) and trigger another build automatically. After the pipeline finishes, check the browser again – you’ll see the updated message. This proves that the pipeline is working: code changes automatically lead to a rebuilt Docker image and updated container.

---

### Understanding the Big Picture

What we have done is establish a fully automated pipeline:

1. **Code Update**: When you change the code and push to GitHub…
2. **Jenkins Detection**: Jenkins regularly checks GitHub for changes.
3. **Build & Push**: Jenkins builds a new Docker image and pushes it to Docker Hub.
4. **Deploy**: Jenkins then runs a new container from that freshly built image.
5. **Updated Application**: The running application updates to the new version automatically.

---

### Key Takeaways

- **Git & GitHub**: Handle version control and remote repository.
- **Jenkins**: Automates the process (CI/CD pipeline), triggered by changes in the GitHub repo.
- **Docker & Docker Hub**: Containerizes the application, making it easy to deploy and scale.
- **Pipeline as Code (Jenkinsfile)**: Keeps the pipeline steps under version control, making it easy to modify, share, and reproduce environments.

---

# Notes CI CD Pipeline

### What is a CI/CD Pipeline?

**CI/CD** stands for **Continuous Integration/Continuous Deployment (or Delivery)**. It’s a set of practices and tools designed to help developers integrate their code changes frequently, get them tested automatically, and then deploy them to production or a staging environment in an automated way. The benefits of CI/CD include:

- Ensuring code integrations happen smoothly.
- Automating testing and building steps so that issues are caught early.
- Reducing manual, repetitive work for developers and operations.
- Speeding up the delivery cycle, allowing faster updates and improvements.

When we talk about a **CI/CD pipeline**, we’re describing a connected series of steps that start with a code change in a repository (like GitHub) and end with that updated code being deployed and running in an environment (like a container on a server).

---

### Our Example Scenario

In this tutorial, we are focusing on deploying a simple Python web application using a CI/CD pipeline that involves several key tools and platforms:

1. **Git & GitHub**: For version controlling our code and hosting it remotely.
2. **Jenkins**: An automation server that will run our pipeline steps. It will:
   - Pull the latest code changes.
   - Build our Docker image.
   - Push the Docker image to Docker Hub.
   - Pull the image back and run it as a container.
3. **Docker & Docker Hub**: For containerizing our application. Docker Hub is a public registry where we store and retrieve our application’s Docker images.

**The Flow**:

- You make changes to your local code and push them to GitHub.
- Jenkins polls GitHub regularly. When it detects a new commit (a code update), it triggers a pipeline.
- That pipeline builds a Docker image of the new application version and pushes it to Docker Hub.
- Jenkins then deploys this new image as a running container on your server.
- The updated application is now available to users through the container’s exposed port.

This means every time you push a code change to GitHub, your running application will update automatically without you having to do all the manual steps.

---

### Prerequisites and Setup

We assume a few things:

- You have a machine (like an Ubuntu server or VM) where you will install Java, Git, Jenkins, and Docker.
- You have a GitHub account (for remote code repository).
- You have a Docker Hub account (to store built images).

**Why these prerequisites?**  
Jenkins requires Java to run, Git is necessary for pulling code, and Docker is required to build and run containers. GitHub and Docker Hub are cloud services that will host your code and images respectively.

---

### Conclusion

In this deep-dive tutorial, you learned how to:

- Set up Java, Git, and Jenkins on your machine.
- Configure Git and GitHub to securely connect with SSH keys.
- Create a simple Python/Flask application and wrap it in a Docker container.
- Write a Jenkinsfile to define a CI/CD pipeline.
- Automatically build, push, and run a new Docker image whenever you push code changes to GitHub.

This setup is a basic foundation. In more advanced scenarios, you can incorporate testing frameworks, more complex build steps, different deployment strategies (like Kubernetes), and integrate further DevOps best practices. But as a beginner-friendly introduction, this approach lets you see how code changes can move smoothly from your development environment, through CI/CD automation, into a running production-like environment with minimal manual intervention.


# Additional Notes
Below is a significantly expanded explanation focusing specifically on the role of Docker and Jenkins within the CI/CD pipeline, why we use them, how they fit together, and how the example usage scenario works. Think of this as an extended background and rationale combined with a more detailed walk-through of each step and tool involved. This should help you—and anyone reading it—develop a deeper understanding of what’s really happening under the hood.

---

### Why Jenkins?

**Jenkins** is an open-source automation server widely used for implementing CI/CD pipelines. Imagine you have a development team pushing code to a shared repository like GitHub. Without automation, every time you update the code, you’d have to:

- Manually pull the latest changes to a server.
- Rebuild the application or its container image.
- Manually push that image to a registry (if you’re using containers).
- Manually run the new container to serve your application.

This is not only time-consuming but also prone to human error. Jenkins solves this problem by automating these tasks. It continuously listens for changes in your code repository. Whenever it detects a new commit or a code change, it automatically executes a series of predefined steps. These steps, defined in a **Jenkinsfile**, describe exactly what should happen with each update.

**Key Benefits of Jenkins:**
- **Continuous Integration**: Integrate code changes into a shared repository and ensure they are tested and built automatically.
- **Continuous Delivery/Deployment**: Once tested and built, code can be automatically deployed to a test server, staging environment, or even production.
- **Extensibility**: Jenkins has a massive plugin ecosystem, allowing integration with countless tools (Docker, Kubernetes, AWS, etc.).
- **Visibility and Control**: Jenkins provides a dashboard where you can see the status of builds, pipelines, and tests, giving you immediate feedback on the health of your code.

---

### Why Docker?

**Docker** is a technology that allows you to package your application and all its dependencies into a single, lightweight, and portable unit called a **container**. Containers make it easier to run your application consistently across different environments—your laptop, a staging server, or a production cluster—without worrying about library conflicts or system variations.

**Key Benefits of Docker:**
- **Portability**: A container runs the same way no matter where you deploy it.
- **Consistency**: All dependencies, runtime, and configuration are wrapped inside the container image. No “works on my machine” issues.
- **Easy Scaling**: Spinning up multiple containers is simpler than manually setting up more servers or VMs.
- **Faster Deployment**: Containers start quickly, making continuous deployment cycles more efficient.

When combined, Jenkins and Docker form a powerful duo. Jenkins orchestrates the steps needed to build, test, and deploy your code. Docker ensures that your application runs in a predictable, reproducible environment.

---

### Why Combine Docker and Jenkins?

By integrating Docker into Jenkins, you get an automated, repeatable pipeline that can:

1. **Pull the latest code** from GitHub.
2. **Build a Docker image** from that code, ensuring that your application and its dependencies are encapsulated properly.
3. **Push the Docker image** to a registry like Docker Hub so it can be stored and accessed later.
4. **Deploy the container** automatically—Jenkins can pull the container from Docker Hub and run it locally or on a remote server.
5. **Test the running container** by hitting the application endpoint to ensure it’s working as expected.

This approach removes manual intervention. For each code change, the entire cycle (Build → Push → Deploy) happens automatically, which is the essence of CI/CD.

---

### Example Usage Scenario (Detailed)

Let’s dive deeper into a scenario where you’re working on a Python web application (like a small Flask API). Here’s what the end-to-end workflow looks like with more detail:

1. **Developer Workflow Without CI/CD:**
   - You write code on your local machine.
   - You run the application locally on your laptop with `python helloworld.py` to see if it works.
   - You manually build a Docker image (e.g., `docker build -t myapp:v1 .`).
   - You push the image to Docker Hub (e.g., `docker push myusername/myapp:v1`).
   - You log into your production server via SSH, pull the image, and run it.
   - This manual process is repeated every time you make a change.

2. **Developer Workflow With Jenkins + Docker:**
   - You write code on your local machine.
   - You commit your changes and run `git push` to your GitHub repository.
   - **At this point, you do nothing else manually.** Jenkins takes over.

3. **What Jenkins Does Automatically:**
   - Jenkins regularly polls your GitHub repo (or listens for a webhook) to detect changes.
   - When a new commit is found, Jenkins starts executing the pipeline steps defined in the Jenkinsfile.
   
   **Pipeline Steps (As Defined in the Jenkinsfile):**
   
   a. **Clone Repository (Stage: Clone repository)**  
      Jenkins checks out the latest code from your GitHub repo. This ensures Jenkins always works with the freshest code.

   b. **Build Docker Image (Stage: Build image)**  
      Using the `docker.build` command (provided by Jenkins’ Docker Pipeline plugin), Jenkins reads your Dockerfile. The Dockerfile typically starts from a base image (like `python:3.8`), installs your dependencies (`pip install flask`), copies your application code, and sets the startup command. By doing this, Jenkins creates a self-contained Docker image that represents your updated application.

   c. **Push Docker Image (Stage: Push image)**  
      Once the image is built, Jenkins authenticates to Docker Hub (using credentials you’ve set up in Jenkins) and pushes two versions of your image:
      - A version tagged with the Jenkins build number (e.g., `myusername/pythonapp:15`).
      - A version tagged as `latest` (e.g., `myusername/pythonapp:latest`).
      
      Storing images in Docker Hub ensures that your application’s packaged form is accessible from anywhere—testing servers, production servers, or even local machines.

   d. **Deploy the Container (Stage: Deploy)**  
      Now that the image is on Docker Hub, Jenkins can run a container based on that image. It pulls down the `myusername/pythonapp:15` image and starts a container on your server. This step could be as simple as `docker run -d -p 3333:3333 myusername/pythonapp:15`. The `-d` flag means “run in detached mode” so Jenkins doesn’t get stuck waiting, and `-p 3333:3333` maps the container’s port to your machine’s port, making the application accessible via `http://localhost:3333`.

   e. **Cleanup (Stage: Remove old images)**  
      Over time, you might accumulate many old images. This stage removes outdated images to save storage space and keep your environment clean. It might remove old build tags, leaving only the current and `latest` versions of your image.

4. **End Result:**
   - With each commit you push, Jenkins updates the running container with the new code automatically.
   - You open your browser, navigate to `http://your-server-ip:3333`, and see the updated version of your Python app.
   - No manual pulling, building, pushing, or running steps are required from you once the pipeline is set up.

---

### More on the Setup Details

**Jenkins Setup:**
- **Java Installation:** Jenkins is a Java-based application, so you need Java installed. On Ubuntu/Debian systems, this can be as simple as `sudo apt-get install -y openjdk-11-jre`.
- **Jenkins Installation:** Add the Jenkins repository to your package manager and install it with `sudo apt-get install -y jenkins`.
- **Initial Configuration:** Once Jenkins is running, access it via `http://localhost:8080`, unlock it with the secret key, and install the suggested plugins. This gives you a baseline Jenkins environment ready for configuring pipelines.

**Docker Setup:**
- **Docker Installation:** Install Docker Engine and CLI to build and run containers. On Ubuntu, this involves adding the Docker GPG key, setting up the Docker repository, and installing via `sudo apt-get install docker-ce`.
- **Non-Root Docker Use:** By default, Docker requires `sudo` to run commands. Add your user to the `docker` group and `newgrp docker` to run Docker commands without `sudo`. This is important because Jenkins often runs under its own user, and you’ll want Jenkins to have the ability to run Docker commands for building and deploying containers.

**Integrating Jenkins and Docker:**
- **Docker Plugins for Jenkins:** Install the "Docker" and "Docker Pipeline" plugins via Jenkins > Manage Jenkins > Manage Plugins. These plugins allow your Jenkins pipeline to call Docker commands natively.
- **Credentials Management:** Store Docker Hub credentials in Jenkins. With this, Jenkins can login to Docker Hub, push images, and pull them without exposing your password in the Jenkinsfile.

**Setting Up the Jenkinsfile:**
- **Pipeline as Code:** The Jenkinsfile is checked into your repository. This means that all the instructions for building and deploying your app are version-controlled, just like your code.
- **Stages:** The Jenkinsfile defines stages like "Clone repository," "Build image," "Push image," "Deploy," and "Remove old images." This structure helps you visualize the flow of tasks and debug which stage might have issues if something goes wrong.
- **Environment Variables and Parameters:** You can parameterize your Jenkinsfile, allowing you to easily change things like Docker image names, repository URLs, or credential IDs without hardcoding them.  

---

### Example Usage Beyond the Basics

In a real-world scenario, you might:

- **Add Tests:** Before building the Docker image, you could have Jenkins run unit tests. Only if the tests pass does Jenkins continue to the build and push stages. This ensures you never deploy broken code.
- **Use Multiple Environments:** Jenkins could build a "dev" image first, deploy it to a testing environment, run integration tests, and then only if everything passes, tag the image as "prod" and deploy it to production servers.
- **Integrate With Cloud Providers:** Instead of running on a local machine, Jenkins and Docker could be integrated with a Kubernetes cluster on AWS or GCP, making it easy to scale out the number of containers or roll back to previous versions if something goes wrong.
  
---

### Why This Matters

By understanding the "why" and the "what" of using Docker and Jenkins together, you see how this approach drastically simplifies and streamlines the lifecycle of your application. Instead of relying on manual, repetitive commands, you have a defined, automated process. This leads to:

- **Faster Release Cycles:** Code changes can be tested, built, and deployed in minutes rather than hours or days.
- **Higher Reliability:** Automated tests and a standardized environment (containers) reduce the chance of errors slipping into production.
- **Better Developer Experience:** Developers can focus on writing code, confident that their changes will be quickly and safely integrated into the running application.

---

### In Summary

- **Jenkins** is your automation engine, orchestrating the pipeline.
- **Docker** is the packaging and runtime tool that ensures your app runs the same way everywhere.
- Together, they allow you to fully automate the journey from code commit to a running, up-to-date application, building a robust and efficient CI/CD pipeline that can grow and evolve as your project and team scale up.

This deeper explanation should give you a clearer picture of how Jenkins and Docker fit into the CI/CD puzzle and why they’re so commonly used together.