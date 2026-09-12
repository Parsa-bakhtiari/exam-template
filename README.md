1. DNS :
-----------
First Problem is that Docker Compose is not exist in VM . 
so I have to install docker-compose or docker-compose-v2 . but because of DNS problem I can't even "apt update" . 
so I disable systemd-resolvd service and remove default fire resolv.conf and create new file with below Item : 
nameserver 8.8.8.8
nameserver 1.1.1.1


after installing docker-compose , the problem is : 
root@parsa-bakhtiari-scenario1:/opt/service-catalog# curl localhost/graph
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.27.5</center>
</body>
</html>


2.Backend returned HTTP 500
--------------------------------
Symptom. Every endpoint returned 500, even called from inside the backend container. Gunicorn itself was running fine.

Cause. backend was attached only to nginx-backend-net, and db only to backend-db-net. Docker's DNS resolves service names only within shared networks, so db was unresolvable and the database connection always failed.

bash
getent hosts db    # no output

Fix. Attach backend to both networks:

yaml
  backend:
    networks:
      - nginx-backend-net
      - backend-db-net

Verified. curl http://127.0.0.1:5000/graph inside the container returned the seeded graph.



3. HTTP 502 from the host
-------------------------------
Symptom. curl http://backend:5000/graph worked inside the nginx container, but curl http://localhost/graph from the host returned 502.

Cause. The upstream in nginx.conf was set to backend-api, which does not exist. The Compose service is named backend, and Docker's DNS only registers the real service name. The error log confirmed it:

backend-api could not be resolved (3: Host not found)

The curl inside the container worked because it was pointed at the correct name by hand, not at what nginx was actually configured to use.

Fix. Point the upstream at the real service name:

nginx
        location / {
            resolver 127.0.0.11 valid=10s ipv6=off;
            set $backend_upstream http://backend:5000;
            proxy_pass $backend_upstream$request_uri;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

Applied without downtime. nginx.conf is bind-mounted, so it was edited on the host and reloaded in place:

bash
docker exec service-catalog_nginx_1 nginx -T | grep -e resolver -e proxy_pass
docker exec service-catalog_nginx_1 nginx -t
docker exec service-catalog_nginx_1 nginx -s reload

Verified. curl http://localhost/graph and /nodes both returned 200 with the expected JSON.



4. Summary of changes
-------------------------------
docker-compose.yml	---> Added backend-db-net to the backend service
nginx/nginx.conf	--> Upstream changed from backend-api to backend
