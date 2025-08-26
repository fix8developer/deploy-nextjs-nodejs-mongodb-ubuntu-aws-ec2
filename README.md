# How to Deploy a Next.js/Node.js Application with MongoDB on an Ubuntu VPS (AWS-EC2)

## 1. Prepare Your Ubuntu VPS

Before deploying the application, it is important to properly set up and secure your Ubuntu VPS (AWS-EC2).
> Follow **steps 1–14** from the repository ([deploy-flask-postgresql-ubuntu-aws-ec2](https://github.com/fix8developer/deploy-flask-postgresql-ubuntu-aws-ec2.git)) to configure your server environment.

- These steps include updating system packages
- creating a non-root user
- configuring the firewall
- securing SSH access
- installing essential dependencies.

Completing this preparation ensures that your VPS is both stable and hardened against common security risks.

### Update System

Run to ensure all packages are up to date.

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Node.js and npm

Install Node.js and its package manager (npm) on your server. Using a tool like NVM ([Node Version Manager](https://nodejs.org/en/download)) is recommended for managing Node.js versions.

```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"

# Download and install Node.js:
nvm install 22

# Verify the Node.js version:
node -v # Should print "v22.18.0".
nvm current # Should print "v22.18.0".

# Verify npm version:
npm -v # Should print "10.9.3".
```

### Install PM2

Install PM2 globally to manage your Node.js application process, ensuring it runs continuously and restarts automatically.

```bash
npm install -g pm2
```

### Install MongoDB

Install MongoDB on your Ubuntu server. Ensure it's configured to bind to localhost for security unless you explicitly need remote access and have properly secured it.

1. Import MongoDB GPG Key

    ```bash
    curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
      sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
      --dearmor
    ```

2. Add MongoDB Repository

    ```bash
    echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
    https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/7.0 multiverse" | \
    sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
    ```

3. Install MongoDB

    ```bash
    sudo apt update
    sudo apt install -y mongodb-org
    ```

4. Start & Enable MongoDB

    ```bash
    sudo systemctl start mongod
    sudo systemctl enable mongod
    ```

5. Check status:

    ```bash
    systemctl status mongod
    ```

6. Bind MongoDB to localhost (for security)

    By default, MongoDB binds to `127.0.0.1`. To confirm:

    ```bash
    sudo nano /etc/mongod.conf
    ```

    Find:

    ```text
    net:
      bindIp: 127.0.0.1
    ```

    ⚠️ Only change this if you want remote access (and then use a firewall/VPN + authentication).

    Save and restart:

    ```bash
    sudo systemctl restart mongod
    ```

7. Verify Installation

    Open MongoDB shell:

    ```bash
    mongosh
    ```

    Test:

    ```javascript
    db.runCommand({ connectionStatus: 1 })
    ```

## 2. Prepare Your Application

### Build Next.js Application

On your local machine, build your Next.js application for production using `npm run build`. This generates the optimized static assets and server-side code.

### Clone Repository

On your VPS, clone your application's Git repository to a suitable directory (e.g., `/var/www/your-app-name`).

```bash
mkdir -p /var/www/
cd /var/www/
sudo git clone https://<Personal Access Token (PAT)>@github.com/Project-Buck/R3_Diagnostics_Website.git myapp
cd myapp
```

You should give your system user (e.g., "grader" or your actual username) ownership of the project directory

```bash
sudo chown -R grader:grader /var/www/myapp
```

### Install Dependencies

Navigate to your application's directory on the VPS and install the Node.js dependencies using npm install.

1. Navigate to your project directory

    ```bash
    cd /var/www/myapp
    ```

2. Install dependencies (if not installed yet)

    ```bash
    npm install
    ```

3. Run the production build

    ```bash
    npm run build
    ```

    This command creates an optimized `.next/` folder containing:
    - **Static assets** → for client-side usage.
    - **Server-side code** → for server rendering (if you’re not using static export).
    - **Optimized JavaScript & CSS**.

4. Start the production server (locally test)

    ```bash
    npm run start
    ```

⚡ At this stage, your app is ready for deployment

### Configure Environment Variables

Set up environment variables for your application, including your MongoDB connection string and any other necessary configurations. This can be done using a `.env` file or by setting them in your PM2 configuration.

1. Inside your **project root**, create a `.env` file:

    ```bash
    nano .env
    ```

2. Add your environment variables:

    ```text
    MONGODB_URI=mongodb://localhost:27017/myapp
    PORT=3000
    NODE_ENV=production
    SECRET_KEY=mySuperSecretKey
    ```

3. In Next.js, only environment variables prefixed with `NEXT_PUBLIC_` are exposed to the browser; all other variables remain server-side:

    ```text
    NEXT_PUBLIC_API_URL=https://mydomain.com/api
    NEXT_PUBLIC_SITE_URL=http://mydomain.com
    ```

4. Restart your app so the `.env` is loaded:

    ```bash
    cd /var/www/myapp
    rm -rf .next
    npm run build
    npm run start
    ```

## 3. Configure Nginx as a Reverse Proxy

### Install Nginx

Install Nginx on your VPS using `sudo apt install nginx`.

- Here’s how you can install Nginx on your Ubuntu VPS:

  ```bash
  # Update package lists
  sudo apt update

  # Install Nginx
  sudo apt install nginx -y

  # Enable Nginx to start on boot
  sudo systemctl enable nginx

  # Start Nginx service
  sudo systemctl start nginx

  # Check Nginx status
  sudo systemctl status nginx
  ```

  ✅ After installation, open your browser and go to your server’s IP address (e.g., `http://your_server_ip`). You should see the default “Welcome to Nginx” page.

### Create Nginx Configuration

Create a new Nginx configuration file for your application (e.g., in `/etc/nginx/sites-available/your-app-name`) and configure it to act as a reverse proxy, forwarding requests to your Next.js/Node.js application running on a specific port (e.g., 3000 or 8080).

1. Create a new configuration file

    ```bash
    sudo nano /etc/nginx/sites-available/myapp
    ```

2. Add the following configuration

    Replace `your_domain.com` with your actual domain name or server IP, and `3000` with the port your app is running on:

    ```text
    server {
        listen 80;
        server_name your_domain.com;

        location / {
            proxy_pass http://127.0.0.1:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
    ```

3. Clear old build & rebuild

    ```bash
    cd /var/www/myapp
    rm -rf .next
    npm run build
    npm run start
    ```

### Enable Nginx Configuration

Create a symbolic link from your configuration file in `sites-available` to `sites-enabled`.

- Link the file from sites-available to sites-enabled:

  ```bash
  # Go to Nginx config directory
  cd /etc/nginx/sites-available/

  # Create a symbolic link
  sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
  ```

### Test and Reload Nginx

Test your Nginx configuration with sudo nginx -t and reload Nginx with sudo systemctl reload nginx.

- Test the configuration (to ensure there are no syntax errors):

  ```bash
  sudo nginx -t
  ```

- If the test is successful, reload Nginx:

  ```bash
  sudo systemctl reload nginx
  ```

✅ Now, Nginx will forward all requests coming to `your_domain.com` to your Next.js app running on port `3000`.

## 4. Start and Manage Your Application with PM2

- From your application's directory, start your Next.js/Node.js application using PM2 (e.g., `pm2 start npm --name "your-app-name" -- run start`).

  1. Navigate to your app’s directory

      ```bash
      cd /var/www/myapp
      ```

  2. Start the app with PM2

      Run this command (replace `your-app-name` with your app name and in my case is `myapp`):

      ```bash
      pm2 start npm --name "myapp" -- run start
      ```

      Explanation:
      - `pm2 start npm`→ tells PM2 to run an npm command.
      - `--name "myapp"` → gives your process a readable name in PM2.
      - `-- run start` → executes your `"start"` script from package.json (usually `next start`).

- Save PM2 Process List: Use `pm2 save` to save the current process list, ensuring PM2 restarts your application automatically after a server reboot.

  1. Check if the app is running

      ```bash
      pm2 list
      ```

  2. View logs if needed

      ```bash
      pm2 logs myapp
      ```

  3. Keep app running after reboot

      ```bash
      pm2 save
      ```

- Enable PM2 Startup: Configure PM2 to start on system boot using the following command:
  
  ```bash
  pm2 startup systemd
  ```

  1. To setup the Startup Script, copy/paste the following command:

      You will get this command after running the above command according to your OS

      ```bash
      sudo env PATH=$PATH:/home/grader/.nvm/versions/node/v22.18.0/bin /home/grader/.nvm/versions/node/v22.18.0/lib/node_modules/pm2/bin/pm2 startup systemd -u grader --hp /home/grader
      ```

## 5. Secure Your Application (Optional but Recommended)

- Install Certbot: Install Certbot to easily obtain and manage SSL certificates from Let's Encrypt for HTTPS encryption.
- Obtain SSL Certificate: Use Certbot to obtain an SSL certificate for your domain and configure Nginx to use it.
- Automate Renewal: Configure Certbot to automatically renew your SSL certificates.

By following these steps, you can successfully deploy your Next.js and Node.js application with MongoDB on an Ubuntu VPS, ensuring it's accessible, secure, and running reliably.

## 📜 License

`deploy-nextjs-nodejs-mongodb-ubuntu-aws-ec2` is Copyright ©️ 2025 Kashif Iqbal. It is free, and may be redistributed under the terms specified in the [LICENSE](LICENSE) file.
