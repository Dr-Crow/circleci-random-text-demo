# CircleCI Random Text Demo
[![CircleCI Build Status](https://circleci.com/gh/Dr-Crow/circleci-random-text-demo.svg?style=shield)](https://circleci.com/gh/Dr-Crow/circleci-random-text-demo) [![Software License](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/Dr-Crow/circleci-random-text-demo/main/LICENSE) [![Docker Pulls](https://img.shields.io/docker/pulls/drcrow/random-text-demo)](https://hub.docker.com/r/drcrow/random-text-demo)

Utilizing CircleCI's platform we can build, test, and deploy a simple application. In this demo we are building a
Flask application to serve a webpage that contains a button. When the button is pressed, randomly generated text will be
displayed on the page.

With the release of CircleCI's Arm executor, we can now build on multiple architectures via CircleCI's Cloud offering.
Building on architecture offers better performance/speed and the assurance that your application will indeed work on your platform.

For this demo, we will be using PyTest to test our Flask application, build Docker images for each architecture(`arm64` and `amd64`),
test the Docker images with Goss/dgoss, and construct a manifest to tie the Docker images together into a single tag.
Finally, we will deploy our Flask application to Heroku utilizing CircleCI's Heroku orb.

To check out the fully deployed application, please visit `https://circleci-random-text-demo.herokuapp.com/`.

## CI/CD Setup

We want to setup CircleCI to accomplish the following:

- Trigger on every commit to `main`
- Run our tests with Pytest
- If the tests pass, build the Flask Demo image
- Test the image to make sure it is functioning with Goss/dgoss
- If tests pass, tag the images and push the architecture specific images
- Create Docker manifests to allow users to pull down the image without caring about architecture
- Deploy the Flask application to Heroku

That should cover the CI and CD part of this demo. Utilizing some more advanced features of CircleCI, such as matrices
and parameters, we can speed up the overall build time of our pipeline. Parallelization helps out a lot during the 
multi-arch builds, since each architecture is running the same jobs. Having both architectures running at the same time
cuts down the overall build time of our pipeline by a couple of minutes. 

You can see the general flow of the workflow below:

![Workflow](/.img/workflow.PNG?raw=true "Workflow")


## Deployment Documentation

### Python/Flask Application

Project structure:

```
.
├── .ci                         - Docker Build Tools
├── .circleci                   - CircleCI Config
├── .imgs                       - Image Folder
├── demo
|    ├── app/                   - Python Application
|    ├── requirements/
|    ├── tests/                 - PyTest Files
|    └── flask_run.py
|    └── config.py
├── Dockerfile                  - Flask Application Dockerfile
├── goss.yaml                   - Goss Test Configuration
├── VERSION                     - Version Number for Docker Images

```

[_Dockerfile_](Dockerfile)

```dockerfile
FROM python:3.9-slim

ARG USER=demo
ARG GROUP=demo
ARG UID=1000
ARG GID=1000
ARG PORT=8080

# Set enviroment variable for Flask
ENV PORT=${PORT}

# Expose the server port
EXPOSE ${PORT}

# Copy in requirements file for demo
COPY ./demo /demo

# Add a group and user to not use root
# Then set permissions to the demo files
RUN addgroup --gid ${GID} ${GROUP} \
  && adduser --disabled-password --no-create-home --home "/demo" --uid ${UID} --ingroup ${GROUP} ${USER} \
  && chown -R ${UID}:${GID} /demo

# Set Working Directory
WORKDIR /demo

# Use the production requirements file to install needed Python Packages
# Then delete requirements and tests folder as they are not needed
RUN pip install --no-cache-dir -r requirements/production.txt \
    && rm -rf requirements/ tests/

# Switch to non-root user
USER demo

# Set Python3 as entrypoint
# Using -u for unbuffered ouput
ENTRYPOINT ["python3", "-u"]

# Run the Flask Demo
CMD ["flask_run.py"]
```


## Deploy with Docker (Locally)
We can build the Dockerfile locally using the commands below:

```
$ docker build -t random-text-demo:local . 
[+] Building 9.1s (10/10) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                    0.0s
 => => transferring dockerfile: 32B                                                                                                                     0.0s
 => [internal] load .dockerignore                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                         0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                      0.0s
 => [internal] load build context                                                                                                                       0.0s
 => => transferring context: 7.39kB                                                                                                                     0.0s
 => CACHED [1/5] FROM docker.io/library/python:3.9-slim                                                                                                 0.0s
 => [2/5] COPY ./demo /demo                                                                                                                             0.1s
 => [3/5] RUN addgroup --gid 1000 demo   && adduser --disabled-password --no-create-home --home "/demo" --uid 1000 --ingroup demo demo   && chown -R 1  1.0s
 => [4/5] WORKDIR /demo                                                                                                                                 0.1s
 => [5/5] RUN pip install --no-cache-dir -r requirements/production.txt     && rm -rf requirements/ tests/                                              7.2s
 => exporting to image                                                                                                                                  0.4s
 => => exporting layers                                                                                                                                 0.4s
 => => writing image sha256:6e56db70318641129e787dc993263e8ef43314e2b4d735436896075b976c8f2e                                                            0.0s
 => => naming to docker.io/library/random-text-demo:local                                                                                               0.0s

$ docker images
REPOSITORY                                          TAG            IMAGE ID       CREATED          SIZE
random-text-demo                                    local          6e56db703186   34 seconds ago   138MB
```

Or we can use the prebuilt images by pulling them down from DockerHub using the command below:

```
$ docker pull drcrow/random-text-demo:1.1
1.1: Pulling from drcrow/random-text-demo
69692152171a: Already exists                                                                                                                                 59773387c0e7: Already exists                                                                                                                                 3fc84e535e87: Already exists                                                                                                                                 68ebeebdab6f: Already exists                                                                                                                                 3d3af2ef8baa: Already exists                                                                                                                                 3fc7aa65f4f9: Pull complete                                                                                                                                  b7b875e07c9c: Pull complete                                                                                                                                  445ecd4a33e4: Pull complete                                                                                                                                  Digest: sha256:d8dbbc9e0d4ae1116adf29393dde5ba7dac57a2074d2975063645934a20f50fd
Status: Downloaded newer image for drcrow/random-text-demo:1.1
docker.io/drcrow/random-text-demo:1.1
```

Finally, we can run the image by issuing the following command:

```
$ docker run -d -p 8080:8080 drcrow/random-text-demo:1.1
```

## Expected Results

Listing the containers should show the random-text-demo container running and the port mapping as below:

```
$ docker ps
CONTAINER ID   IMAGE                         COMMAND                  CREATED              STATUS              PORTS                                       NAMES
f77d9f84fc69   drcrow/random-text-demo:1.1   "python3 -u flask_ru…"   About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   amazing_hawking                             deployment_db_1
```

After the application starts, we can navigate to `http://localhost:8080` in the web browser. We should see a page
that looks like this:

![Home Page](/.img/home_page.PNG?raw=true "Home Page")