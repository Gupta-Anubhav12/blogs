![django](https://i.postimg.cc/Pf0LGHvw/develope-python-django-website.jpg)

# Hosting Django Using Nginx and Gunicorn on Linux 

There are multiple blog posts and tutorials already available for Django Hosting, but I was convinced enough to write one for personal as well as community use, For this we are using A Debian Linux, say Ubuntu server or something similar. For sake of simplicity I am dividing it into 3 parts: namely configuration ,Configuring Django, Writing Servers

##  Part 1

- ssh into server
  - ```ssh <username>@<ip>```
- Set up dependencies 
  - ```sudo apt-get update```
  - ```sudo apt-get install python3-pip python3-dev nginx```
  - ```sudo apt install git```
- Set Up virtual Environment 
  - ``` sudo -H pip3 install --upgrade pip```
  - ```sudo -H pip3 install virtualenv```
- we shall now get the code from git repo using command this makes a dir with name as of our repo say **webserver**
  - ``` git clone <url to git repo here >```
  - ```cd webserver```
- we make a virtualenv for holding python packages 
  - ```virtualenv env```
  - ```source env/bin/activate```
- install python packages 
  - ```pip install django gunicorn```
  - ```pip install -r requirements.txt```

## Part 2
- we now configure settings.py file for django in my case it is in a dir called **website** this folder the name of folder should come handy afterwards and should be remembered 
  - ```cd website```
  - we will change allowed host and static root now
    - ```ALLOWED_HOSTS = ['your Public IP here, and domain names , keep it * to allow all hosts']``` 
  - then run the following commnad for static files 
    - ``` python manage.py collectstatic```
  - static and media roots shall look like this 
    - ```
        STATIC_URL = '/static/'
         MEDIA_URL = '/media/'
         MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
         STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
         ```

  - we are ready to fire up test server but for this we should add inbound firewall rule at port 8000
    - ```sudo ufw allow 8000```
  - also do same in router management 
  - now run these commands to try working of code and test if requirements are fullfilled
    - ``` python manage.py runserver 0.0.0.0:8000```
  - go to your public IP to see website running on development server
  - now we gotta patch it up in nginx and gunicorn 


## Part 3
- Try the following command to test if gunicorn is good to be written as a service
  - ```gunicorn --bind 0.0.0.0:8000 ```website```.wsgi```
- remember to change ```website`` from above to name of your directory having wsgi.py
- this fires up gunicorn and you can visit the ip address to check if it works 
- Now we shall write gunicorn socket and service and make it a daemon process 
  - ``` sudo nano /etc/systemd/system/gunicorn.socket```
  - now here we write the following in file 
  ``` 
        [Unit]
        Description=gunicorn socket

        [Socket]
        ListenStream=/run/gunicorn.sock

        [Install]
        WantedBy=sockets.target
    ```
    ---
    ```sudo nano /etc/systemd/system/gunicorn.service```
    - add following in the file
     ```

        [Unit]
        Description=gunicorn daemon
        Requires=gunicorn.socket
        After=network.target

        [Service]
        User=judge
        Group=www-data
        WorkingDirectory=/home/<username>/<name of repo>
        ExecStart=/home/<username>/<name of repo>/env/bin/gunicorn \
            --access-logfile - \
            --workers 3 \
            --bind unix:/run/gunicorn.sock \
            <name of django application>.wsgi:application
        [Install]
        WantedBy=multi-user.target
    ```

    - run the following commands to fire it up and create symlink
     - ``` sudo systemctl start gunicorn.socket```
     - ``` sudo systemctl enable gunicorn.socket ```
     - ``` sudo systemctl daemon-reload```
     - ``` sudo systemctl restart gunicorn``` 
---
Now we will configure the final file for nginx web server 
 - we shall make a file with following command
 - ``` sudo nano /etc/nginx/sites-available/<myproject>```
 - copy following configurations in the file 
    ``` 
        server{
        listen 80;
        server_name  <domain name> <ip address>;
        client_max_body_size 2M;
        location = /favicon.ico { access_log off; log_not_found off; }
        location /static/ {
            gzip_static on;
            autoindex on;
            root /home/<username>/<path to static folder>/;
        }
        
        location /media/ {
            root /home/<username>/<path to media folder>;
        }
        location /{
            include proxy_params;
            proxy_pass http://unix:/run/gunicorn.sock;
        }
    }
    ```
    - on saving the file we need to make a symlink with this command 
        - ``` sudo ln -s /etc/nginx/sites-available/<name of my project> /etc/nginx/sites-enabled```

        - we now check configuration for errors
         -   ``` nginx -t ```
         - we should give firewall access for nginx
         - ```sudo ufw allow 'nginx full' ```
- if it shows success we are good to go, just restart the servers and we are live for love of Django 
-   ``` sudo systemctl restart nginx```
- ``` sudo systemctl restart gunicorn```

Now if it dosen't show up or shows only welcome to Nginx , there might be errors in gunicorn configurations that can be checked using
-   ``` sudo systemctl status gunicorn``` 



Lets understand what is writtern there and how it shall work :
1. The server is listening at port 80 (default for web servers)
2. The server name shows allowed ip and domains 
3. client_max_body_size shows maximum limit to media/files that can be uploaded from client side (Admin panel and froms)
4. gzip_static allows gzip compression for static files and helps them optimise
5. location static and media points to static and media folders of project
6. autoindex on signifies the indexing of media files on google automatically

The work is tried and tested on multiple websites as on 29/03/2021 , any errors/suggestions can be sent at [webmaster@realdevils.com](mailto:webmaster@realdevils.com)






    
