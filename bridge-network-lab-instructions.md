#Demo: Default bridge network with cli

Let’s start by running two Nginx containers (in the background with -d) named app1 and app2:

`docker run -d --name app1 nginx:alpine`
`docker run -d --name app2 nginx:alpine`

We can check that Nginx is in fact running on both containers by using docker exec to run curl localhost on both containers:

`docker exec app1 curl localhost`
`docker exec app2 curl localhost`

[^1]:docker exec is used to execute a command inside a running container.
[^1]:nginx runs on port 80 inside its container.
[^1]:curl localhost is the same as curl http://localhost:80

Now let’s try to communicate between the containers.

Trying to reach app2 from app1 using the hostname app2 will not work since DNS doesn’t work out of the box with the default bridge.

`docker exec app1 curl app2`

Let’s get the IP address of app2 with:
`docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' app2`

If we now try to reach app2 from app1 again, but with the IP address:
`docker exec app1 curl <ip>`

We should see that it works!

Using IP addresses is neither reliable nor easy to maintain since more containers could be added to the default network.

If we used the legacy --link option, we could have used the name of the container instead of the IP. But there are major drawbacks to using the default bridge (as discussed in the previous section), and using the --link is discouraged.

#Demo: User-defined bridge network with cli

A user-defined bridge network has to be created before it can be used. So let’s create one named my-bridge:
`docker network create --driver bridge my-bridge`

NOTE: since bridge is the default network driver, specifying --driver bridge in the command is optional.

We will run two containers again and name them app3 and app4:
`docker run -d --name app3 --network my-bridge nginx:alpine`
`docker run -d --name app4 --network my-bridge nginx:alpine`

Again, let’s test if Nginx is running properly on both containers:

`docker exec app3 curl localhost`
`docker exec app4 curl localhost`

On successfully printing the index pages, let’s move on to communication between the containers.

If we try to reach app4 from app3 using the hostname app4:
`docker exec app3 curl app4`

The same thing will also work when reaching app3 from app4:
`docker exec app4 curl app3`

Will we be able to reach app2 from app3 using DNS?
`docker exec app3 curl app2`

Nope.

What about using app2’s IP address? (connection timeout added to not keep curl waiting indefinitely)

`docker exec app3 curl 172.17.0.3 --connect-timeout 5`

No again.

We cannot reach app2 from app3 (or app4) and vice versa because they are on two separate networks, the default bridge and our user-defined my-bridge.

#Demo: publishing container ports

Containers connected to the same bridge network effectively expose all ports to each other. For a container port to be accessible from the host machine or hosts of external networks, that container port must be published using the -p or --publish flag.

Let’s see this in action.

All the existing containers have Nginx running inside them separately on port 80 of each container, but none of them can be reached from the host’s port 80 yet. We can confirm by running curl on the host:

`curl localhost`

Assuming you didn’t have any applications running on port 80, curl should return an error like “Connection refused”.

If we want the container’s application (in this case Nginx) to be reachable from the host, we can publish its port (i.e. map the container’s port to a host port).

`docker run -d --name app5 -p 3000:80 nginx:alpine`

This will map port 3000 on the host to port 80 inside the container.

If we now try to curl on port 3000 of the host:

`curl localhost:3000`

We should see the Nginx index page.

#Clean Up

We can remove all containers on the system using the following command:
`docker rm -f $(docker ps -aq)`

Let’s also remove the my-bridge as well.
`docker network rm my-bridge`