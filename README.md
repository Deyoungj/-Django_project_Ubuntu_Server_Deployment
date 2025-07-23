# -Django_project_Ubuntu_Server_Deployment
A step by step tutorial on deploying a django project to an ubuntu server using nginx and docker


- Prerequisites: 
    - Create a requirements.txt file that contains all packages used in project:  
        - `pip freeze > requirements.txt`
    - Set up static files:
        - Install whitenoise:
            - `pip install whitenoise`
            - `pip freeze > requirements.txt`
        - Paste the following at the top of middleware list:
            - `"whitenoise.middleware.WhiteNoiseMiddleware",`
        - Add the following:
            ```python
            STATIC_URL = '/static/'
            STATIC_ROOT = BASE_DIR / "staticfiles"
            STATICFILES_DIRS = [
                os.path.join(BASE_DIR, 'static'),
            ]

            STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"
            ```
        - Collect static files:
            - `python manage.py collectstatic`


    - Set up Docker:
      - In your project root directory create a `Dockerfile` file :
      - Paste this in `Dockerfile` you've just created;
        
         ```python
            # Use a lightweight Python image
            FROM python:3.10-slim
            
            # Avoid .pyc files
            ENV PYTHONDONTWRITEBYTECODE=1
            ENV PYTHONUNBUFFERED=1
            
            # Set working directory
            WORKDIR /app
            
            # Install system dependencies
            RUN apt-get update \
              && apt-get install -y build-essential libpq-dev curl \
              && apt-get clean
            
            # Install Python dependencies
            COPY requirements.txt .
            # RUN pip install --upgrade pip && pip install -r requirements.txt
            RUN pip install --upgrade pip && pip install --no-cache-dir --root-user-action=ignore -r requirements.txt
            
            
            # Copy project files
            COPY . .
            
            
            
            # Expose port
            EXPOSE 8001
            
            RUN python manage.py collectstatic --noinput
            
            # Run Gunicorn
            CMD ["gunicorn", "tethersurge.wsgi:application", "--bind", "0.0.0.0:8000"]
         ```

      - create a `.env` file :
     
        ```python
            Other variables you want to add
            ........

        
            POSTGRES_DB=yourdbname
            POSTGRES_USER=user
            POSTGRES_PASSWORD=password
        ```
        
      - In the same directory create a `docker-compose.yml` file :
      - Paste this in `docker-compose.ym` you've just created;
        ```python
           
            services:
            
              web:
            
                container_name: name-web # add any name
            
                build:
            
                  context: .
                  dockerfile: Dockerfile
                  
                env_file:
                  - .env
                depends_on:
                  - db
                volumes:
                  - ./static:/app/static
                  - ./media:/app/media
            
                ports:
                  - "8000:8000"
                restart: always
            
                command: gunicorn tethersurge.wsgi:application --bind 0.0.0.0:8000
                
            
              db:
                image: postgres:14
                container_name: name-db # any your db name
                environment:
                  POSTGRES_DB: ${POSTGRES_DB}
                  POSTGRES_USER: ${POSTGRES_USER}
                  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
                volumes:
                  - postgres_data:/var/lib/postgresql/data/
                ports:
                  - "5432:5432"
                restart: always
            
            
            volumes:
              postgres_data:
              static_volume:
              media_volume:


        ```
          
    - Push to github repository
 
1. Ssh into server:
  - run in your terminal:
      - `ssh username@server_ip_address`
      - Enter password
      &nbsp;

2. Update & Upgrade Server:
    - `sudo apt-get update`
    - `sudo apt-get upgrade`
    &nbsp;

4. Create & activate virtual environment:
    - `virtualenv /opt/myproject`
    - `source /opt/myproject/bin/activate`
    &nbsp;

4. install docker:
   
    - Install Required Dependencies
        - `sudo apt install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release -y`

    - Add Docker’s Official GPG Key
        - `sudo mkdir -p /etc/apt/keyrings`
        - `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
          sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`

    - Add Docker Repository
        - `echo \
          "deb [arch=$(dpkg --print-architecture) \
          signed-by=/etc/apt/keyrings/docker.gpg] \
          https://download.docker.com/linux/ubuntu \
          $(lsb_release -cs) stable" | \
          sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

     - Install Docker Engine
         - `sudo apt update`
         - `sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y`
      
     -  (Optional) Run Docker Without `sudo`
          - `sudo usermod -aG docker $USER`
          - `newgrp docker`
           
      - Verify Docker Compose
          - `docker compose version`
    &nbsp;

4. Create & activate virtual environment:
    - `python3 -m venv venv`
    - `source /venv/bin/activate`
    &nbsp;

5. Clone Repository:
    - `cd /opt/myproject`
    - `mkdir myproject`
        - It may seem redundant to have two directories with the same name; however, it makes it so that your virtualenv name and project name are the same.
    - `cd myproject`
   - check if git is installed:
        - `git status`
        - `sudo apt-get install git`
    - Clone repo:
        `git clone repo-url`
    - install requirements:
        - `cd repo-name`
        - `pip install -r requirements.txt`
      &nbsp;

6. Start docker:
    - `docker-compose up -d --build`
    &nbsp;

7. Confirm It’s Running:
    - `docker ps`
      - You should now see:
        - `0.0.0.0:8000->8000/tcp`

       - Then:
        - `curl http://127.0.0.1:8000`
        - You should see your Django app response (HTML or redirect).
    &nbsp;

8. Start Nginx:
    - `sudo systemctl start nginx`
    &nbsp;

9. Nginx Configuration:
    
    - Install nginx:
        - `sudo apt install nginx`
    - `sudo nano /etc/nginx/sites-available/myproject.conf`
    - Type the following into the file:
      
         ```nginx
              server {
                  listen 80;
                  server_name yourip;

                  access_log /var/log/nginx/website-name.log;
              
                  location / {
                      proxy_pass http://127.0.0.1:8001;
                      proxy_set_header Host $host;
                      proxy_set_header X-Real-IP $remote_addr;
                      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                      proxy_set_header X-Forwarded-Proto $scheme;
                  }
              
                  location /static/ {
                      alias /home/ubuntu/project/static/;
                      autoindex on;
                  }
              
                  location /media/ {
                      alias /home/ubuntu/project/media/;
                      autoindex on;
                  }
              }
         ```
    - Now we need to set up a symbolic link in the /etc/nginx/sites-enabled directory that points to this configuration file. That is how NGINX knows this site is   active. Change directories to /etc/nginx/sites-enabled like this:
        - `sudo ln -s /etc/nginx/sites-available/tethersurge /etc/nginx/sites-enabled/`


    - Go to:
        - `sudo nano /etc/nginx/nginx.conf`
        - uncomment this line:
            - `# server_names_hash_bucket_size 64;`

    - Restart nginx:
        - `sudo nginx -t`
        - `sudo service nginx restart`
        
    
    &nbsp;

10. Adjusting the Firewall

    - Before testing Nginx, the firewall software needs to be adjusted to allow access to the service. Nginx registers itself as a service with ufw upon installation, making it straightforward to allow Nginx access.

    - `sudo apt-get install ufw`
    - `sudo ufw allow 8000`

    - Restart nginx:
        - `sudo service nginx restart`
    &nbsp;

11. Connecting domain:
    - open your domain registrar and open your dns settings for the specified domain

    - Add an A record with name @, pointing to the ip address of the server you are hosting your project on

    - Save changes

    - Open your nginx configuration file:
        - `sudo nano /etc/nginx/sites-available/myproject`

    - Make the following changes:

        ```nginx
                server {
                  listen 80;
                  server_name yourdomain.com www.yourdomain.com;

                  access_log /var/log/nginx/website-name.log;
              
                  location / {
                      proxy_pass http://127.0.0.1:8001;
                      proxy_set_header Host $host;
                      proxy_set_header X-Real-IP $remote_addr;
                      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                      proxy_set_header X-Forwarded-Proto $scheme;
                  }
              
                  location /static/ {
                      alias /home/ubuntu/project/static/;
                      autoindex on;
                  }
              
                  location /media/ {
                      alias /home/ubuntu/project/media/;
                      autoindex on;
                    }
                }
        ```
    - Restart nginx:
        - `sudo service nginx restart`
        
    - Wait for dns changes to propagate and check your website on your connected domain
    &nbsp;


12. Install SSL on vps with Let's Encrypt (Requires domain):

    - `sudo apt update`
    - `sudo apt install certbot python3-certbot-nginx -y`
    - run the following and follow the process:
        - `sudo certbot --nginx -d domain.com -d www.domain.com`
    - check nginx configuration:
        - `sudo nginx -t`
    - reload nginx:
        `sudo systemctl reload nginx`






  
   
