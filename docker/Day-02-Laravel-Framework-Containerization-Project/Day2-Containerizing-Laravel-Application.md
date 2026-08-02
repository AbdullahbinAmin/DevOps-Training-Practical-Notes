# Day 5: Containerizing a Laravel Application (Beginner Guide)

In this practical, students will:

- Clone the Laravel learning repository.
- Write a custom Dockerfile with PHP, Apache dependencies, and Composer.
- Build the Docker image.
- Run the Laravel application inside a Docker container.
- Push their lab code to GitHub.

---

## ✅ Prerequisites (Check Before Starting)

Make sure the tools from earlier days are installed and working:

```bash
docker --version      # Docker installed (Day 4)
git --version         # Git installed (Day 1)
docker ps             # Docker daemon is running (no permission error)
```

> If `docker ps` gives a **permission denied** error, run `sudo usermod -aG docker $USER`, then log out and back in (or run `newgrp docker`).

**About the repository:** We clone [`hanieas/Docker-Laravel`](https://github.com/hanieas/Docker-Laravel) purely as ready-made Laravel source code. The repo may already contain its own Docker files — that's fine. In this lab we write **our own beginner-friendly `Dockerfile`** and let it overwrite any existing one.

---

## 📌 Step 1: Create Folder & Clone Repository

Navigate to your Docker course folder and clone the GitHub repository:

```bash
# 1. Go to your local docker training folder
cd ~/devops-training-notes/docker

# 2. Create today's lab directory and go inside
mkdir Day-05-Laravel-App-Container
cd Day-05-Laravel-App-Container

# 3. Clone the Laravel source code from GitHub
git clone https://github.com/hanieas/Docker-Laravel.git .
```

**Why we run this:** Downloads the complete Laravel application source code so we can wrap it inside a Docker container.

**Word Breakdown:**

- `git clone`: Clones a remote repository to local machine.
- `.`: Tells Git to clone directly into the current directory instead of creating a subfolder.

---

## 📌 Step 2: Create the Dockerfile

Create a new file named `Dockerfile` in the root of the project:

```bash
nano Dockerfile
```

Paste the following beginner-friendly Dockerfile configuration:

```dockerfile
# 1. Base Image: PHP 8.2 with Apache Web Server
FROM php:8.2-apache

# 2. Set working directory inside container
WORKDIR /var/www/html

# 3. Install required system libraries and PHP extensions for Laravel
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    git \
    curl && \
    docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# 4. Enable Apache mod_rewrite for Laravel routing
RUN a2enmod rewrite

# 5. Install Composer (PHP package manager)
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# 6. Copy application code into the container
COPY . /var/www/html

# 7. Configure Apache DocumentRoot to point to Laravel's /public folder
ENV APACHE_DOCUMENT_ROOT /var/www/html/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/conf-available/*.conf

# 8. Install Laravel PHP dependencies via Composer
RUN composer install --no-dev --optimize-autoloader

# 9. Set permissions for Laravel storage and cache directories
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

# 10. Expose HTTP port 80
EXPOSE 80

# 11. Start Apache in foreground mode
CMD ["apache2-foreground"]
```

**Why we run this:** A Dockerfile serves as a step-by-step blueprint that packages PHP, Apache web server, Composer, Laravel dependencies, and application source code into a single portable image.

---

## 📌 Step 3: Configure Environment File (.env)

Laravel requires an `.env` file to set application keys and configurations.

```bash
# Copy the example environment file
cp .env.example .env
```

**Why we run this:** Creates the runtime configuration file `.env` required by Laravel to read settings like database credentials and environment variables.

---

## 📌 Step 4: Build the Docker Image

Build the Docker image using `docker build`:

```bash
docker build -t laravel-app:v1 .
```

**Why we run this:** Executes the instructions in the Dockerfile line by line to create a read-only Docker image named `laravel-app:v1`.

**Word Breakdown:**

- `docker build`: Compiles a Docker image from a Dockerfile.
- `-t laravel-app:v1`: Tags/names the image `laravel-app` with version `v1`.
- `.`: Specifies the build context (current directory).

> **Note:** The first build downloads the PHP base image and installs extensions, so it may take a few minutes. Later builds are faster thanks to Docker's layer cache.

---

## 📌 Step 5: Verify Image Creation

Check if the newly built image is present in local storage:

```bash
docker images
```

**Why we run this:** Confirms that `laravel-app:v1` was created successfully and shows its size and ID.

---

## 📌 Step 6: Run the Laravel Container

Run the Laravel application container in background (detached) mode:

```bash
docker run -d -p 8000:80 --name my-laravel-container laravel-app:v1
```

**Why we run this:** Starts an active container running Apache and Laravel, making it accessible on host port 8000.

**Word Breakdown:**

- `docker run`: Spins up a container from an image.
- `-d`: Detached mode (runs container in the background).
- `-p 8000:80`: Port forwarding (`HostPort:ContainerPort`).
- `--name my-laravel-container`: Assigns a readable name to the container instance.

---

## 📌 Step 7: Generate Laravel Application Key Inside Container

Laravel requires an encryption application key (`APP_KEY`) to run. Run `php artisan key:generate` inside the running container:

```bash
docker exec -it my-laravel-container php artisan key:generate
```

**Why we run this:** Generates a secure `APP_KEY` inside the `.env` file of the running container to enable sessions and password encryption.

**Word Breakdown:**

- `docker exec`: Executes a command inside a running container.
- `-it`: Interactive terminal mode.
- `php artisan key:generate`: Built-in Laravel CLI command to construct encryption keys.

---

## 📌 Step 8: Verify Application in Browser

Open your browser and navigate to:

```plaintext
http://localhost:8000
```

(If running on an AWS EC2 instance, use `http://<EC2-PUBLIC-IP>:8000` and ensure **Port 8000** is open in Security Groups.)

You should see the official **Laravel Welcome Page** live from inside your Docker container! 🎉

---

## 📌 Step 9: Verify Container Logs & Status

Check running container status and logs to ensure there are no PHP or Apache errors:

```bash
# Check container status
docker ps

# View container output logs
docker logs my-laravel-container
```

**Why we run this:** Confirms the container is in an `Up` state and lets you read Apache/PHP output to catch any startup errors early.

---

## 📌 Step 10: Push Your Lab Code to GitHub

Now save your work (Dockerfile + notes) to your **own** GitHub repository.

```bash
# 1. Point origin to YOUR new empty GitHub repo (replace the URL)
git remote remove origin
git remote add origin https://github.com/<your-username>/Day-05-Laravel-Docker.git

# 2. Stage, commit, and push your changes
git add Dockerfile .env.example
git commit -m "Day 5: Containerized Laravel app with custom Dockerfile"
git branch -M main
git push -u origin main
```

**Why we run this:** Publishes your lab work to your personal GitHub so it becomes part of your portfolio and version history.

**Word Breakdown:**

- `git remote remove origin`: Detaches the original cloned repo's remote so you don't push to someone else's project.
- `git remote add origin`: Links your local lab to **your** GitHub repository.
- `git push -u origin main`: Uploads commits and sets `main` as the default upstream branch.

> **Do not commit the real `.env` file** — it can contain secrets. Commit only `.env.example`. Add a `.gitignore` line `\.env` if it isn't already ignored.

---

## 🛠️ Troubleshooting (Common Beginner Errors)

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `Bind for 0.0.0.0:8000 failed: port is already allocated` | Port 8000 already in use | Use a different host port, e.g. `-p 8001:80`, or stop the other container. |
| Browser shows **500 / Permission denied** on storage | Storage/cache not writable | Re-run: `docker exec -it my-laravel-container chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache` |
| `No application encryption key has been specified` | `APP_KEY` missing | Re-run Step 7 (`php artisan key:generate`). |
| Composer killed / out of memory during build | Low RAM on instance | Add swap, or use a larger EC2 instance type. |
| Page loads but assets/routes 404 | `mod_rewrite` not active | Confirm Step 4 (`a2enmod rewrite`) ran in the Dockerfile; rebuild the image. |
| Can't reach app on EC2 | Security Group blocks port | Open inbound port `8000` for your IP (or `0.0.0.0/0` for testing). |

> **Note on the database:** The Laravel welcome page and `key:generate` work without a database. If a later exercise needs one, run a MySQL container and update the `DB_*` values in `.env` accordingly.

---

## 🧹 Cleanup (After the Lab)

Stop and remove the container and image to free resources when you're done:

```bash
# Stop and force-remove the container
docker rm -f my-laravel-container

# Remove the image
docker rmi laravel-app:v1

# Verify nothing is left running
docker ps -a
docker images
```

---

## 📋 Quick Command Reference

```bash
git clone <repo-url> .                                   # Clone into current folder
nano Dockerfile                                          # Create the Dockerfile
cp .env.example .env                                     # Prepare Laravel env file
docker build -t laravel-app:v1 .                         # Build image
docker run -d -p 8000:80 --name my-laravel-container laravel-app:v1   # Run container
docker exec -it my-laravel-container php artisan key:generate         # Generate APP_KEY
docker ps && docker logs my-laravel-container            # Verify
git push -u origin main                                  # Push lab to GitHub
docker rm -f my-laravel-container && docker rmi laravel-app:v1        # Cleanup
```
