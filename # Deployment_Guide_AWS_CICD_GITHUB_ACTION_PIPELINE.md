# Deployment Guide

This guide covers the steps required to set up and deploy the Role-Permission-API project using Node.js, PM2, Nginx, AWS, EC2 and GitHub Actions.

## Prerequisites

- A server running Ubuntu
- Access to the server via SSH
- A GitHub repository for the project

## Server Setup
EC2 Instance Setup
Create an EC2 Instance on AWS and connect to it using SSH.
Edit the Security Group Inbound Rules to allow HTTP (port 80) and HTTPS (port 443) access from everywhere.
After logging to server 
GitHub Actions Runner Setup
Create a Directory for the Runner:

```
mkdir folder-name && cd folder-name 
```
and so on, all other commands listed on github account to create runner

### 0. Add new user

```
adduser username 
usermod -aG sudo username 
su - username

```

### 1. Update and Install Required Packages

```
sudo apt update
sudo apt upgrade
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - 
sudo apt install -y nodejs
sudo apt install npm
```

2. Verify Node.js and npm Installation
```
node --version
npm --version
```

1. Install PM2 Globally
```
sudo npm install pm2 -g
```

1. Install and Configure Nginx
```
sudo apt install nginx
sudo nginx -t
sudo systemctl start nginx
```

5. Configure Nginx
Create a new configuration file for your API:

```
sudo nano /etc/nginx/sites-available/role-permission-api
```
Add the following configuration:

```
server {
    listen 80;
    server_name 3.95.8.112;

    location /api/ {
        proxy_pass http://localhost:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### For seprate port 

```

# Frontend configuration
server {
    listen 8080;
    server_name 107.20.26.187;

    location / {
        root /var/www/html/role-permission-client-build;
        try_files $uri /index.html;
    }
}

# Backend configuration
server {
    listen 8081;
    server_name 107.20.26.187;

    location / {
        proxy_pass http://localhost:9000;  # Proxy requests to Node.js backend running on port 9000
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Proxying WebSocket and API requests
    location /socket.io/ {
        proxy_pass http://localhost:8000;  # Replace with your backend server's URL
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;  # Required for WebSocket connection
        proxy_set_header Connection "upgrade";  # Required for WebSocket connection
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Optional: Increase timeout for WebSocket
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}

```
Enable the new configuration:

```
sudo ln -s /etc/nginx/sites-available/role-permission-api /etc/nginx/sites-enabled/
```

Test and restart Nginx:

```
sudo nginx -t
sudo systemctl restart nginx
```

6. Start the Application with PM2
  
```
pm2 start server.js --name "role-permission-api"
pm2 save
```


1. Additional Setup ( After creating Runner and this will be last step of runner configure ) 
If you have a service script:

```
sudo ./svc.sh install
sudo ./svc.sh start
```


Deployment Script
Create a deploy.sh script for deployment:

```

#!/bin/bash

# Ensure the script exits if any command fails
set -e

echo "Deployment started ..."

# Check PM2 Process
if pm2 list | grep -q 'role-permission-api'; then
  echo "Process role-permission-api is running"
  echo "Restarting role-permission-api"
  pm2 restart role-permission-api --update-env || { echo "Failed to restart process"; exit 1; }
else
  echo "Process role-permission-api is not running"
  echo "Starting role-permission-api"
  # Start the server using the server.js file
  pm2 start server.js --name "role-permission-api" || { echo "Failed to start process"; exit 1; }
fi

# Save PM2 status to synchronize processes
pm2 save || { echo "Failed to save PM2 process list"; exit 1; }

# Save PM2 status
pm2 status


# Restart Nginx only if the deployment is successful
echo "Restarting Nginx ..."

sudo systemctl restart nginx

# Or Read the password from the file and restart Nginx

# sudo_pass=$(<~/.sudo_pass)
# echo $sudo_pass | sudo -S systemctl restart nginx || { echo "Failed to restart Nginx"; exit 1; }

echo "Deployment completed successfully!"

```

 Make it executable when create on local machine then in root folder open terminal and run this command

```
chmod +x deploy.sh
```

#### GitHub Actions for CI/CD
#### Create a GitHub Actions workflow file .github/workflows/node.js.yml:

```

name: Node.js CI/CD

on:
  push:
    branches: [ "master" ]
  pull_request:
    branches: [ "master" ]

jobs:
  # Job to pull latest changes
  pull_latest_changes:
    runs-on: self-hosted
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        clean: false

    - name: Pull latest changes
      run: |
        git reset --hard
        git checkout master
        git pull origin master

  # Job to install dependencies
  install_dependencies:
    runs-on: self-hosted
    needs: pull_latest_changes  # This job depends on the 'pull_latest_changes' job

    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        clean: false

    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'

    - name: Install dependencies
      run: npm ci  # 'npm ci' is faster for CI environments

  # Main build and deploy job
  deploy:
    runs-on: self-hosted
    needs: install_dependencies  # This job depends on the 'install_dependencies' job

    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        clean: false

    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'

    - name: Run deployment script
      run: |
            chmod +x ./deploy.sh
            ./deploy.sh

```
### For REACT JS 

```
name: React CI/CD

on:
  push:
    branches: [ "master" ]
  pull_request:
    branches: [ "master" ]

jobs:
  pull_latest_changes:
    runs-on: self-hosted

    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        clean: false

    - name: Pull latest changes
      run: |
        git reset --hard
        git checkout master
        git pull origin master

  install_dependencies:
    runs-on: self-hosted
    needs: pull_latest_changes

    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        clean: false

    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

  build_and_deploy:
    runs-on: self-hosted
    needs: install_dependencies

    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        clean: false

    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'

    - name: Build project
      run: npm run build --if-present

    - name: Deploy using deploy.sh script
      run: |
            chmod +x ./deploy.sh
            ./deploy.sh
```
###### If create-react-app  then folder name "build" if react-vite the folder name "dist"
sudo cp -r dist or build  /var/www/html/role-permission-client-build


If dis then in package.json where scripts build 
"build": "GENERATE_SOURCEMAP=false vite build",

```
#!/bin/bash

# Check if the build directory exists
if [ -d "dist" ]; then
  echo "Build directory exists."
else
  echo "Build directory does not exist." >&2
  exit 1
fi

# Remove the existing build directory if it exists
if [ -d "/var/www/html/role-permission-client-build" ]; then
  sudo rm -rf /var/www/html/role-permission-client-build
  echo "Old build directory removed."
else
  echo "No existing build directory to remove."
fi

# Copy the new build directory to the deployment location
sudo cp -r dist /var/www/html/role-permission-client-build
echo "New build directory copied to /var/www/html/role-permission-client-build."

# Restart Nginx only if the deployment is successful
echo "Restarting Nginx ..."

sudo systemctl restart nginx

# Or Read the password from the file and restart Nginx

# sudo_pass=$(<~/.sudo_pass)
# echo $sudo_pass | sudo -S systemctl restart nginx || { echo "Failed to restart Nginx"; exit 1; }

echo "Deployment completed successfully!"


# Deployment completed message
echo "Deployment completed successfully."

```
```
#!/bin/bash

# Check if the build directory exists
if [ -d "dist" ]; then
  echo "Build directory exists."
else
  echo "Build directory does not exist." >&2
  exit 1
fi

# Remove the existing build directory if it exists
if [ -d "/var/www/html/client-build" ]; then
  echo "Removing old build directory..."
  sudo_pass=$(<~/.sudo_pass)
  echo $sudo_pass | sudo -S rm -rf /var/www/html/client-build
  echo "Old build directory removed."
else
  echo "No existing build directory to remove."
fi

# Copy the new build directory to the deployment location
echo "Copying new build directory..."
echo $sudo_pass | sudo -S cp -r dist /var/www/html/client-build
echo "New build directory copied to /var/www/html/client-build."

# Restart Nginx to apply changes
# Read the password from the file and restart Nginx
echo "Restarting Nginx service..."
sudo_pass=$(<~/.sudo_pass)
echo $sudo_pass | sudo -S systemctl restart nginx || { echo "Failed to restart Nginx"; exit 1; }

# Deployment completed message
echo "Deployment completed successfully."

```
#### OR ( If Permission issue  )
```
#!/bin/bash

# Ensure the sudo password file has correct permissions
chmod 600 ~/.sudo_pass  # Ensure the file is only readable by the owner

# Read the sudo password from the file
sudo_pass=$(<~/.sudo_pass)

# Check if the build directory exists
if [ -d "dist" ]; then
  echo "Build directory exists."
else
  echo "Build directory does not exist." >&2
  exit 1
fi

# Remove the existing build directory if it exists
if [ -d "/var/www/html/admin-build" ]; then
  echo "Removing old build directory..."
  echo "$sudo_pass" | sudo -S rm -rf /var/www/html/admin-build || { echo "Failed to remove old build"; exit 1; }
  echo "Old build directory removed."
else
  echo "No existing build directory to remove."
fi

# Copy the new build directory to the deployment location
echo "Copying new build directory..."
echo "$sudo_pass" | sudo -S cp -r dist /var/www/html/admin-build || { echo "Failed to copy new build"; exit 1; }
echo "New build directory copied to /var/www/html/admin-build."

# Restart Nginx to apply changes
echo "Restarting Nginx service..."
echo "$sudo_pass" | sudo -S systemctl restart nginx || { echo "Failed to restart Nginx"; exit 1; }

# Deployment completed message
echo "Deployment completed successfully."

```

### OR 
```
#!/bin/bash

# Password file ka path
PASSWORD_FILE=~/.sudo_pass

# Check agar password file exist karti hai aur password ko ek baar read karein
if [ -f "$PASSWORD_FILE" ]; then
  sudo_pass=$(<"$PASSWORD_FILE")
else
  echo "Password file $PASSWORD_FILE exist nahi karti." >&2
  exit 1
fi

# Check if the build directory exists
if [ -d "dist" ]; then
  echo "Build directory exists."
else
  echo "Build directory does not exist." >&2
  exit 1
fi

# Remove the existing build directory if it exists
if [ -d "/var/www/html/client-build" ]; then
  echo "Removing old build directory..."
  echo "$sudo_pass" | sudo -S rm -rf /var/www/html/client-build
  echo "Old build directory removed."
else
  echo "No existing build directory to remove."
fi

# Copy the new build directory to the deployment location
echo "Copying new build directory..."
echo "$sudo_pass" | sudo -S cp -r dist /var/www/html/client-build
echo "New build directory copied to /var/www/html/client-build."

# Restart Nginx to apply changes
echo "Restarting Nginx service..."
echo "$sudo_pass" | sudo -S systemctl restart nginx || { echo "Failed to restart Nginx"; exit 1; }

# Deployment completed message
echo "Deployment completed successfully."

```
### Additional step if required 

### Check UFW Status and Allow Ports:
```
sudo ufw status
sudo ufw allow 22    # Allow SSH
sudo ufw allow 80    # Allow HTTP
sudo ufw allow 443   # Allow HTTPS
sudo ufw enable      # Enable UFW
sudo ufw status      # Check UFW status after enabling
sudo ufw reload      # Reload UFW to apply rules
sudo systemctl restart ssh  
sudo systemctl restart sshd
```

### Additional step if required 
##### If you face nginx restart problem and get an error to provide password during CI/CD
```
ssh <username>@your_server_ip

echo "your_password_here" > ~/.sudo_pass
chmod 600 ~/.sudo_pass  # Ensure the file is only readable by the owner

Then start nginx like this 
# Read the password from the file and restart Nginx
sudo_pass=$(<~/.sudo_pass)
echo $sudo_pass | sudo -S systemctl restart nginx || { echo "Failed to restart Nginx"; exit 1; }

```
```
if you want to check password the current user password like created user
cat ~/.sudo_pass
```
### Additional step if required for Passwordless sudo
```
sudo nano /etc/sudoers.d/sudoUser
sudoUser ALL=(ALL) NOPASSWD:ALL
sudo chmod u+w /etc/sudoers.d/sudoUser # To make it writable 
sudo chmod u-w /etc/sudoers.d/sudoUser # make it read-only
Test the Passwordless sudo
```
### Additional step if required  ( For SSL )
Install Certbot and it’s Nginx plugin

```
sudo apt install certbot python3-certbot-nginx
```


Verify Web Server Ports are Open and Allowed through Firewall

```
sudo ufw status verbose
```


Obtain an SSL certificate

```
sudo certbot --nginx -d your_domain.com -d www.your_domain.com -d api.your_domain.com -d admin.your_domain.com -d www.admin.your_domain.com
```

Check Status of Certbot

```
sudo systemctl status certbot.timer
```


Dry Run SSL Renewal

```
sudo certbot renew --dry-run
```

### Additional step if required 

##### SSH Key Generation for GitHub
Generate SSH keys for GitHub access:
First login to server

1. Check the Key File
Verify the Contents:
Ensure the SSH key file isn't corrupted by examining its contents:
```
cat ~/.ssh/id_rsa
```

If you see gibberish or the output is garbled, the key file might be corrupted.

2. Regenerate the SSH Key
If the key is corrupted, you’ll need to regenerate a new one:
1.	Generate a New SSH Key:
```
ssh-keygen -t rsa -b 4096 -C "your_custom_label"
```
2.	Ensure the Permissions Are Correct:
```
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

Update the SSH Key on GitHub
Copy the New Public Key:
```
cat ~/.ssh/id_rsa.pub
```
Add the Key to GitHub:

•	Go to GitHub SSH keys settings.
•	Add a new SSH key by pasting the contents of ~/.ssh/id_rsa.pub.

4. Add the SSH Key to the SSH Agent
Add the new SSH key to the SSH agent:
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```

5. Test the SSH Connection
Verify the SSH key is working by connecting to GitHub:
```
ssh -T git@github.com
```
You should see a success message, confirming that GitHub recognizes the SSH key.

##### SSH Key Generation for GitHub ( 2nd Method )

**Step 1: Generate a new SSH key**
```
ssh-keygen -t ed25519 -C "email@example.com" -f ~/.ssh/key-path

```
**Step 2: Display the public key (use this to add to GitHub)**
```
cat ~/.ssh/key-path.pub

```
**Step 3: Start the SSH agent**
```
eval "$(ssh-agent -s)"

```
**Step 4: Add the new key to the SSH agent**
```
ssh-add ~/.ssh/key-path

```
**Step 5: Edit the SSH config file to specify which key to use for GitHub**
```
nano ~/.ssh/config

```
**Inside the config file, add the following:** 

```
Host github.com
   HostName github.com
  User git
   IdentityFile ~/.ssh/key-path
```

**Step 6: Set appropriate permissions for the .ssh directory and the keys**

```
chmod 700 ~/.ssh              # Permissions for the .ssh folder
chmod 600 ~/.ssh/key-path      # Permissions for the private key
chmod 644 ~/.ssh/key-path.pub  # Permissions for the public key
```

**Step 7: Test the connection to GitHub**
```
ssh -T git@github.com

```


**You can list all users**
```
cat /etc/passwd | cut -d: -f1
``` 
6. Update Remote URL
Ensure your Git remote URL uses SSH:
```
git remote set-url origin git@github.com:<github-username>/<repo-name>.git
```


Final Notes
Make sure your server.js is properly configured to run on port 9000.
Ensure Nginx is correctly set up to proxy requests to your Node.js server.
Test the deployment script and GitHub Actions workflow to ensure everything is working as expected.



Feel free to adjust the file paths, URLs, and any other specifics to match your project's setup.
