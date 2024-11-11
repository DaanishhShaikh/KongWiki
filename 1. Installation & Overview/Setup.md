# Advanced Installation Guide for Kong Gateway with PostgreSQL on Ubuntu

This guide provides step-by-step instructions for installing Kong Gateway with PostgreSQL as the database on an Ubuntu Server. It includes setting up PostgreSQL, configuring Kong to use PostgreSQL, and verifying the setup.

## Prerequisites

- **Ubuntu Server** (recommended version: 20.04 or later)
- **VM** (Virtual Machine) or bare-metal server environment
- **Sudo Privileges**: Ensure you have root or sudo access on your system.


## Step 1: Prepare the Installation Script

1. **Create a new file** to store the installation script. Open your terminal and type:

   ```bash
   nano kong_install.sh
   ```

2. **Copy the following script** into `kong_install.sh`:

   ```bash
   #!/bin/bash

   # Exit on any error
   set -e

   # Update and Upgrade System
   echo "Updating and upgrading system packages..."
   sudo apt update && sudo apt upgrade -y

   # Install PostgreSQL if not already installed
   if ! command -v psql > /dev/null; then
       echo "Installing PostgreSQL..."
       sudo apt install -y postgresql
       echo "PostgreSQL installed."
   else
       echo "PostgreSQL is already installed. Skipping installation."
   fi

   # Start and Enable PostgreSQL Service
   echo "Starting and enabling PostgreSQL service..."
   sudo systemctl start postgresql
   sudo systemctl enable postgresql

   # Create Database and User for Kong
   echo "Setting up PostgreSQL for Kong Gateway..."
   sudo -i -u postgres psql <<EOF
   DO
   \$do\$
   BEGIN
      IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = 'kong') THEN
         CREATE ROLE kong WITH LOGIN PASSWORD 'kongpassword';
      END IF;
   END
   \$do\$;

   CREATE DATABASE kong OWNER kong;
   ALTER USER kong WITH SUPERUSER;
   EOF
   echo "PostgreSQL database and user setup complete."

   # Add Kong Repository if not already added
   if [ ! -f /etc/apt/sources.list.d/kong.list ]; then
       echo "Adding Kong repository..."
       echo "deb [trusted=yes] https://download.konghq.com/gateway-3.x-ubuntu-focal all main" | sudo tee /etc/apt/sources.list.d/kong.list
   else
       echo "Kong repository is already added. Skipping this step."
   fi

   # Update package list and install Kong
   echo "Installing Kong Gateway..."
   sudo apt update
   sudo apt install -y kong-enterprise-edition

   # Copy and Edit Kong Configuration File
   echo "Configuring Kong Gateway to use PostgreSQL..."
   KONG_CONF="/etc/kong/kong.conf"
   if [ ! -f "$KONG_CONF" ]; then
       sudo cp /etc/kong/kong.conf.default "$KONG_CONF"
   fi

   # Set up Kong configuration for PostgreSQL
   sudo sed -i 's/^#database = .*/database = postgres/' "$KONG_CONF"
   sudo sed -i 's/^#pg_host = .*/pg_host = 127.0.0.1/' "$KONG_CONF"
   sudo sed -i 's/^#pg_port = .*/pg_port = 5432/' "$KONG_CONF"
   sudo sed -i 's/^#pg_user = .*/pg_user = kong/' "$KONG_CONF"
   sudo sed -i 's/^#pg_password = .*/pg_password = kongpassword/' "$KONG_CONF"
   sudo sed -i 's/^#pg_database = .*/pg_database = kong/' "$KONG_CONF"

   # Run Kong Database Migrations
   echo "Running Kong database migrations..."
   sudo kong migrations bootstrap -c "$KONG_CONF"

   # Start Kong Gateway
   echo "Starting Kong Gateway..."
   sudo kong start -c "$KONG_CONF"

   # Enable Kong to start on boot
   echo "Enabling Kong Gateway to start on boot..."
   sudo systemctl enable kong

   echo "Kong installation and setup completed successfully!"
   ```

3. **Save and exit** by pressing `CTRL + X`, then `Y` to confirm, and `Enter` to save.

---

## Step 2: Make the Script Executable

In the terminal, make the script executable:

```bash
chmod +x kong_install.sh
```

---

## Step 3: Run the Installation Script

Execute the script with sudo privileges:

```bash
sudo ./kong_install.sh
```

The script will:

- Update and upgrade the system packages.
- Install PostgreSQL if not already installed.
- Create a `kong` database and a `kong` user with appropriate privileges in PostgreSQL.
- Add the Kong repository and install Kong Gateway.
- Configure Kong to use PostgreSQL as its database.
- Run the necessary database migrations.
- Start Kong and enable it to run on system startup.

---

## Step 4: Verify Kong is Running

After the script completes, verify that Kong Gateway is up and running with:

```bash
curl -i http://localhost:8001
```

If Kong is set up correctly, this command will return Kong’s version and other configuration details.

## Explanation of Key Steps in the Script

### 1. System Update and Upgrade
Ensures that your system is updated with the latest packages, which reduces potential compatibility issues.

### 2. PostgreSQL Installation and Configuration
Installs PostgreSQL (if not already installed) and sets it up for Kong Gateway by creating a `kong` database and a `kong` user with the necessary permissions.

### 3. Adding the Kong Repository and Installing Kong
Adds the Kong APT repository to ensure you’re getting the latest stable version of Kong Gateway, then installs it.

### 4. Configuring Kong for PostgreSQL
Modifies Kong’s configuration file (`kong.conf`) to use PostgreSQL as the database. This setup includes specifying the database type, host, port, user, password, and database name.

### 5. Database Migrations
Runs the necessary migrations to initialize Kong's database structure.

### 6. Starting Kong Gateway and Enabling on Boot
Starts Kong Gateway and enables it to start on boot for continuous operation.

---

## Troubleshooting

- **Kong Fails to Start**: Check the logs with `tail -f /usr/local/kong/logs/error.log`.
- **PostgreSQL Connection Issues**: Verify that PostgreSQL is running (`sudo systemctl status postgresql`) and confirm the connection details in `kong.conf`.
- **Port Conflicts**: If another service is using port 8001, Kong might fail to start. Adjust the `admin_listen` directive in `kong.conf` to use a different port.