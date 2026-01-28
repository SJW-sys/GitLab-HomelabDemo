# GitLab-HomelabDemo
How to Selfhost your own Git Repository with Gitlab and setup Gitlab runners for Pipelines.

## Why this project:
I like a challenge... while this is true, I wanted a private space I can tinker and learn without stressing about leaking anything private, or with the potential to compromise my homelab. I also want to develop my skills with git, tooling within a DevOps tool like GitLab and more CI/CD pipeline experince.

## Setup:

### Deployment Design:
 We will be deploying Gitlab instance via docker on our main server, this might be where you run other containers typically in your lab. We will then be going with a more standard approach you might see in an enterprise setting, and deploying the Gitlab-runner (which is what executes pipelines from Gitlab) on a separate host for security and control. Finally we will setup a target server (maybe your main server with the Gitlab instances) where we want the Gitlab-runner to be able to ssh into and complete commands, like spinning up a docker container. We will be setting up a dedicated account on the target host to prevent unneeded additional access, and generating a SSH key pair that the runner will NOT be able to store the ssh private key at rest (stored in ram within container when running, and controlled via a masked variable in gitlab instance).

#### Networking Design:
 We will not dive into the networking of the computers as this can be a complex task that is a large scope to add, maybe I will add it later, however **you are expected to have the units in this demo internet accessible and able to communicate via ssh to each other**. There is much to be said about networking and hardening it, but I would encourage you to never expose anything without good reason on any network (zero trust mindset). I personally expose the bare minimum (inbound and outbound), change default ports, use vlans, run a IDP/ISP (snort), using explicit rules (this computer can only talk to that host on that ip using this protocol during this time), and review firewall at every level (CT, Host, hypervisor and network firewall). Those tools when aligned with hardening at other OSI levels, can be critical to understand in an enterprise environment and prevent the next ransomware breach (until someone clicks on the pishing email). I would say my setup is overkill and alot of work for most homelabs, however it is a great learning experience to trial.


### Gitlab via docker:
Prerequisites: linux machine (server A - gitlab), sudo & Git (version 28.2.2) & Docker Engine (version: 28.2.2) Installed, Running Caddy from my demo (or change the .env GITLAB_URL and the docker network in the compose yaml), account with sudo level permissions, a web browser.

1) Clone this git repository locally:

    `git clone`

2) Navigate into DeploymentFiles:

    `cd DeploymentFiles`

3) Always review external scripts, commands and tools before executing on your system.

4) run Docker Compose in detached mode, which will pull images and setup everything within the compose file:

    `sudo docker compose up -d`

5) Access interface and change default password. I also recommend turning off "sign up", so it could not be leveraged in an attack with access to your local network. Default webpage for instance will be:

    https://localhost:4443


### Gitlab Runner install on another host via script:
Prerequisites: linux (debian 12.3 trixie) machine (server B - gitlab runner), Gitlab running, Curl & sudo (or permissions) & Docker Engine installed & Open-ssh server installed, account with sudo level permissions.

This can be running on the same host as the gitlab docker instance, however we will deploy directly on a 2nd server as this is more common in a enterprise environment. In my homelab, my is separated via vlan and has very strict firewall rules (Network, Hypervisor and Host), this will not be outlined here. While more complex, this is a design and security standard I believe is worth the time to setup.

You can even use docker itself to run your runner, however their are some added complexities with avoiding docker in docker. This is my [recommended guide](https://youtu.be/zBrP8MzA5y0) for this deployment and more details on the complexities.

1) Pull down gitlab official install script to setup repositories depending on OS.

    `curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" -o script.deb.sh`

2) Always review external scripts, commands and tools before executing on your system.

    `less script.deb.sh`

3) Run script:

    `sudo bash script.deb.sh`

4) Install gitlab-runner:

    `sudo apt install gitlab-runner`

5) configure:

**Important** - You can either run the registration tool, or manually fill out the toml config file. I had issues with the registration tool (ran it as non-root initially), so I had to go the config file route. It did allow additional settings outside the scope of this demo to be set easily. I have provided an example of my config you could use (ExampleFiles/config.toml). Just get the token you will generate in at the end of step D, to use in the file and correct the FIXME with your environment information. Be sure to set it at /etc/gitlab-runner/config.toml.

 A) Open gitlab instance, and decided if you want the runner to run at the instance, group or project level. The steps are relatively the same, but we will be demoing how to setup for a group.

 B)  Go to "groups" on the left hand size, click. Then click "New Group" on the top right. Select "new group"

 C) Create a group to preferences, we will refer to it as "HomelabDemoGroup"

 D) Navigate back and into the group, "HomelabDemoGroup". Open "build" drop down, click "Runners". Click "Create group runner".

 The most important thing to set here, is tags; if the tags noted here, match the tags in a pipeline, this runner is then eligible to run your pipeline. We will use "HomelabDemo" as a tag. After these are filled in, you can leave everything else blank.

 E) Take the gitlab-runner register command below, and run that on your gitlab-runner server as root/sudo. Command will look like this:

`sudo gitlab-runner register  --url URL --token TOKEN`

 F) You will get asked to name your server (you can leave it blank and it will default). Then you will ask for a "executor type". For this deployment we will be choosing "Docker", with using the default image as "python:3.12-slim".

 Executor Type is HOW your gitlab-runner executes pipelines. Ie. "shell" will use the host system of the gitlab-runner to execute the commands. "Docker" will pull and run a docker container to execute the jobs within. "Kubernetes", spins up a pod for the job to execute.

6) start and enable via systemd or distro init controller:

    `sudo systemctl enable gitlab-runner
    sudo systemctl start gitlab-runner`

7) check gitlab-runner is enabled:
    
    `sudo systemctl status gitlab-runner`

8) check registered:

    `sudo gitlab-runner list`

9) verify it can run jobs:

    `gitlab-runner run`

10) (Optional) staging default docker image for runner:

    `sudo docker pull python:3.12-slim`

### Setup Pipeline Demo and SSH Keypair
Prerequisites: linux (any computer), a web browser, openssh-client or openssh-server installed.

1) Generate a key-pair w/o a passphrase (while passphrases are best practice, its outside the scope of this demo to show that configuration)

    `ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_NAME_ed25519`

2) We will encode this with base64 for making it possible to hide the variable in gitlab instance. If you choose not to do this in the demo, you will need to modify the example pipeline to not unencrypt it. Be able to reference the output of the below command temporary.

    `base64 -w0 -i ~/.ssh/id_NAME_ed25519`

3) Navigate to the url of your gitlab instance and login.

4) Navigate to our group we setup, "HomelabDemoGroup".  We will be setting this SSH key for our group so it can be used for any project pipeline within the group, but you can set these for at the instance level, or the project level as well for more granular security.

5) On left hand menu, click on the carrot for settings. Click "CI/CD", choose "variables". Click "Add Variable"

6) Configure & save:
Key(name): SSH_HOMELABDEMO_PRIVATE_KEY (if you change this, be sure to modify the example pipeline)
For best practice you would want to select the visibility level of "Masked and hidden" for something as sensitive as a ssh key. The most important part of this is in the "value" field, putting your priv key you generated here that was encoded via base64. Everything else you can leave default or set to your needs.

7) I have provided a pipeline variable file (ExampleFiles/.gitlab-ci.yml), add this to any repositories root, and as long as the below variables are set at a level the pipeline can reference it will run successfully. More details are comments in the pipeline file. **Notice the tags might be different from what you set your pipeline to run, make sure these match.**
    REMOTE_PORT - ssh port to connect to
    PROD_REMOTE_USER - ssh user to connect to your prod server, our demo this is: gitrunner
    PROD_REMOTE_HOST - your prod target server of the pipeline
    TEST_REMOTE_USER - ssh user to connect to your test server.
    TEST_REMOTE_HOST - your test target server of the pipeline

Please note, I have my pipeline setup to target a different server (another set of REMOTE_HOST REMOTE_PORT ) based on the branch name (CI_COMMIT_BRANCH) it was pushed to (main/master or test). That way you could have a test sever you can check you pushes on, before committing to main. Gitlab is smart enough that it sets CI_COMMIT_BRANCH automatically based on the branch name when running a pipeline.

8) (optional) Once you have fully tested the whole deployment, come back and delete (or save in a secure location) the local copy of the keypair (priv and pub), ~/.ssh/id_NAME_ed25519

### Setup Servers to be targeted by Gitlab Runner via ssh:
Prerequisites: Debian Server Deployed, Docker engine installed & Vim (or another text editor) & Open-ssh server installed, account with sudo level permissions.

On target host we are going to take the extra steps to have ssh setup via a dedicated user for better security and control of what the runner can do. I also add it to the docker group, so you can avoid giving it sudo permissions. However it would be best to setup docker rootless, to have docker run in the user space. That is dependant on what containers you will be deploying, and depending on your needs you might run both docker and docker-rootless side by side.

1) make user:

    `sudo useradd gitrunner`

2) Set password so ssh can work without modifying allowing users without password to login (bad security), recommend something random and long that you don't save. Just reset if you ever need it.

    `sudo passwd gitrunner`

3) Normally I am having gitrunner target docker, so add it to the management group.

    `sudo usermod -aG docker gitrunner`

4) Ensure the user has access to docker:

    `sudo -u gitlab-runner docker info`

5) Make sure ssh is setup for the user and permissions are correct.

    `sudo mkdir -p /home/gitrunner/.ssh
    sudo touch /home/gitrunner/.ssh/authorizedkeys
    sudo chown gitrunner:gitrunner --recursive /home/gitrunner
    sudo chmod --recursive 700 /home/gitrunner
    sudo chmod 600 /home/gitrunner/.ssh/authorized_keys
    sudo vim /home/gitrunner/.ssh/authorized_keys ## add .pub key of gitrunner agent server private ssh key we generated earlier.`

### Final setup

Now you just make your repository in Gitlab, modify my example pipeline that you drop in the root of your repo, and push/pull into main branch.


## Resources:
### Gitlab:
- Gitlab : https://gitlab.com/gitlab-org/gitlab
- DockerHub (CE): https://hub.docker.com/r/gitlab/gitlab-ce
- website: https://about.gitlab.com/
- Documentation: https://docs.gitlab.com/
### Gitlab Runner:
- DockerHub: https://hub.docker.com/r/gitlab/gitlab-runner
- Documentation: https://docs.gitlab.com/runner/
### Other helpful resources:
- Forum post on Reverse proxy:https://forum.gitlab.com/t/gitlab-with-docker-and-behind-nginx/99050
- video setup walkthrough on gitlab: https://www.youtube.com/watch?v=qoqtSihN1kU
- gitlab setup via docker guide (1):https://docs.gitlab.com/install/docker/installation/#install-gitlab-by-using-docker-compose
- gitlab setup via docker guide (2): https://medium.com/@BuildWithLal/gitlab-setup-using-docker-compose-a-beginners-guide-3dbf1ef0cbb2


## FAQ for repository visitors
### Why does the repository have 'Demo' in the title?
this "demo" repository is based on real world local deployments in my Homelab. Some settings may be changed, or different for privacy and safety.Typically in a real world scenario, you would use .gitignore to filter out potential sensitive files, as well has have pipeline jobs to find secrets. Typically you also consider storing variables within the git repository platform itself in a mask state to prevent jr devs from pushing secrets accidentally.

### Why multiple branches
I have a test and main branch to more demo a enterprise setup, where you might have people pushing changes to a protected ‘test’ branch that is then has pipelines to stage tooling on a test server. Which once pulled into ‘main’, would deploy the same setup to a production server.

### Where can I find more about this project and your thought process?
I make it a habit that my files typically have dozens of in-line comments to better help anyone using them for the first time to understand what is happening, maybe not always why. Also please check out my blog, it typically has more information on my projects (sometimes the post is still being planned).

### Does ths connect to your other Homelab Demo repositories?
Yes! Most of these demos connect to this exact project or my Caddy demo for the use of ssl termination.

### Was AI used to generate this?
No, but I have learned and expanded my knowledge of the tools within this project (and prior projects) with AI to design a better solution and skills for other deployments. AI I see as a tool and resource that is loose in the market place regardless of your stance, that you need to follow, know how to use, while educating others on its complexities and putting safeguards around it. I firmly design and deploy with my own brainstorming, knowledge, and troubleshooting as my first approach, but i have used AI to troubleshoot, help expand understanding, research, interpreter (ie. Bash to Go) and experimented with vibe coding. I have deeper thoughts and opinions, but those are better discussed rather than a few sentences in a git repository. 

## Issues to note with this "demo"
I wanted to do a proper code repository that could be poke around so you could see commits and pulls that you might normally see in a team production repository, unfortunately due to the overhead and this [issue](https://github.com/orgs/community/discussions/6292), I will be "cutting corners" and doing everything locally then pushing to main directly from my machine. However, I will still leave the demo "main" and "test" branches with there protections.

## Docker features, CI/CD tooling/skills, and other tools leveraged in this project.
- Docker:
    - image pull
    - compose
    - Networking
    - bind mounts
    - ct documentation
    - memory size
    - environment variables
- Yaml
- Git
- Gitlab interface and secret management
- Gitlab deployment
- Gitlab runner deployment
- Pipelines (all in Gitlab formatting)
- Git CD/CI best practices (branches, branch protections, etc)
- Linux (general and permissions)
- Bash
- SSH
- Documentation