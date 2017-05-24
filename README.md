# Deploying FGH Confluence over HTTPS with letsencrypt.

## How it works

When you bring up the service with ```docker-compose up```, docker compose starts an nginx reverse proxy, your app container, and the official letsencrypt container.

The proxy image's init script starts nginx in the initial config:


```nginx
events { worker_connections 1024; }
http {
	server {
		listen 80;
		server_name ___my.example.com___;

		location /.well-known/acme-challenge {
			proxy_pass http://letsencrypt:80;
			proxy_set_header Host            $host;
			proxy_set_header X-Forwarded-For $remote_addr;
			proxy_set_header X-Forwarded-Proto https;
		}

		location / {
			proxy_pass http://confluence-fgh:8090;
			proxy_set_header Host            $host;
			proxy_set_header X-Forwarded-For $remote_addr;
		}

	}
}
```

The initial config allows letsencrypt's acme challenge to get to the letsencrypt container. The letsencrypt container runs in _standalone_ mode, connecting to letsencrypt.org to make the cert request and then waiting on port 80 for the acme-challenge.

When letsencrypt issues the challenge request, the le client writes the certs to /etc/letsencrypt, which is a volume mounted to the nginx container. The nginx container's init script notices the certs appear, and loads a new config, setting up the https port forward.

```nginx
events { worker_connections 1024; }
http {
	server {
		listen 80;
		server_name ___my.example.com___;

		location /.well-known/acme-challenge {
			proxy_pass http://letsencrypt:80;
			proxy_set_header Host            $host;
			proxy_set_header X-Forwarded-For $remote_addr;
			proxy_set_header X-Forwarded-Proto https;
		}

		location / {
			return         301 https://$server_name$request_uri;
		}

	}

	server {
		listen 443;
		server_name ___my.example.com___;

		ssl on;
		ssl_certificate /etc/letsencrypt/live/___my.example.com___/fullchain.pem;
		ssl_certificate_key /etc/letsencrypt/live/___my.example.com___/privkey.pem;
		ssl_session_timeout 5m;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
		ssl_prefer_server_ciphers on;

		ssl_session_cache shared:SSL:10m;
		ssl_dhparam /etc/ssl/private/dhparams.pem;

		location /.well-known/acme-challenge {
			proxy_pass http://letsencrypt:443;
			proxy_set_header Host            $host;
			proxy_set_header X-Forwarded-For $remote_addr;
			proxy_set_header X-Forwarded-Proto https;
		}

		location / {
			proxy_pass http://confluence-fgh:8090;
			proxy_set_header Host            $host;
			proxy_set_header X-Forwarded-For $remote_addr;
		}
	}
}
```

The service is now running over https.

## How to run it
Run the confluence/wiki with nginx reverse proxy configured with TLS. Go to project folder
(i.e. confluence-ssl-docker)

Edit `confluence-ssl-docker/confluence/mysql/Dockerfile` providing appropriate values for
mysql root user, confluence database name, confluence user and confluence user password.
After that run the following.

```
cd confluence
docker-compose  up -d
cd ../le-docker-compose
docker-compose  up -d
```

Confirm the services are running by running `docker-compose ps`

The letsencrypt container exited - this is what we want.

## Renew your certificate

Start the letsencrypt container with docker compose. The container starts, runs the acme process, and exits.

```
cd le-docker-compose
docker-compose run letsencrypt
```

Then, reload the nginx config

```
docker exec ledockercompose_nginx_1 nginx -s reload
```

Done.
