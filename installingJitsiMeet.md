## Setting Up a Jitsi Meet Server on Ubuntu

Jitsi Meet is an open-source video conferencing solution that you can host on your own server. This guide will walk you through the steps to set up a Jitsi Meet server on Ubuntu. Whether you're a beginner or have some experience with Linux servers, this tutorial will help you get your own Jitsi Meet instance up and running.

### Prerequisites
- A server running Ubuntu (20.04 or later)
- A domain name pointing to your server's IP address
- Basic knowledge of using the terminal

### Step 1: Update Your Server

First, update your server to ensure all packages are up to date. Connect to your server via SSH and run:

```javascript
sudo apt update && sudo apt upgrade -y
```

### Step 2: Set the Hostname

Set the hostname for your server. Edit the /etc/hosts file:

```javascript
sudo nano /etc/hosts
```
Add the following line, replacing your_domain with your actual domain name:

```javascript
your server ssh and  your_domain
// 56.117.120.123   abc.com  do not prefix https://www.
```
Save the file and exit.

### Step 3: Install Node.js, NVM, and Git

- Install Git:

```javascript
sudo apt install git -y
```
- Install NVM (Node Version Manager):

```javascript
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
source ~/.bashrc
```

- Install Node.js using NVM:

```javascript
nvm install 16
nvm use 16
nvm alias default 16

```

- Verify the installations:

```javascript
node -v
npm -v
git --version
```

### Step 4: Add Jitsi Repository and Key

Add the Jitsi package repository and its GPG key:

```javascript
sudo curl https://download.jitsi.org/jitsi-key.gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/jitsi-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/" | sudo tee /etc/apt/sources.list.d/jitsi-stable.list
```

### Step 5: Install Jitsi Meet

Update your package list and install Jitsi Meet:

```javascript
sudo apt update
sudo apt install jitsi-meet -y
```

During the installation, you will be prompted to enter your domain name and choose the type of SSL certificate to use. Select "Let's Encrypt" to obtain a free SSL certificate. Provide your email address when prompted.

### Step 6: Configure Firewall

Open necessary ports for Jitsi Meet to work:

```javascript
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 10000/udp
sudo ufw allow 22/tcp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp
sudo ufw enable
```

### Step 7: Restart Jitsi Services

Restart Jitsi services to apply the changes:

```javascript
sudo systemctl restart prosody
sudo systemctl restart jicofo
sudo systemctl restart jitsi-videobridge2
```

### Troubleshooting Screens

### Domain Configuration

When configuring Jitsi, if you see a screen asking for the domain (as shown in the image), enter your domain name (your_domain) and press OK.

### SSL Certificate

If you see the screen for selecting an SSL certificate, choose "Let's Encrypt certificates" and proceed. Provide your email address when asked.

### Firewall Warning

If enabling the firewall shows a warning about disrupting SSH connections, type y to proceed.

### Join Meeting Screen

When you see the Jitsi Meet interface, enter a meeting name and click "Join meeting". If you face disconnection issues, ensure your firewall settings are correct and the Jitsi services are running.

### Customizing Jitsi Meet

- Removing Chat Panel

To hide the chat panel, modify the config.js file:

```javascript
sudo nano /etc/jitsi/meet/your_domain-config.js
```

Ensure the toolbarButtons array does not include 'chat':

```javascript
toolbarButtons: [
    'camera',
    'closedcaptions',
    'desktop',
    'download',
    'embedmeeting',
    'etherpad',
    'feedback',
    'filmstrip',
    'fullscreen',
    'hangup',
    'help',
    'highlight',
    'invite',
    'livestreaming',
    'microphone',
    'noisesuppression',
    'participants-pane',
    'profile',
    'raisehand',
    'recording',
    'security',
    'select-background',
    'settings',
    'shareaudio',
    'sharedvideo',
    'shortcuts',
    'stats',
    'tileview',
    'toggle-camera',
    'videoquality',
    'whiteboard'
],
```

Save the file and restart Jitsi services.

### Conclusion

By following these steps, you should have a fully functional Jitsi Meet server running on your own domain. This setup allows you to host secure and private video meetings. If you encounter any screens or prompts mentioned in the images, follow the provided instructions to proceed. Happy conferencing!