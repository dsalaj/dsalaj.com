---
layout: post
title:  "pip install from private github repository in Dockerfile"
date:   2017-10-04 09:58:00 +0100
---


This is a short note on how to effectively pip install requirements that
contain private repositories in a Dockerfile.

For the security reasons it is a good idea to generate an ssh key specificly
for the repository you want to pull with only read permissions. 
If you refuse to generate a new ssh key, you can transfer existing key through
local webserver and remove it after installing the requirements. Details of
this method can be read
[here](https://farazdagi.com/2016/using-ssh-private-keys-securely-when-building-docker-images/).
Generating new key
([more details](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/#generating-a-new-ssh-key)):

``` bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

After generation add newly generated public key to github repository by going
to repository settings, *Deploy Keys* sidebar, *Add deploy key* button
([more details](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys)).

Since the key has only read rights, it is safe to push it to your repository.
Next adapt your Dockerfile to copy the keys from the repository:
``` docker
WORKDIR /root
RUN mkdir /.ssh
ADD local_repo/keys/repo-github-deploy-key /root/.ssh/repo-github-deploy-key
ADD local_repo/keys/repo-github-deploy-key.pub /root/.ssh/repo-github-deploy-key.pub
```

Finally start the ssh agent, add keys, and pip install like usual:
``` docker
WORKDIR /code
ADD local_repo/requirements_frozen.txt requirements.txt
RUN \
  apt-get -qy upgrade git openssh-client && \
  chmod 600 /root/.ssh/repo-github-deploy-key && \
  eval $(ssh-agent) && \
  echo "Host github.com\n\tStrictHostKeyChecking no\n" >> /root/.ssh/config && \
  ssh-add /root/.ssh/repo-github-deploy-key && \
  ssh-keyscan github.com >> ~/.ssh/known_hosts && \
  pip3 install -r requirements.txt --src /usr/local/src
```

It is important to start the agent and add keys in the same `RUN` command,
otherwise it will fail.

If you never added github repository to you requirements here is an example
*requirements.txt*:
``` markdown
Django==1.10.6
django-allauth==0.24.1
# ...
-e git+ssh://git@github.com/username/private-repo-name.git@c0m1th4sh#egg=app-name
```

