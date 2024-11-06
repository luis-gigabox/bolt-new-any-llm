## Detailed Google Compute Engine Deployment Guide

### 1. Initial GCP Setup
1. Create a new GCP project or select an existing one
2. Enable the Compute Engine API
3. Install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
4. Authenticate with GCP:
   ```bash
   gcloud auth login
   gcloud config set project YOUR_PROJECT_ID
   ```

### 2. Create VM Instance
1. Create a new VM instance through GCP Console or CLI:
   ```bash
   gcloud compute instances create bolt-new-instance \
     --zone=us-central1-a \
     --machine-type=e2-medium \
     --image-family=ubuntu-2204-lts \
     --image-project=ubuntu-os-cloud \
     --tags=http-server,https-server \
     --boot-disk-size=200GB
   ```

2. Create firewall rules for HTTP/HTTPS:
   ```bash
   gcloud compute firewall-rules create allow-http \
     --allow tcp:80 \
     --target-tags http-server

   gcloud compute firewall-rules create allow-https \
     --allow tcp:443 \
     --target-tags https-server
   ```
### Assigning a Static IP Address

1. Using gcloud CLI:
   ```bash
   # Create a static IP address
   gcloud compute addresses create bolt-static-ip \
     --region=us-central1 \
     --network-tier=PREMIUM

   # Verify the static IP was created and note the address
   gcloud compute addresses list

   # Assign the static IP to your instance
   gcloud compute instances delete-access-config bolt-new-instance \
     --access-config-name="external-nat" \
     --zone=us-central1-a

   gcloud compute instances add-access-config bolt-new-instance \
     --access-config-name="external-nat" \
     --address=STATIC_IP_ADDRESS \
     --zone=us-central1-a
   
   gcloud compute instances add-access-config bolt-new-instance \
  --zone=us-central1-a \
  --access-config-name="external-nat" \
  --address=34.69.200.189  # Replace with your actual static IP
   
   ```
### 3. Configure the VM
1. SSH into your instance:
   ```bash
   gcloud compute ssh bolt-new-instance --zone=us-central1-a
   ```
```
# 1. First, clean up any existing Node.js installations
sudo apt-get remove nodejs npm -y
sudo apt-get purge nodejs npm -y
sudo apt-get autoremove -y

# 2. Clean the apt cache
sudo rm -rf /var/lib/apt/lists/*
sudo apt-get clean
sudo apt-get update

# 3. Remove problematic files
sudo rm /var/cache/apt/archives/nodejs_18.20.*
sudo rm /usr/share/systemtap/tapset/node.stp

# 4. Now try installing Node.js 18 again
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

2. Install required software:
   ```bash
   # Update system
   sudo apt update && sudo apt upgrade -y

   # Install Node.js and npm
   curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
   sudo apt install -y nodejs


   #Install npm
   #sudo apt install -y npm

   # Install pnpm
   sudo npm install -g pnpm

   # Install nginx
   sudo apt install -y nginx

   # Install certbot for SSL
   sudo apt install -y certbot python3-certbot-nginx
   ```

### 4. Deploy Application
1. Clone and setup the application:
   ```bash
   # Clone repository
   git clone https://github.com/coleam00/bolt.new-any-llm.git
   cd bolt.new-any-llm

   # Install dependencies
   pnpm install

   # Create .env file
   cp .env.example .env
   nano .env  # Add your API keys
   ```

2. Build the application:
   ```bash
   pnpm run build
   ```

3. Set up PM2 for process management:
   ```bash
   sudo npm install -g pm2
   pm2 start npm --name "bolt-new" -- start
   pm2 startup
   pm2 save
   ```

### 5. Configure Nginx
1. Create nginx configuration:
   ```bash
   sudo nano /etc/nginx/sites-available/bolt-new
   ```

2. Add the following configuration:
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;  # Replace with your domain

       location / {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;

           # WebSocket support
           proxy_set_header Connection "upgrade";
       }

       # Add security headers
       add_header X-Frame-Options "SAMEORIGIN";
       add_header X-XSS-Protection "1; mode=block";
       add_header X-Content-Type-Options "nosniff";
       add_header Referrer-Policy "no-referrer-when-downgrade";
       add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src 'self' data: https:; font-src 'self' data: https:;";
   }
   ```

   server {
       listen 80;
       # server_name your-domain.com;  # Replace with your domain

       location / {
           proxy_pass http://localhost:5173;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;

           # WebSocket support
           proxy_set_header Connection "upgrade";
       }

       # Add security headers
       add_header X-Frame-Options "SAMEORIGIN";
       add_header X-XSS-Protection "1; mode=block";
       add_header X-Content-Type-Options "nosniff";
       add_header Referrer-Policy "no-referrer-when-downgrade";
       add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src 'self' data: https:; font-src 'self' data: https:;";
   }
   ```

3. Enable the site:
   ```bash
   curl -s ifconfig.me
   sudo rm /etc/nginx/sites-enabled/default
   sudo ln -s /etc/nginx/sites-available/bolt-new /etc/nginx/sites-enabled/
   sudo nginx -t  # Test configuration
   sudo systemctl restart nginx
   ```

### 6. Set Up SSL with Let's Encrypt
1. Configure SSL certificate:
   ```bash
   sudo certbot --nginx -d your-domain.com
   ```

2. Auto-renewal setup:
   ```bash
   sudo systemctl status certbot.timer  # Verify auto-renewal is active
   ```

### 7. Monitoring and Maintenance

1. View application logs:
   ```bash
   pm2 logs bolt-new
   ```

2. Monitor system resources:
   ```bash
   pm2 monit
   ```

3. Update application:
   ```bash
   cd ~/bolt.new-any-llm
   git pull
   pnpm install
   pnpm run build
   pm2 restart bolt-new
   ```

### 8. Security Best Practices

1. Set up a static IP address:
   ```bash
   gcloud compute addresses create bolt-new-ip --region us-central1
   gcloud compute instances delete-access-config bolt-new-instance --zone=us-central1-a
   gcloud compute instances add-access-config bolt-new-instance --zone=us-central1-a --address=[IP_ADDRESS]
   ```

2. Configure UFW firewall:
   ```bash
   sudo ufw allow 22/tcp
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw enable
   ```

3. Store sensitive data in Secret Manager:
   ```bash
   # Enable Secret Manager API in GCP Console
   gcloud secrets create OPENAI_API_KEY --data-file="/path/to/api/key/file"
   ```

4. Set up monitoring:
   ```bash
   # Install monitoring agent
   curl -sSO https://dl.google.com/cloudagents/add-monitoring-agent-repo.sh
   sudo bash add-monitoring-agent-repo.sh
   sudo apt-get update
   sudo apt-get install stackdriver-agent
   ```

### Troubleshooting Tips

1. Check nginx error logs:
   ```bash
   sudo tail -f /var/log/nginx/error.log
   ```

2. Check application status:
   ```bash
   pm2 status
   pm2 logs
   ```

3. Test nginx configuration:
   ```bash
   sudo nginx -t
   ```

4. Verify SSL certificate:
   ```bash
   sudo certbot certificates
   ```