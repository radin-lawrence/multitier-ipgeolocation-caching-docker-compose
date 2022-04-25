
# multitier-ipgeolocation-caching-docker-compose

Geolocation refers to the use of location technologies such as GPS or IP addresses to identify and track the whereabouts of connected electronic devices. Because these devices are often carried on an individual's person, geolocation is often used to track the movements and location of people and surveillance.

Geolocation is the ability to track a device’s whereabouts using GPS, cell phone towers, WiFi access points or a combination of these. Since devices are used by individuals, geolocation uses positioning systems to track an individual’s whereabouts down to latitude and longitude coordinates, or more practically, a physical address. Both mobile and desktop devices can use geolocation.

Whether the goal is to create an instant connection with a first-time site visitor; to drive revenue for mobile marketing campaigns; or to ensure content is in the right hands, today’s IP Intelligence and geolocation technology provides the type of all-in-one tool to take online and mobile interactions in the right direction—toward a closer relationship with your audience.

The five most common business applications are:


- Targeted Online Advertising — Enables online advertisers and mobile marketers to “geo-target” to a post-code level, increasing advertising’s reach, relevance and response. Additional IP parameters allow for more pinpointed targeting and flexibility.

- Content Localization — Provides online entities with the tools to move away from “one-size-fits-all” web content and instead deliver relevant content, language, currency, products, and promotions—creating an instant connection with visitors—that helps reduce website and transaction abandonment, resulting in increased sales and revenue.

- Geographic Rights Management — Allows online content distributors to adhere to licensing and copyright agreements surrounding usage of digital audio and video content. Additionally, geotargeting can be used to restrict downloads in certain geographic locations or in other online uses where content needs to be legally restricted.

- Analytics — Offers companies a new way to view, parse and analyze data to improve web performance and site effectiveness. It also provides a real-time mechanism that allows customers to analyze data and to take immediate action.

- Fraud Prevention — Delivers the data necessary to expose the anonymity or lift the cloak of fraudsters. Leveraged within a fraud-prevention framework, companies can marry the virtual world and physical attributes to make real-time decisions on the validity of online customers and transactions.

Here we are using the free plan of ipgeolocation.io and it offers a freemium geolocation API that returns:
country, city, state, province, local currency, latitude and longitude, company detail, ISP lookup, language, zip code, country calling code, time zone, current time.

This API is free to use up to 1,000 requests/day and 30K requests per monthn.
The paid plans start at $15/month.

In order to reduce the request count sent to API, we are making use of caching service with the use of Redis.

![image](https://user-images.githubusercontent.com/100775027/164792312-af18a4bb-262d-4d7f-8788-4458a59e1f0b.png)

When a user makes a request, the API service calls Redis to see if the IP is cached, and if it isn't, the API connects the ipgeolocation database to acquire the IP information. Rather than sending the information to the user, the API service caches it in Redis before giving it to the user. So, the following time a user makes a request for the same IP, there is no need for an API request because the information can be retrieved from Redis cache, reducing API queries.

If there is more than one connection to an API at the same time, it can't handle them all at once, thus we're moving to a multi-layer architecture with three front-end services and three API layer and each handled by a load balancer. See the final architecture of the app.

![image](https://user-images.githubusercontent.com/100775027/164792222-1c144577-6433-4c4c-a993-c94dbce1f6cd.png)

# Creating docker-compose.yml file and Defining Services

Create a docker-compose.yml file and add the below configuration to the file.

**Redis cache**

It's for caching and here I'm usin the redis latest version image.

```bash
version: "3"

services:
  redis:
    image: redis:latest
    container_name: redis-ipgeolocation
    networks:
      - geolocation-nw 
```

**API Layer**

Using flask app I build api layer image(radinlawrence/ipgeolocation-api:v1). For reference: https://github.com/radin-lawrence/multitier-ipgeolocation-caching-docker-compose/tree/main/ipgeolocation-api

Here we are using three API layers, which will help to handle more than one connection to the API at the same time.

```bash
api-1:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-1
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=<api key>
    

  api-2:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-2
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=<api key>
    

  api-3:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-3
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=<api key>
    
  ```
  > Remember to replace "<api key>" with your own key. Here I'm using the free plan of ipgeolocation.io. to generate api_key: https://app.ipgeolocation.io/


**Nginx-Internal load balancer**

 Here we are using the nginx load balancer for the three API layers.
```bash
nginx-internal:
    image: nginx:latest
    container_name: nginx-internal
    networks:
      - geolocation-nw
    volumes:
      - ./internal/:/etc/nginx/conf.d/
    ports:
     - "8080:80"
```
 We have to define the configuration for the Nginx webserver to work as a load balancer.
For that,

```bash
mkdir internal && cd internal
vim nginx.conf
```

Add the below lines to the server section in nginx.conf 

```bash
upstream myapp {
    server geolocation-api-1:8080;
    server geolocation-api-2:8080;
    server geolocation-api-3:8080;
}
server {
    listen 80;
    location / {
        proxy_pass http://myapp;
    }
}
```


**Front-end**

Set up an image for ipgeolocation-frontend using flask app. For referance: https://github.com/radin-lawrence/multitier-ipgeolocation-caching-docker-compose/tree/main/ipgeolocation-frontend. Using three frontend with ports 8081,8021,8083.

```bash
 frontend-1:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-1
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8081:8080"

  frontend-2:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-2
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8082:8080"

  frontend-3:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-3
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8083:8080" 
      
```

**Nginx-load balancer**

Here we are using a load balancer to manage the request between the client and frontend.
```bash
 nginx:
    image: nginx:latest
    container_name: nginx-lb
    networks:
      - geolocation-nw
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
    ports:
      - "80:80"


```
Define the configuration for the Nginx webserver.

```bash
upstream frontend {
    server geolocation-frontend-1:8080;
    server geolocation-frontend-2:8080;
    server geolocation-frontend-3:8080;
    
}
server {
    listen 80;
    server_name example.com www.example.com;
    location / {
        proxy_pass http://frontend;
    }
}

```

The docker-compose.yml file will be like
```bash
version: "3"


services:
  redis:
    image: redis:latest
    container_name: redis-ipgeolocation
    networks:
      - geolocation-nw 
  
  api-1:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-1
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
    

  api-2:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-2
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
   

  api-3:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-3
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
    
  
  nginx-internal:
    image: nginx:latest
    container_name: nginx-internal
    networks:
      - geolocation-nw
    volumes:
      - ./internal/:/etc/nginx/conf.d/
    ports:
     - "8080:80"
      

  frontend-1:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-1
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8081:8080"

  frontend-2:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-2
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8082:8080"


  frontend-3:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-3
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8083:8080" 
      
  nginx:
    image: nginx:latest
    container_name: nginx-lb
    networks:
      - geolocation-nw
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
    ports:
      - "80:80"

networks:
  geolocation-nw:

volumes:
  mycert-auth:
```
  
Create the containers with docker-compose up and the -d flag, the site will load with out ssl.
  
```bash
docker-compose up -d
```

**Create the certificate using Certbot**

Edit the  configuration for the Nginx webserver as:

```bash
upstream frontend {
    server geolocation-frontend-1:8080;
    server geolocation-frontend-2:8080;
    server geolocation-frontend-3:8080;
    
}
server {
    listen 80;
    server_name example.com www.example.com;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://docker.radinlaw.tech$request_uri;
    }
}
```


> location /.well-known/acme-challenge/ { root /var/www/certbot; }

"location /" block serves the files Certbot need to authenticate our server and to create the HTTPS certificate for it.

Basically, we say "always redirect to HTTPS except for the /.well-know/acme-challenge/ route".




```bash
 nginx:
    image: nginx:latest
    container_name: nginx-lb
    networks:
      - geolocation-nw
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - mycert-auth:/var/www/certbot/
    ports:
      - "80:80"
      - "443:443"
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - mycert-auth:/var/www/certbot/

```

The doker-compose.yml file will look like this:

```bash
version: "3"


services:
  redis:
    image: redis:latest
    container_name: redis-ipgeolocation
    networks:
      - geolocation-nw 
  
  api-1:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-1
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
    

  api-2:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-2
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
    

  api-3:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-3
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
   
  
  nginx-internal:
    image: nginx:latest
    container_name: nginx-internal
    networks:
      - geolocation-nw
    volumes:
      - ./internal/:/etc/nginx/conf.d/
    ports:
     - "8080:80"
      

  frontend-1:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-1
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8081:8080"

  frontend-2:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-2
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8082:8080"


  frontend-3:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-3
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8083:8080" 
      
  
  nginx:
    image: nginx:latest
    container_name: nginx-lb
    networks:
      - geolocation-nw
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - mycert-auth:/var/www/certbot/
    ports:
      - "80:80"
      - "443:443"

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - mycert-auth:/var/www/certbot/
      

networks:
  geolocation-nw:

volumes:
  mycert-auth:
  

```

We can now reload nginx,
```bash
docker-compose up -d --force-recreate --no-deps nginx
```

Certbot will write its files into the volume mycert-auth and nginx will serve them on port 80 to every user asking for /.well-know/acme-challenge/. That's how Certbot can authenticate our server.

You can now test that everything is working by running 

```bash 
docker-compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ --dry-run -d example.com -d www.example.com
```

 You should get a success message like "The dry run was successful".
  
  ![image](https://user-images.githubusercontent.com/100775027/164792462-2ce4f76a-98a5-427a-99b6-d6676732e6d6.png)

 Now that we can create certificates for the server, we want to use them in nginx to handle secure connections with end users' browsers.

Certbot create the certificates in the /etc/letsencrypt/ folder. Same principle as for the webroot, we'll use volumes to share the files between containers.

```bash
 nginx:
    image: nginx:latest
    container_name: nginx-lb
    networks:
      - geolocation-nw
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - mycert-auth:/var/www/certbot/
      - certbot-ssl:/etc/letsencrypt/
    ports:
      - "80:80"
      - "443:443"

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - mycert-auth:/var/www/certbot/
      - certbot-ssl:/etc/letsencrypt/

```

We have finished docker-compose.yml file and the final code will look like this:

```bash
version: "3"


services:
  redis:
    image: redis:latest
    container_name: redis-ipgeolocation
    networks:
      - geolocation-nw 
  
  api-1:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-1
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
    ports:
      - "8084:8080"

  api-2:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-2
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
    ports:
      - "8085:8080"

  api-3:
    image: radinlawrence/ipgeolocation-api:v1
    container_name: geolocation-api-3
    networks:
      - geolocation-nw 
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=redis-ipgeolocation
      - APP_PORT=8080
      - API_KEY=a65f448146884af995a8be4d1cfc461e
    ports:
      - "8086:8080"
  
  nginx-internal:
    image: nginx:latest
    container_name: nginx-internal
    networks:
      - geolocation-nw
    volumes:
      - ./internal/:/etc/nginx/conf.d/
    ports:
     - "8080:80"
      

  frontend-1:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-1
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8081:8080"

  frontend-2:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-2
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8082:8080"


  frontend-3:
    image: radinlawrence/ipgeolocation-frontend:v1
    container_name: geolocation-frontend-3
    networks:
      - geolocation-nw
    environment: 
      - API_SERVER=nginx-internal
      - API_SERVER_PORT=80
      - API_PATH=/api/v1/
      - APP_PORT=8080
    ports:
      - "8083:8080" 
      
  
  nginx:
    image: nginx:latest
    container_name: nginx-lb
    networks:
      - geolocation-nw
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - mycert-auth:/var/www/certbot/
      - certbot-ssl:/etc/letsencrypt/
    ports:
      - "80:80"
      - "443:443"

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - mycert-auth:/var/www/certbot/
      - certbot-ssl:/etc/letsencrypt/


networks:
  geolocation-nw:

volumes:
  mycert-auth:
  certbot-ssl:
  ```
  Restart your container using 
  ```bash 
  docker-compose up -d
  ```
   Nginx should now have access to the folder where Certbot creates certificates.

   However, this folder is empty right now. Re-run Certbot without the --dry-run flag to fill the folder with certificates:

```bash
docker-compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d example.com -d www.example.com
```

Edit configuration on nginx as:

```bash
upstream frontend {
    server geolocation-frontend-1:8080;
    server geolocation-frontend-2:8080;
    server geolocation-frontend-3:8080;
    
}
server {
    listen 80;
    server_name docker.radinlaw.tech www.docker.radinlaw.tech;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://docker.radinlaw.tech$request_uri;
    }
}

server {
    listen 443 default_server ssl http2;
    listen [::]:443 ssl http2;

    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://frontend;
    }
}

```
Reloading the nginx server now will make it able to handle secure connection using HTTPS

```bash
docker-compose up -d --force-recreate --no-deps nginx
```

We can check the IP info by using the URL http://example.com/ip/ followed by the required IP like http://example.com/ip/8.8.8.8
  
  ![image](https://user-images.githubusercontent.com/100775027/164792541-34948962-bef1-490f-be25-6087acc5dda5.png)
  
  ## Conclusion
Here is a simple document on how to use ipgeolocation with caching. Please contact me if you encounter any difficulty with this.

  
 ### ⚙️ Connect with Me
<p align="center">
<a href="https://www.linkedin.com/in/radin-lawrence-8b3270102/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a>
<a href="mailto:radin.lawrence@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
