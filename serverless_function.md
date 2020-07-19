### Serverless Function

I'm taking a different route with this question as I want to be as honest as I can be. I havent got the knowledge, yet, to be able to deal with serverless functions but am hoping that what I am going to show you can help you understand what DevOp skills I do have (self taught and used commercially) and show that I am definitely willing to learn more. I built my CI/CD pipeline for my current website and I can give you a breakdown of the tech stake I used.

My [website](https://matthewpigram.co.uk) was created as a one page CV to introduce myself and show off what skills I have. I used the following to create it:

- Basic HTML/CSS/Java
- Flask backend + Gunicorn for prod
- Docker
- Github Actions
- Ansible
- HAProxy
- Let's Encrypt

I decided to build my website using the Flask framework because it was lightweight, easy to configure and would serve my purpose far better than using apache or nginx. Especially as I was going to use Docker and wanted to try and keep the container low in size.

So, once I had my website built and was happy with it, I then needed to create the Flask framework for my web app.

The main file that would serve the "index.html" was the "app.py" - which contained the following

``` python
from flask import Flask, render_template

# creates the flask application

app = Flask(__name__)
app.jinja_env.auto_reload = True
app.config['TEMPLATES_AUTO_RELOAD'] = True

# Home page route

@app.route('/')
def home():
    return render_template('index.html')
def before_request():
    app.jinja_env.cache = {}

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

This was basically importing the Flask framework, creating the Flask application, creating a route to my home page and running it locally on "0.0.0.0".

Now I have my web app running in Flask - as it should be. My next step was containerisation using Docker.

I opted to use Ubuntu 16.04 as the container OS. I could have used Alpine as it's very lightweight but with it's reduced size you do lose quite a bit of functionaility, which makes troubleshooting issues painful.

Below is what was contained in my Dockerfile:

``` Dockerfile
FROM ubuntu:16.04

MAINTAINER Your Name "matthew.pigram2@gmail.com"

RUN apt-get update -y && \
    apt-get install -y python3-pip python-dev && \
    apt-get install -y vim

COPY ./requirements.txt /app/requirements.txt

WORKDIR /app

RUN python3 -m pip install --upgrade pip && \
    python3 -m pip install Flask gunicorn

COPY . /app

CMD ["gunicorn" , "-b" , "0.0.0.0:5000" , "app:app" , "--certfile fullchain.pem" , "--keyfile privkey.pem"]It
```

I'm making sure with this Dockerfile I am updating to most recent patches, installing my essentials such as Vim and Python3-pip of course! Then I'm copying in my requirements file (which has one item in it "Flask=1.1.1") and making sure my working directory is "/app".

Then, I'm running an update to pip and making sure Flask and Gunicorn are installed.

Now I have my running Container and I can access my webpage no problem. The next step is making the process of updating my Docker images a lot easier and this is where I use Github actions.

With this step I created a simple yml file to use with actions and this will build my image and then push it to Dockerhub

``` yml
name: Docker Image CI 

on:
    release:
            types: [published]

jobs:
    build-and-deploy:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v2
            - name: 'Login to DockerHub Registry'
              run: echo ${{ secrets.DOCKERHUB_PASSWORD }} | docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin

            - name: 'Get the version'
              id: vars
              run: echo ::set-output name=tag::$(echo ${GITHUB_REF:10})

            - name: 'Build the tagged Docker image'
              run: docker build . --file Dockerfile --tag pigstah/flask:${{steps.vars.outputs.tag}}

            - name: 'Push teh tagged Docker image'
              run: docker push pigstah/flask:${{steps.vars.outputs.tag}}

            - name: 'Build the latest Docker image'
              run: docker build . --file Dockerfile --tag pigstah/flask:latest

            - name: 'Push the latest Docker image'
              run: docker push pigstah/flask:latest
```

I have two pushes within this yml file, one to name it as the release tag and one to name is as "latest". This was just my preference, as I could keep track of the updates, as well as just being able to pull latest version down using the "latest" tag.

I also added my variables for my dockerhub username and password to this repo's secrets tab.

So finally I have managed to push any updates to Github as a tagged release and this will automatically build my Docker images and push them to my Dockerhub repo. Next, I want to be able to deploy my website to my Digital Ocean droplit I have created.

I do this with an Ansible and I ssh directly to my Droplet and then initiate the commands through the playbook. I was planning on adding some extra features within my website, and wanted the option to only update certain aspects of the site if the "BUILD_UPDATE" variable was set to "True". In my case it doesn't matter either way because if I have any updates, I will always be pulling a new image from Dockerhub to apply the updates. Another use case for it could be that if I have any issues with the container, I could tear down the container and re-build it to solve the issues.

The website is up and running, with automation for the build steps and automation for my deployment.

The final piece of the puzzle is directing traffic to my container, as this will be running on a localhost and port 5000, so any incoming traffic to ports 443 will just recieve an error. I decided to use HAProxy for this, mainly because I used it in a last company and again was fairly lightweight compared to likes of NGinx.

So in order to get HAProxy reversing my requests to the website, I needed to ammend the config file to point to my container on port 5000.

```Bash
# This is taken from my own config on my webserver

frontend public
    # Listen on ports 80 and 443
        mode http
        bind *:80
        bind *:443 ssl crt /etc/haproxy/certs/matthewpigram.co.uk.pem 

    # Define ACLs for each domain
        acl one hdr(host) -i www.matthewpigram.co.uk
        acl two hdr(host) -i matthewpigram.co.uk
       
    # Figure out which backend 0(= VM) to use
        use_backend onepagecv if one
        use_backend onepagecv if two

backend onepagecv
        mode http
        option forwardfor
        option httpchk HEAD / HTTP/1.1\r\nHost:localhost
        server s1 *:80 check
        server s1 *:443 check
        http-request set-header X-Forwarded-Port %[dst_port]
        http-request add-header X-Forwarded-Port https if { ssl_fc }

        # Redirect to flaskv1 container
        server s1 0.0.0.0:5000 check

backend letsencrypt-backend

        server letsencrypt 127.0.0.1:54321
```

Sidenote - I'm using Let' Encrypt for my SSL certs, and auto renew is on using their bot and a cronjob. This is just so I don't have to worry about renewing every 3 months.

Finally the website is up and running, working as planned and I can update my website in some very short simply steps!

### Five generic Metrics

- Build time
- Number of failed builds
- Number of errors
- Memory usage
- Number of succesful invocations
