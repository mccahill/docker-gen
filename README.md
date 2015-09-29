## Managing an nginx/docker-gen/rstudio cluster

Use the nginx.tmpl template from the Duke-templates folder to create config files for nginx
dynamically for RStudio container instances.

This assumes you would start nginx, then docker-gen, and the optionally the RStudio 
containers (see https://github.com/mccahill/docker-rstudio) something like this:
```
	sudo docker run -d -p 80:80 -p 30000-31000:30000-31000 --name nginx -v /srv/docker-stuff/nginx/:/etc/nginx/conf.d -t nginx

	sudo docker run -d --volumes-from nginx -v /var/run/docker.sock:/tmp/docker.sock -v /srv//docker-stuff/nginx-templates:/etc/docker-gen/templates -t docker-gen -notify-sighup nginx -watch -only-exposed /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```

# Starting many containers at once based on a mapping file

A better example is to use a shell script to read a mapping file and
run containers based on the user-numbers and passwords in that file. 
We mount a user home directory volume for each user based on their user-number (PORT),
and set the port which they are proxied to (MAP_VIRTUAL_PORT) based on that number as well. 

The docker-gen template keys off the VIRTUAL_HOST environmental variable to know which docker containers
it should update the nginx config file for. 

```
#!/usr/bin/csh
#
# walk the mapping file to get ports and passwords to start containers for each
#
$PWD=pwd
$HOSTNAME=hostname
foreach line (`cat $PWD/mapping-example`)
  set split = ($line:as/,/ /)
  set PORT = $split[1] 
  set PW = $split[2]
  echo $HOSTNAME:30$PORT
  sudo docker run -d -p 127.0.0.1::8787 -e USERPASS=$PW \
    --name 30$PORT \
    -e VIRTUAL_HOST=$HOSTNAME:30$PORT \
    -e MAP_VIRTUAL_PORT=30$PORT \
    -v $PWD/homedirs/user$PORT\:/home/guest \
    -i -t r-studio
  sleep 3
end
```

Be careful when you run this because if old, terminated containers are still present, the name for the 
container (in the example above -name 30$PORT) will still exist, and docker will refuse to run a new instance. 

To get around this, do a 'docker rm' on terminated containers.

Examples of how to start and stop everything for an nginx/docker-gen/rstudio cluster can be found in the files:
```
	start-rstudios
	stop-rstudios
```

