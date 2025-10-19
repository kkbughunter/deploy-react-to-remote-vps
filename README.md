
# React Vite Application Deployment Guide

This document outlines the process to deploy a React Vite TailwindCSS application from the GitHub repository (`https://github.com/kkbughunter/deploy-react-to-remote-vps`) to an Ubuntu 24.04 VPS (`203.57.85.21`) using GitHub Actions. The application is served via Apache on the `brooms.astraval.com` subdomain with SSL enabled via Let’s Encrypt.

## Prerequisites

- **VPS Details**:
  - IP: `203.57.85.21`
  - OS: Ubuntu 24.04
  - User: `deploy` (with SSH access on port 22)
  - Apache2 installed and configured
  - Open ports: 22 (SSH), 443 (HTTPS), 5153 (if needed for custom SSH)
  - Node.js (`v20.19.5`), npm (`10.8.2`), and npx installed (for local testing, optional)

- **GitHub Repository**:
  - URL: `https://github.com/kkbughunter/deploy-react-to-remote-vps`
  - Contains a React Vite TailwindCSS project
  - Build output: `dist/` folder

- **Apache Configuration**:
  - Config file: `/etc/apache2/sites-available/brooms.astraval.com-le-ssl.conf`
  - DocumentRoot: `/home/deploy/astraval-brooms`
  - SSL certificates: Let’s Encrypt (`/etc/letsencrypt/live/brooms.astraval.com`)

- **SSH Access**:
  - SSH key pair for the `deploy` user
  - GitHub Secrets configured for deployment

## Step 1: VPS Setup

### 1.1 Configure SSH Access
1. **Generate SSH Key Pair** (if not already done):
   ```bash
   ssh-keygen -t ed25519 -C "github-actions@astraval" -f ~/.ssh/github_actions
   ```
   - Creates `~/.ssh/github_actions` (private key) and `~/.ssh/github_actions.pub` (public key).
   - Skip passphrase for automation.

2. **Add Public Key to VPS**:
   Log in to the VPS as the `deploy` user:
   ```bash
   ssh deploy@203.57.85.21
   ```
   Add the public key to `~/.ssh/authorized_keys`:
   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   cat >> ~/.ssh/authorized_keys
   # Paste contents of github_actions.pub
   chmod 600 ~/.ssh/authorized_keys
   ```

3. **Test SSH Access**:
   From your local machine:
   ```bash
   ssh -i ~/.ssh/github_actions deploy@203.57.85.21
   ```
   Ensure login works without a password prompt.

### 1.2 Configure Sudo for Apache Restart
To allow the `deploy` user to restart Apache without a password:
1. Log in to the VPS:
   ```bash
   ssh deploy@203.57.85.21
   ```
2. Edit the sudoers file:
   ```bash
   sudo visudo
   ```
3. Add the following line:
   ```
   deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart apache2
   ```
4. Save and exit.
5. Test the command:
   ```bash
   sudo systemctl restart apache2
   ```
   Ensure it runs without prompting for a password.

### 1.3 Set Up Directory Permissions
Ensure the `deploy` user and Apache’s `www-data` group can access the deployment directory:
```bash
sudo chown -R deploy:www-data /home/deploy/astraval-brooms
sudo chmod -R 755 /home/deploy/astraval-brooms
sudo usermod -a -G deploy www-data
```

### 1.4 Configure Apache
1. **Update Apache Configuration**:
   Edit `/etc/apache2/sites-available/brooms.astraval.com-le-ssl.conf`:
   ```apache
   <IfModule mod_ssl.c>
   <VirtualHost *:443>
       ServerName brooms.astraval.com
       DocumentRoot /home/deploy/astraval-brooms
       <Directory /home/deploy/astraval-brooms>
           Options -Indexes +FollowSymLinks
           AllowOverride All
           Require all granted
           RewriteEngine On
           RewriteBase /
           RewriteRule ^index\.html$ - [L]
           RewriteCond %{REQUEST_FILENAME} !-f
           RewriteCond %{REQUEST_FILENAME} !-d
           RewriteRule . /index.html [L]
       </Directory>
       ErrorLog ${APACHE_LOG_DIR}/brooms_error.log
       CustomLog ${APACHE_LOG_DIR}/brooms_access.log combined
       SSLEngine on
       SSLCertificateFile /etc/letsencrypt/live/brooms.astraval.com/fullchain.pem
       SSLCertificateKeyFile /etc/letsencrypt/live/brooms.astraval.com/privkey.pem
       Include /etc/letsencrypt/options-ssl-apache.conf
   </VirtualHost>
   </IfModule>
   ```
   - The rewrite rules ensure React Router handles client-side routing.

2. **Enable Rewrite Module**:
   ```bash
   sudo a2enmod rewrite
   ```

3. **Enable Site**:
   ```bash
   sudo a2ensite brooms.astraval.com-le-ssl.conf
   ```

4. **Test Configuration**:
   ```bash
   sudo apache2ctl configtest
   ```
   Expected output: `Syntax OK`

5. **Restart Apache**:
   ```bash
   sudo systemctl restart apache2
   ```

### 1.5 Verify SSL Certificates
Ensure Let’s Encrypt certificates are valid:
```bash
sudo certbot certificates
```
Test auto-renewal:
```bash
sudo certbot renew --dry-run
```
Regenerate if needed:
```bash
sudo certbot --apache -d brooms.astraval.com
```

## Step 2: GitHub Repository Setup

### 2.1 Configure GitHub Secrets
In your GitHub repository (`https://github.com/kkbughunter/deploy-react-to-remote-vps`):
1. Go to **Settings > Secrets and variables > Actions > New repository secret**.
2. Add:
   - `SSH_PRIVATE_KEY`: Contents of `~/.ssh/github_actions` (private key).
   - `VPS_HOST`: `203.57.85.21`
   - `VPS_USER`: `deploy`
   - `VPS_DEPLOY_PATH`: `/home/deploy/astraval-brooms`

### 2.2 Verify Vite Configuration
Ensure `vite.config.js` (or `vite.config.ts`) sets the correct base path:
```javascript
export default {
  base: '/',
  // Other configurations
}
```
Confirm `package.json` has the build script:
```json
"scripts": {
  "build": "vite build"
}
```

## Step 3: GitHub Actions Workflow

Create or update `.github/workflows/deploy.yml` to split the build and deployment into two jobs:

```yaml
name: Deploy React App to VPS

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm install

      - name: Build the app
        run: npm run build
        env:
          VITE_BASE_URL: https://brooms.astraval.com

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: react-build
          path: dist/
          retention-days: 1

  deploy:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: react-build
          path: dist/

      - name: Deploy to VPS
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          VPS_HOST: ${{ secrets.VPS_HOST }}
          VPS_USER: ${{ secrets.VPS_USER }}
          VPS_DEPLOY_PATH: ${{ secrets.VPS_DEPLOY_PATH }}
        run: |
          sudo apt-get update && sudo apt-get install -y openssh-client
          mkdir -p ~/.ssh
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H $VPS_HOST >> ~/.ssh/known_hosts
          rsync -avz --delete -e "ssh -i ~/.ssh/id_ed25519" dist/ $VPS_USER@$VPS_HOST:$VPS_DEPLOY_PATH/
          ssh -i ~/.ssh/id_ed25519 $VPS_USER@$VPS_HOST "sudo systemctl restart apache2"
```

### Workflow Explanation
- **Trigger**: Runs on pushes to the `main` branch.
- **Build Job**:
  - Checks out the code.
  - Sets up Node.js 20.
  - Installs dependencies (`npm install`).
  - Builds the app (`npm run build`), producing the `dist/` folder.
  - Uploads `dist/` as an artifact.
- **Deploy Job**:
  - Depends on the `build` job.
  - Downloads the `dist/` artifact.
  - Installs SSH client.
  - Configures SSH key and known hosts.
  - Uses `rsync` to transfer `dist/` to `/home/deploy/astraval-brooms` on the VPS.
  - Restarts Apache to apply changes.

## Step 4: Deploy and Test

1. **Commit Workflow**:
   Add the `deploy.yml` file to `.github/workflows/`:
   ```bash
   git add .github/workflows/deploy.yml
   git commit -m "Add GitHub Actions workflow for build and deploy"
   git push origin main
   ```

2. **Monitor Workflow**:
   - Go to the **Actions** tab in your GitHub repository.
   - Verify that both `build` and `deploy` jobs complete successfully.

3. **Test the Website**:
   - Visit `https://brooms.astraval.com`.
   - Test multiple routes (e.g., `/about`, `/contact`) to confirm rewrite rules work.
   - Check Apache logs if issues occur:
     ```bash
     sudo tail -f /var/log/apache2/brooms_error.log
     sudo tail -f /var/log/apache2/brooms_access.log
     ```

## Step 5: Troubleshooting

- **Build Failures**:
  - Check logs for `npm install` or `npm run build` errors.
  - Ensure all dependencies are in `package.json`.

- **Deploy Failures**:
  - Verify SSH connectivity:
    ```bash
    ssh -i ~/.ssh/github_actions deploy@203.57.85.21
    ```
  - Confirm GitHub Secrets are correct.
  - Check VPS directory permissions:
    ```bash
    ls -l /home/deploy/astraval-brooms
    ```

- **Website Issues**:
  - **404 Errors**: Verify rewrite rules and `index.html` in `/home/deploy/astraval-brooms`.
  - **403 Forbidden**: Check permissions (`chown`, `chmod`).
  - **SSL Errors**: Validate certificates with `sudo certbot certificates`.

- **Apache Restart Issues**:
  - Ensure the sudoers configuration allows passwordless `systemctl restart apache2`.
  - Test manually:
    ```bash
    sudo systemctl restart apache2
    ```

## Step 6: Optional Enhancements

- **Custom SSH Port**:
  If SSH uses port 5153, update the `rsync` and `ssh` commands:
  ```yaml
  rsync -avz --delete -e "ssh -p 5153 -i ~/.ssh/id_ed25519" dist/ $VPS_USER@$VPS_HOST:$VPS_DEPLOY_PATH/
  ssh -p 5153 -i ~/.ssh/id_ed25519 $VPS_USER@$VPS_HOST "sudo systemctl restart apache2"
  ```

- **Environment Variables**:
  Add `.env.production` for production-specific variables and load them in the `build` job.

- **Backup**:
  Before `rsync --delete`, back up the existing `/home/deploy/astraval-brooms`:
  ```bash
  ssh -i ~/.ssh/id_ed25519 $VPS_USER@$VPS_HOST "cp -r $VPS_DEPLOY_PATH $VPS_DEPLOY_PATH.bak"
  ```

## Conclusion
This setup ensures automated deployment of your React Vite application to `https://brooms.astraval.com` via GitHub Actions. The split `build` and `deploy` jobs improve modularity, and the Apache configuration with rewrite rules supports React Router. The passwordless sudo configuration (`deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart apache2`) ensures seamless Apache restarts. Monitor logs and test thoroughly to confirm successful deployment.

</xaiArtifact>
