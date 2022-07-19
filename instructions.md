<h4>how to set up the web service using containers</h4>

<h4 color='red'>create a network on the machine</h4>

```markdown
docker network create --subnet=160.100.0.1/16 app-net

    ~check networks ~
    docker network ls
    docker inspect app-net
```
#### create the database container ###

```markdown
docker run -d --name app-db --cpus 0.5 --memory 512m -e MYSQL_USER=michael -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=web_app_db --net app-net mysql:5.7
```

create our application container to run in the background -d map the current directory to the /code directory
the container uses port 8000, so direct the port to 8080. specify the memory and cpu allocation 
```markdown
 docker run -itd --name web-app-1 -v $PWD:/code --cpus 0.5 --memory 512m -p 8080:8000 --workdir /code --net app-net -e DOCKER_CONTAINER_ID=1 python:3.8 

```
the application should be working if you look at docker ps -a
see the python web-app-1 running
enter the web-app-1 container by docker exec -it web-app-1 bash
now once wwe are in the python container, we need to install all the requirements to run.

```markdown
pip install -r requirments.txt
```

now you need to set up the migrations configuration for your application. 
It sets up the database from the settings information in the models.py and the settings file in the cool_web_app
once the file is setup you can do the migrations

```markdown
python manage.py makemigrations app_events
python manage.py migrate
```

enter the mysql database container

```markdown
docker exec -it app-db -bash
```
access the server
```markdown
mysql -u root -p
 root when asked for a password
 you can then:
        show databases
        and then all SQL commands should work.
```

populate the database with some information from the python web-app-1 run
```markdown
python manage.py populate_events
```

run a load tester, use a nodejs tester like this:
create a container with nodejs

```markdown
docker run -itd --name load_tester --cpus 2 --memory 2g --net net-app node:latest
```
remember that net-app is the network we created.

access the load_tester container
```markdown
docker exec -it load_tester bash
```

load the load test library on npm
```markdown
npm install -g loadtest
```
add some software to the container
```markdown
apt update 
apt install - y
apt install vim or nano curl // you can use vim or nano 
```
you don't have to enter the python application container each time you want to send a command, you can 
```markdown
docker exec web-app-1 -sc -c "python manage.py runserver 0.0.0.0:8000"
```
open a new terminal and split the terminal in half.
In the top half run
```markdown
docker stats web-app-1 app-db load_tester
```
in the bottom half 
access the load_tester container with 
```markdown
docker exec -it load_tester bash
```
see if you can access the web page with a curl command
```markdown
curl http://web-app-1:8000
```
to do a load test run 
```markdown
loadtest -c 1 --rps 1 http://web-app-1:8000
```
 -c 1 is the amount of concurrent requests
 --rps is the amount of requests per seconds
 the more request per requests per seconds you send the more load the server has to deal with until it fails.
 
sometimes the application will fail outright and you will have to stop the application
with docker stop web-app-1
the start the application with docker start web-app-1
restart the server with docker exec web-app-1 sh -c "python manage.py runserver 0.0.0.0:8000"
  or 
gunicorn if you are using gunicorn

Start the python application container
```markdown
docker exec -it web-app-2 bash 
```
then start a gunicorn server
```markdown
gunicorn -b 0.0.0.0:8000 cool_web_app.wsgi:application
```
If it is in use that means that you django server is still running?

create a nginx load balancer
```markdown


```

copy the configured configuration file for the nginx load balancer
```markdown
docker cp deployment/nginx/default.conf load_balancer:/etc/ngnix/conf.d
```

create the load balancer, a nginx server
```markdown
docker run -itd --name load_balancer --cpus 2 --memory 2g --net app-net -p 80:80 --wordir /etc/nginx/conf.d -v $PWD/static:/static nginx:1.15
```

If you want to use the cool-web-app.com url in the load tester you need to nano /etc/hosts and add the ip address oc the load
balancer with a specified host name of cool-web-app.com

run the load test

Redis is a caching server that caches requests for a short time.
If the application can find it in the Redis server it will use that else it will get the result from Mysql and store the
request results in the redis cache.

```markdown
docker run -itd --name app-redis --cpus 0.5 --memory 512m --net app-net redis

```

finally, this example used 2 application to expand the capabilities horizontally.
create another application 
```markdown
 docker run -itd --name web-app-2 -v $PWD:/code --cpus 0.5 --memory 512m --workdir /code --net app-net -e DOCKER_CONTAINER_ID=2 python:3.8
you will notice that the port information has been removed, django will look after the port allocation -
```

see changes made nginx default configuration will select between the two applications as required.