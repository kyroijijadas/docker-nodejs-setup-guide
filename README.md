# Docker & Node.js Easy Setup Guide

A beginner-friendly guide to running a full-stack project (Node.js + Express backend and a MySQL database) with Docker. It covers the essential files, terminal commands, and the live-sync setup so your server restarts the moment you save a file.

## Table of Contents

- [Requirements](#requirements)
- [1. Project Structure](#1-project-structure)
- [2. The Main Files](#2-the-main-files)
- [3. Terminal Commands](#3-terminal-commands)
- [4. Live Sync (Auto-Restart on Save)](#4-live-sync-auto-restart-on-save)
- [5. What NOT to Put on GitHub](#5-what-not-to-put-on-github)
- [6. Troubleshooting](#6-troubleshooting)

## Requirements

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose)
- [VS Code](https://code.visualstudio.com/) or any code editor
- Git

Check that Docker is installed:

```bash
docker --version
docker compose version
```

## 1. Project Structure

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

## 2. The Main Files

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

Create a copy of `.env` with fake values. This one **does** go on GitHub so other people know which variables they need:

```env
DB_HOST=db
DB_USER=root
DB_ROOT_PASSWORD=your_password_here
DB_NAME=your_database_name
PORT=3000
```

## 3. Terminal Commands

Run these commands in your terminal, in the folder that contains `docker-compose.yml`.

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

Open a terminal inside the backend container:

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

## 4. Live Sync (Auto-Restart on Save)

If you use Windows, file-change events from your Windows folder often don't reach the Linux container, so the server doesn't notice when you save. To make the server restart the moment you press **Save** in VS Code, you need two things:

### A. The Folder Link (in `docker-compose.yml`)

Link your local folder to the container so they share the exact same files. This is the `volumes` section already included in the backend service above:

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

### B. The Watcher (in `package.json`)

Sometimes the folder link needs a little push. Add `-L` (short for **Legacy Watch**) to your nodemon script. This forces nodemon to constantly check whether files changed:

```json
"scripts": {
  "dev": "nodemon -L server.js",
  "start": "node server.js"
}
```

Make sure nodemon is installed as a dev dependency:

```bash
npm install --save-dev nodemon
```

> **Tip:** For the best speed on Windows, keep your project inside the WSL 2 file system (for example `\\wsl$\Ubuntu\home\you\my-project`) instead of `C:\`.

## 5. What NOT to Put on GitHub

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

## 6. Troubleshooting

| Problem | Fix |
| --- | --- |
| `port is already allocated` | Another program is using that port. Stop it, or change the left number in `"3000:3000"` (for example `"3001:3000"`). |
| Backend can't connect to the database | Make sure `DB_HOST=db` (not `localhost`), and check `docker compose logs db`. |
| Code changes don't update | Check that the `volumes` link exists and that your script uses `nodemon -L`. |
| `Cannot find module` after installing a package | Run `docker compose up --build`. |
| Database won't start after changing the password | MySQL only reads the password on first creation. Run `docker compose down -v`, then `docker compose up --build` (this erases your data). |

## License

This project is licensed under the [MIT License](LICENSE). 
