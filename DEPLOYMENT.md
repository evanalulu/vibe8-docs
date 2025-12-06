# Beginner's Deployment Guide (Google Cloud)

This guide will walk you through deploying the **Vibe8** application to a Google Cloud Virtual Machine (VM) from scratch. No prior experience is assumed!

---

## 📋 Prerequisites

Before you start, make sure you have:

1.  **Google Cloud Account**: A project created (e.g., `vibe-8-474004`).
2.  **Auth0 Account**: For user login functionality.
3.  **Domain Name**: (Optional but recommended) e.g., `vibe8.me`.
4.  **Terminal**: Access to a command line (Terminal on Mac/Linux, PowerShell on Windows).

---

## 🚀 Step 1: Create the Virtual Machine (VM)

We need a server to run the application. We'll use Google Compute Engine.

1.  **Install Google Cloud CLI (if not installed)**:
    Follow instructions here: https://cloud.google.com/sdk/docs/install
    Then run `gcloud init` to log in.

2.  **Create the VM**:
    Run this command in your terminal to create a server named `vibe8-instance` with Docker pre-installed (using Container-Optimized OS is easier, but standard Ubuntu gives us more control. We will use **Ubuntu** here).

    ```bash
    gcloud compute instances create vibe8-instance3 \
        --project="vibe-8-474004" \
        --zone="us-central1-c" \
        --machine-type="e2-medium" \
        --image-family="ubuntu-2204-lts" \
        --image-project="ubuntu-os-cloud" \
        --tags="http-server,https-server"
    ```

    *   `e2-medium`: A cost-effective server size (2 vCPUs, 4GB RAM).
    *   `tags`: Allows web traffic (HTTP/HTTPS) to reach the server.

3.  **Allow Traffic**:
    Ensure the firewall allows access:
    ```bash
    gcloud compute firewall-rules create allow-http \
        --action=ALLOW \
        --rules=tcp:80,tcp:443 \
        --target-tags=http-server,https-server
    ```

---

## 🌐 Step 2: Configure Your Domain (Optional)

If you have a domain like `vibe8.me`:

1.  **Find your VM's IP address**:
    ```bash
    gcloud compute instances list
    ```
    Look for `EXTERNAL_IP` (e.g., `34.135.134.79`).

2.  **Update DNS**:
    Go to your domain registrar (GoDaddy, Namecheap, Google Domains, etc.).
    *   Create an **A Record**.
    *   **Host**: `@` (root)
    *   **Value**: Your VM's IP address (`34.135.134.79`).
    *   (Optional) Create a second **A Record** for `www` pointing to the same IP.

---

## 🛠️ Step 3: Prepare the Server

Now we need to set up the server software.

1.  **Connect to the VM**:
    ```bash
    gcloud compute ssh --zone "us-central1-c" "vibe8-instance3" --project "vibe-8-474004"
    ```

2.  **Install Docker & Git (Run these commands INSIDE the VM)**:
    ```bash
    # Update system
    sudo apt-get update
    
    # Install Docker
    sudo apt-get install -y docker.io docker-compose-plugin git
    
    # Enable Docker
    sudo systemctl enable --now docker
    ```

3.  **Create Project Directory**:
    ```bash
    sudo mkdir -p /opt/vibe8-project
    sudo chown -R $USER:$USER /opt/vibe8-project
    ```

---

## 📥 Step 4: Get the Code

We need the actual application code on the server so Docker can build it.

1.  **Clone the Repository (First Time Only)**:
    In your **SSH terminal** (on the VM):
    ```bash
    cd /opt/vibe8-project
    # Use HTTPS for public repos, or set up SSH keys for private ones.
    # Replace with your actual repo URL:
    git clone https://github.com/team-cli/vibe8.git .
    ```
    *Note: The `.` at the end tells git to clone into the *current* directory.*

2.  **Pull Latest Changes (Updates)**:
    If you already cloned it, just update it:
    ```bash
    cd /opt/vibe8-project
    git pull origin main
    ```

---

## 📦 Step 5: Configure & Deploy Files

Now that the code is there, we need to upload your sensitive configuration (which isn't in Git). **Open a NEW terminal window on your local computer**.

1.  **Prepare your `.env` file**:
    Make sure you have a `.env` file locally with the **production** values.
    *   `NEXT_PUBLIC_API_URL`: `https://vibe8.me` (or your IP)
    *   `AUTH0_BASE_URL`: `https://vibe8.me`
    *   **IMPORTANT**: Do **NOT** include `PORT=8000` in this file, or the Frontend will break.

2.  **Upload Config Files**:
    Run these from your local project folder:

    ```bash
    # Upload docker-compose.yml (if you have local changes not in git)
    gcloud compute scp docker-compose.yml vibe8-instance3:~/docker-compose.yml --zone "us-central1-c" --project "vibe-8-474004"

    # Upload Nginx Config
    gcloud compute scp nginx/nginx.conf vibe8-instance3:~/nginx.conf --zone "us-central1-c" --project "vibe-8-474004"

    # Upload Environment Variables
    gcloud compute scp .env vibe8-instance3:~/.env --zone "us-central1-c" --project "vibe-8-474004"
    ```

3.  **Move Files into Place (Back on the VM)**:
    Switch to your **SSH terminal**:

    ```bash
    # Create Nginx directory
    mkdir -p /opt/vibe8-project/nginx

    # Move files (overwrite defaults)
    mv ~/docker-compose.yml /opt/vibe8-project/
    mv ~/nginx.conf /opt/vibe8-project/nginx/
    mv ~/.env /opt/vibe8-project/
    ```

---

## 🚀 Step 6: Start the Application

1.  **Start Docker Compose**:
    
    ```bash
    cd /opt/vibe8-project
    
    # Pull latest images and build
    sudo docker compose up -d --build
    ```

2.  **Check Status**:
    Wait a minute or two, then run:
    ```bash
    sudo docker compose ps
    ```
    You should see 5 services:
    *   `backend`
    *   `frontend`
    *   `worker`
    *   `db` (Postgres)
    *   `redis`
    *   `nginx`

    All columns should say "Up".

---

## ✅ Step 7: Verify

1.  **Open Browser**: Go to `http://vibe8.me` (or your IP).
2.  **Login**: Try to log in.
3.  **Test**: Upload a file or run an analysis.

---

## ❓ Troubleshooting

### "502 Bad Gateway" on Login
*   **Cause**: Auth0 headers are too big for Nginx default settings.
*   **Fix**: Ensure `nginx.conf` has increased buffer sizes:
    ```nginx
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    ```

### "Unknown Task" or Frontend Issues
*   **Cause**: Old code or browser cache.
*   **Fix**:
    1.  Hard refresh your browser (Cmd+Shift+R).
    2.  Rebuild frontend:
        ```bash
        sudo docker compose build --no-cache frontend
        sudo docker compose up -d frontend
        ```

### Database Crash (Restarting Loop)
*   **Cause**: Version mismatch (e.g., using Postgres 16 data with Postgres 15 image).
*   **Fix**: Update `docker-compose.yml` to use `postgres:16-alpine`.

### Login 404 Not Found
*   **Cause**: Nginx routing `/api/auth/` to Backend instead of Frontend.
*   **Fix**: Ensure `nginx.conf` has a specific block for `/api/auth/` pointing to `frontend:3000`.
