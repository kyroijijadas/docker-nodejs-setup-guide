# Docker & Node.js Easy Setup Guide

A beginner-friendly guide that takes you from **downloading Docker** to running a full-stack project (Node.js + Express backend and a MySQL database) with live-sync, so your server restarts the moment you save a file.

## Table of Contents

- [1. Install Docker](#1-install-docker)
- [2. Open Your Terminal](#2-open-your-terminal)
- [3. Verify Docker Works](#3-verify-docker-works)
- [4. Project Structure](#4-project-structure)
- [5. The Main Files](#5-the-main-files)
- [6. Docker Commands](#6-docker-commands)
- [7. Live Sync (Auto-Detect Code Changes)](#7-live-sync-auto-detect-code-changes)
- [8. What NOT to Put on GitHub](#8-what-not-to-put-on-github)
- [9. Troubleshooting](#9-troubleshooting)

---

## 1. Install Docker

### Step 1: Check your computer

- Windows 10 (64-bit, version 22H2) or Windows 11
- Virtualization turned on in your BIOS (it is on by default on most computers)
- At least 4 GB of RAM (8 GB recommended)

### Step 2: Install WSL 2 (Windows only)

Docker Desktop on Windows runs on WSL 2 (Windows Subsystem for Linux). Open **PowerShell as Administrator** (right-click Start, then choose *Terminal (Admin)* or *PowerShell (Admin)*) and run:

```powershell
wsl --install
```

Restart your computer when it finishes.

Check that WSL is using version 2:

```powershell
wsl --status
```

### Step 3: Download Docker Desktop

1. Go to [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)
2. Click **Download for Windows** (or Mac / Linux, depending on your computer)
3. Run the downloaded installer (`Docker Desktop Installer.exe`)

### Step 4: Run the installer

1. Keep **Use WSL 2 instead of Hyper-V** ticked
2. Click **OK** and wait for the install to finish
3. Click **Close and restart** if asked

### Step 5: Start Docker Desktop

1. Open **Docker Desktop** from the Start menu
2. Accept the terms of service
3. You can skip signing in
4. Wait until the whale icon in the bottom-left says **Engine running** (green)

> **Important:** Docker Desktop must be open and running every time you use a `docker` command. If you see `error during connect` or `cannot connect to the Docker daemon`, Docker Desktop is not running yet.

> **Mac:** Open the downloaded `.dmg`, drag Docker to **Applications**, then open it. Pick **Apple Chip** or **Intel Chip** to match your Mac.

> **Linux:** Follow the official guide at [docs.docker.com/engine/install](https://docs.docker.com/engine/install/).

---

## 2. Open Your Terminal

You can use any of these. All of them work with Docker commands.

| Terminal | How to open it |
| --- | --- |
| **Command Prompt (CMD)** | Press `Win + R`, type `cmd`, press Enter |
| **PowerShell / Windows Terminal** | Press `Win + X`, then choose *Terminal* |
| **VS Code Terminal** | Open your project in VS Code, then press `` Ctrl + ` `` |

### Basic terminal commands you will need

Show where you are right now:

```bash
cd
```

List the files in the current folder (Windows CMD):

```bash
dir
```

Go into a folder:

```bash
cd my-project
```

Go back one folder:

```bash
cd ..
```

Create a new folder:

```bash
mkdir my-project
```

Clear the screen:

```bash
cls
```

> **Tip:** Docker commands only work in the folder that contains your `docker-compose.yml`. Use `cd` to get there first.

---

## 3. Verify Docker Works

Check the Docker version:

```bash
docker --version
```

Check the Docker Compose version:

```bash
docker compose version
```

Run Docker's test container. If you see `Hello from Docker!`, everything is working:

```bash
docker run hello-world
```

See every container on your computer:

```bash
docker ps -a
```

---

## 4. Project Structure

Create your project folder and open it in VS Code:

```bash
mkdir my-project
cd my-project
code .
```

Your project should end up looking like this:

```text
my-project/
├── Backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   └── server.js
├── docker-compose.yml
├── .env
├── .env.example
└── .gitignore
```

---

## 5. The Main Files

Every Docker project needs two important files:

| File | What it does |
| --- | --- |
| `Dockerfile` | The recipe for your backend. It sets up Node.js, copies your files, and installs your packages. |
| `docker-compose.yml` | The boss file. It connects the parts of your project together (for example, your Node.js backend and your MySQL database). |

### Dockerfile

Create `Backend/Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files first so Docker can cache the npm install step
COPY package*.json ./
RUN npm install

# Copy the rest of your code
COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

### .dockerignore

Create `Backend/.dockerignore` so Docker doesn't copy unnecessary files into the image:

```text
node_modules
npm-debug.log
.env
.git
```

### docker-compose.yml

Create `docker-compose.yml` in the main folder:

```yaml
services:
  backend:
    build: ./Backend
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      # Connects your local Backend folder directly to the container
      - ./Backend:/app
      # Keeps the container's own node_modules (stops Windows from overwriting them)
      - /app/node_modules

  db:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    ports:
      - "3306:3306"
    volumes:
      # Saves your database data so it survives restarts
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p${DB_ROOT_PASSWORD}"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  db_data:
```

### .env

Create `.env` in the main folder (this file is secret and never goes on GitHub):

```env
DB_HOST=db
DB_USER=root
DB_ROOT_PASSWORD=change_this_password
DB_NAME=mydatabase
PORT=3000
```

> **Important:** Inside Docker, your backend connects to the database using the **service name** (`db`), not `localhost`. That's why `DB_HOST=db`.

### .env.example

A copy of `.env` with fake values. This one **does** go on GitHub so other people know which variables they need:

```env
DB_HOST=db
DB_USER=root
DB_ROOT_PASSWORD=your_password_here
DB_NAME=your_database_name
PORT=3000
```

---

## 6. Docker Commands

Run these in your terminal, in the folder that contains `docker-compose.yml`.

### Starting the Project

Start the whole project. Your terminal shows a live feed of everything happening:

```bash
docker compose up
```

Start the project in the background. This lets you keep typing other commands in the same terminal:

```bash
docker compose up -d
```

Rebuild the images and start the project. **Always run this** if you edit your `Dockerfile`, change your `docker-compose.yml`, or install new npm packages:

```bash
docker compose up --build
```

Start the project and let Docker automatically detect code changes (see [Method 2](#method-2-docker-compose-watch) in section 7):

```bash
docker compose up --watch
```

### Stopping the Project

Safely turn off and remove the containers (your database data is kept):

```bash
docker compose down
```

> **Warning:** The command below turns everything off **and deletes your database data**. Only run it if you want to completely erase everything inside your database.

```bash
docker compose down -v
```

### Checking and Fixing Problems

See which containers are running:

```bash
docker compose ps
```

Show the error messages for your backend. If your Express server crashes, this tells you why:

```bash
docker compose logs backend
```

Follow the backend logs live (press `Ctrl + C` to stop watching):

```bash
docker compose logs -f backend
```

Quickly restart just your backend without touching the database:

```bash
docker compose restart backend
```

Open a terminal inside the backend container (type `exit` to leave):

```bash
docker compose exec backend sh
```

Open the MySQL command line inside the database container:

```bash
docker compose exec db mysql -u root -p
```

### Installing a New npm Package

Install the package inside the container, so it matches the Linux environment:

```bash
docker compose exec backend npm install express
```

Then rebuild so the image includes it:

```bash
docker compose up --build
```

### Cleaning Up Docker

Remove stopped containers, unused networks, and dangling images:

```bash
docker system prune
```

---

## 7. Live Sync (Auto-Detect Code Changes)

By default, Docker copies your code into the container once. If you edit a file afterwards, the container doesn't know, and you would have to stop and rebuild again and again.

Use **one** of the two methods below so the server restarts by itself every time you press **Save**.

| Method | Best for | How it works |
| --- | --- | --- |
| **Method 1: Volume + nodemon** | Beginners, works on every Docker version | Your local folder is shared with the container, and nodemon restarts the server |
| **Method 2: Docker Compose Watch** | Newer Docker Desktop (Compose 2.22+) | Docker itself watches your files and copies changes into the container |

> **Important:** Pick **one** method. Don't use both at the same time.

### Method 1: Volume + nodemon (Recommended)

#### Step 1: Install nodemon

```bash
npm install --save-dev nodemon
```

#### Step 2: Add the dev script (in `Backend/package.json`)

Add `-L` (short for **Legacy Watch**). On Windows, file-change events from your folder often don't reach the Linux container, and this flag forces nodemon to keep checking for saved files.

```json
"scripts": {
  "dev": "nodemon -L server.js",
  "start": "node server.js"
}
```

#### Step 3: Make the Dockerfile run the dev script

The last line of `Backend/Dockerfile` must be:

```dockerfile
CMD ["npm", "run", "dev"]
```

#### Step 4: Link your folder (in `docker-compose.yml`)

Share your local folder with the container so they use the exact same files:

```yaml
services:
  backend:
    build: ./Backend
    ports:
      - "3000:3000"
    volumes:
      # Connects your local Backend folder directly to Docker
      - ./Backend:/app
      # Stops Windows from breaking the Linux node_modules
      - /app/node_modules
```

#### Step 5: Start the project and watch the logs

Rebuild once so all the changes above are included:

```bash
docker compose up --build
```

Now open `Backend/server.js`, change anything, and press **Save**. In the terminal you should see nodemon restart by itself:

```text
[nodemon] restarting due to changes...
[nodemon] starting `node server.js`
```

If you started the project in the background with `-d`, watch the restart live with:

```bash
docker compose logs -f backend
```

### Method 2: Docker Compose Watch

Docker Compose can watch your files for you, with no shared volume needed.

#### Step 1: Check your Compose version

You need version **2.22 or newer** (`sync+restart` needs 2.23 or newer). Update Docker Desktop if yours is older.

```bash
docker compose version
```

#### Step 2: Add a `develop.watch` section (in `docker-compose.yml`)

Remove the `volumes:` lines from the backend service, then add this:

```yaml
services:
  backend:
    build: ./Backend
    ports:
      - "3000:3000"
    env_file:
      - .env
    develop:
      watch:
        # When you save a code file, copy it into the container
        - action: sync
          path: ./Backend
          target: /app
          ignore:
            - node_modules/
        # When package.json changes, rebuild the image to install the new packages
        - action: rebuild
          path: ./Backend/package.json
```

Keep `nodemon` in your `dev` script (see Method 1, Steps 1 to 3). Docker copies the file into the container, and nodemon restarts the server.

#### Step 3: Start with watch turned on

Start everything and watch for changes in one command:

```bash
docker compose up --watch
```

Or, if the project is already running in the background:

```bash
docker compose watch
```

Press `Ctrl + C` to stop watching.

### What Happens When I Change a File?

| What you changed | What happens | What you need to do |
| --- | --- | --- |
| A code file (`server.js`, routes, controllers) | The server restarts automatically | Nothing, just save |
| `package.json` (a new package) | Needs a new install | Run `docker compose up --build` (Method 2 does this for you) |
| `Dockerfile` or `docker-compose.yml` | Needs a rebuild | Run `docker compose up --build` |
| `.env` | Containers keep the old values | Run `docker compose up -d --force-recreate backend` |

### Still Not Updating?

Restart just the backend:

```bash
docker compose restart backend
```

Check the logs for errors:

```bash
docker compose logs backend
```

Rebuild everything from the images up:

```bash
docker compose up --build
```

> **Tip:** For the best speed on Windows, keep your project inside the WSL 2 file system (for example `\\wsl$\Ubuntu\home\you\my-project`) instead of `C:\`. File watching is much faster and more reliable there.

---

## 8. What NOT to Put on GitHub

Before you push your code, hide large folders and secret passwords. Create a file named exactly `.gitignore` in the main folder and put this inside:

```gitignore
# Hide the giant folder of downloaded packages
node_modules/

# Hide secret passwords and database usernames
.env

# Hide local database saves (only needed if you use a local folder instead of a Docker volume)
db_data/
```

Already pushed `.env` or `node_modules` by mistake? Remove them from Git tracking (this keeps the files on your computer):

```bash
git rm -r --cached node_modules
git rm --cached .env
git commit -m "Stop tracking node_modules and .env"
```

> **Note:** If you ever pushed real passwords to GitHub, change them. Deleting the file does not remove it from your Git history.

---

## 9. Troubleshooting

| Problem | Fix |
| --- | --- |
| `cannot connect to the Docker daemon` or `error during connect` | Docker Desktop isn't running. Open it and wait for **Engine running**. |
| `WSL 2 installation is incomplete` | Run `wsl --update` in PowerShell, then restart Docker Desktop. |
| `Virtualization must be enabled` | Turn on **Intel VT-x / AMD-V (SVM)** in your BIOS settings. |
| `port is already allocated` | Another program is using that port. Stop it, or change the left number in `"3000:3000"` (for example `"3001:3000"`). |
| Backend can't connect to the database | Make sure `DB_HOST=db` (not `localhost`), and check `docker compose logs db`. |
| Code changes don't update | Check that the `volumes` link exists and that your script uses `nodemon -L`. See [section 7](#7-live-sync-auto-detect-code-changes). |
| `Cannot find module` after installing a package | Run `docker compose up --build`. |
| Database won't start after changing the password | MySQL only reads the password on first creation. Run `docker compose down -v`, then `docker compose up --build` (this erases your data). |

---

## License

This project is licensed under the [MIT License](LICENSE).
