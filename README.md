# Setting Up a Vue.js Application on a Server

This guide will walk you through the steps necessary to set up a Vue.js application on a server.

## Prerequisites

- A server with SSH access
- Basic knowledge of using SSH and terminal commands
- Node.js and npm installed on the server (refer to Step 2 for installation)
- Vue CLI installed on the server (refer to Step 3 for installation)
- Git installed on the server
- A terminal application (like Terminal on macOS/Linux or Command Prompt/PowerShell on Windows)

## Step 1: Connect to Your Server

1. Open your terminal (or Command Prompt on Windows).
2. Connect to your server using SSH:
    ```sh
    ssh user@your-server-ip
    ```

## Step 2: Install Node.js and npm

1. Update the package index:
    ```sh
    sudo apt update
    ```
2. Install Node.js and npm:
    ```sh
    sudo apt install -y nodejs npm
    ```

## Step 3: Install Vue CLI

1. Install Vue CLI globally:
    ```sh
    sudo npm install -g @vue/cli
    ```

## Step 4: Clone Your Vue.js Application Repository

1. Navigate to the directory where you want to clone your Vue.js application:
    ```sh
    cd /path/to/your/directory
    ```
2. Ensure Git is installed:
    ```sh
    sudo apt install -y git
    ```
3. Clone your application repository:
    ```sh
    git clone https://github.com/aleem-333/aries-vue-challenge.git
    ```

## Step 5: Build and Serve Your Vue.js Application

1. Navigate to your Vue.js application directory:
    ```sh
    cd aries-vue-challenge
    ```
2. Install project dependencies:
    ```sh
    npm install
    ```
3. Build your Vue.js application for production:
    ```sh
    npm run build
    ```
4. Install a simple HTTP server to serve your application:
    ```sh
    sudo npm install -g serve
    ```
5. Serve your application:
    ```sh
    serve -s dist
    ```

## Troubleshooting

- Ensure your server's firewall settings allow traffic on the necessary ports (SSH, HTTP).
- Check your Vue.js application logs for errors.
