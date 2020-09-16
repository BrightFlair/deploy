RSync deployments via Github Actions
====================================

This Github Action deploys a repository to a remote server using rsync. There is nothing exciting about it.

Basic concept:

1. Set up your server for deployment
2. Configure how deployments execute
3. Add the action to your project's workflow
4. Push code to deploy


## 1. Set up your server for deployment

The server where deployment will be made needs to be set up. The following bash script will create a `deploy` user, generate an RSA key and authorise it for use with incoming SSH connections. The deploy user will be granted the privilege of running the single `post-deploy.bash` script as root.

```bash
#!/bin/bash
useradd deploy
usermod --shell /bin/bash deploy
mkdir -p /home/deploy/.ssh
openssl genrsa -out /home/deploy/.ssh/deploy.pem 4096
ssh-keygen -y -f /home/deploy/.ssh/deploy.pem >> /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy
chmod g-r,o-r /home/deploy/.ssh/*
echo "%deploy ALL=(ALL) NOPASSWD:/home/deploy/post-deploy.bash" > /etc/sudoers.d/100-deploy
chmod 0440 /etc/sudoers.d/100-deploy
echo "#This file should be called after rsync has completed to move the files to their correct place on disk" > /home/deploy/post-deploy.bash
chmod +x /home/deploy/post-deploy.bash
```

## 2. Configure how deployments execute

The `post-deploy.bash` script should be used to configure where the newly-deployed files will be placed on the server. This script is the only command that can be executed as root by the `deploy` user, and it is read-only to non-root users.

This example `post-deploy.bash` can be used to move incoming deployments to their directory name within `/var/www`, deploying staging branches to their own directories and finally calling a project-specific deploy script that can handle database migrations, etc.

```bash
#!/bin/bash
set -e
PROJECT_NAME=$1
BRANCH_NAME=$2
SOURCE_DIR=/home/deploy/$PROJECT_NAME
DEPLOY_DIR=/var/www/$PROJECT_NAME

# Remove any old backup directory
rm -rf "$DEPLOY_DIR.old"
# Backup the current deploy directory
mv "$DEPLOY_DIR" "$DEPLOY_DIR.old"
# Move the new deploy directory in place
mv "$SOURCE_DIR" "$DEPLOY_DIR"
# Own directory by web server's user
chown -R www-data:www-data "$DEPLOY_DIR"
```

## 3. Add the action to your project's workflow

## 4. Push code to deploy
