<br>
<p align="center">
  <img alt="HackUPC Fall 2016" src="https://github.com/hackupc/frontend/raw/master/src/images/hackupc-header-blue.png" width="620"/>
</p>
<br>

Backend for hackathon application management.

## Features

- Sign up and basic data management interface for hackers
- Review application interface
- Sends invites and controls confirmation and cancellation application flow
- Check-in interface with QR scanner
- Admin dashboard with stats
- Flexible email backend (SendGrid is the default supported backend)
- Reimbursement money admin interface
- (Optional) SendGrid contact list synchronization with confirmed users
- (Optional) Slack invites on confirm and on demand in admin interface

## Configuration (enviroment variables)

- **SG_KEY**: SendGrid API Key. Mandatory if you want to use SendGrid as your email backend. You can manage them [here](https://app.sendgrid.com/settings/api_keys).  Note that if you don't add it the system will write all emails in the filesystem for preview.
You can replace the email backend easily. See more [here](https://djangopackages.org/grids/g/email/)
- **TP_KEY**: Typeform API key. Mandatory for retrieving the information from applications in Typeform. See how to obtain it [here](https://www.typeform.com/help/data-api/)

## Setup

Needs: Python 3.X, virtualenv

- `git clone https://github.com/hackupc/backend && cd backend`
- `virtualenv env --python=python3`
- `source ./env/bin/activate`
- (Optional) If using Postgres, set up the necessary environment variables for its usage before this step
- `pip install -r requirements.txt`
- `python manage.py migrate`
- `python manage.py createsuperuser`

### Typeform setup

There's no way to create Typeform forms automatically (yet), so you will need to create a Typeform for the application part.
TODO: Include guide to create and prepare your Typeform

## Run server

### Local environment

Run server to 0.0.0.0
`python manage.py runserver 0.0.0.0:8000`

### Production environment

See this [tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04) to understand and set it up as in our server.

#### Set up gunicorn service in Systemd
Needs: Systemd.
- Create server.sh from template: `cp server.sh.template server.sh`
- `chmod +x server.sh`
- Run `restart.sh`. This will update the database, dependecies and static files.
- Edit this file `/etc/systemd/system/gunicorn.service`
- Add this content
```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=user
Group=www-data
WorkingDirectory=/home/user/project_folder
ExecStart=/home/user/project_folder/server.sh >>/home/user/project_folder/out.log 2>>/home/user/project_folder/error.log

[Install]
WantedBy=multi-user.target

```

- Replace `user` for your user (deploy in our server).
- Replace `project_folder` by the name of the folder where the project is located
- Create and enable service: `sudo systemctl start gunicorn && sudo systemctl enable gunicorn`

#### Deploy new version

Needs: Postgres environment variables set

- `git pull`
- `./restart.sh`
- `sudo service gunicorn restart`


#### Set up Postgres

Needs: PostgreSQL installed

- Enter PSQL console: `sudo -u postgres psql`
- Create database: `CREATE DATABASE backend;`
- Create user for database: `CREATE USER backenduser WITH PASSWORD 'password';` (make sure to include a strong password)
- Prepare user for Django

```sql
ALTER ROLE backenduser SET client_encoding TO 'utf8';
ALTER ROLE backenduser SET default_transaction_isolation TO 'read committed';
ALTER ROLE backenduser SET timezone TO 'UTC';
```

- Grant all priviledges to your user for the created database: `GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;`
- Exit PSQL console: `\q`

#### Set up nginx

Needs: Nginx

- `sudo vim /etc/nginx/sites-available/default`
- Add site:

```
server {
    listen 80;
    listen [::]:80;

    server_name my.hackupc.com;


    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        alias /home/user/project_folder/staticfiles/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/user/project_folder/backend.sock;
    }


}
```


## Management Commands

This can be added to crontab for periodic automatic execution.

### Fetch last forms

Fetches all aplications and inserts those who don't exist in the DB

- Activate virtualenv (optional)
- Run: `TP_KEY=REPLACE_WITH_TYPEFORM_API_KEY python manange.py insert_applications`

### Check invited applications for expirations

Sends last reminder email to applications invited (not confirmed or cancelled) that are 4 days old. Sets application as expired after 24 hours of sending last reminder email.

- Activate virtualenv (optional)
- Run: `SG_KEY=REPLACE_WITH_SENDGRID_KEY python manange.py expire_applications`

