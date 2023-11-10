This is the documentation about the Nginx configurations to do static resources separation and reverse proxy from one instance to another (based on Vue frontend project).
## User issue
### Goal
Set up nginx reverse proxy for the deployed graph and 10k pages, to make them subpages of the overall landing page.

### User Story
As a client, in the Landing Page (deployed on proxy machine (IP: landingpage), with its 80 port exposed to the public), I want to forward to different project pages that were deployed on different instances (deployed on web machine (IP: webproject), with all ports exposed to the proxy machine through private IP, but no ports to the public), with the same URL prefix as the Landing Page. 

### Definition of Done
For example, if the landing page is on http://landingpage:80, project1 and project2 should be proxied from another instance, and act as http://landingpage/project1 and http://landingpage/project2.

### Problem Statement
1. The common static resources could be deployed separately on the proxy server for efficiency, and then be accessed by the public with a URL so that we can fetch them in any further projects.
2. For the independent resources, we want each project to fetch its own resources from a certain path, it's important to configure the path before doing Nginx Proxy.

## Solution
### Separate common Static Resources
Some static resources in a series of projects are the same, such as the background picture and icons. We don't want them to be deployed along with other resources of these projects every time. So we choose to separate these resources in a directory named /common, and upload it to a certain path of our proxy machine (here is ```/var/www/common```).

In the ```nginx.conf``` file of the proxy server, open a new port server of 9999, and make these common static resources could be visited by http://landingpage:9999
The configuration is: 
```
server {
    listen 9999 default_server;
    listen [::]:9999 default_server;

    location /common {
        autoindex on;
        alias /var/www/common/;
    }
}
```

After you can ```wget localhost:9999``` on the proxy machine successfully, We need to do our first proxy here.

Within the 80 port server of proxy machine (the only one exposed to public), we add this configuration:
```
location /common {
    proxy_pass http://localhost:9999/common;
    autoindex on;
}
```
By doing this, we can access the common static resources from http://landingpage/common, and could directly call this URL inside our projects.

But to avoid the IP address of the proxy server being changed, we choose to use relative path and do another proxy on the web machine. Here is the configuration file within web machine's ```nginx.conf``` at port 80:
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    location /common {
        proxy_pass http://landingpage/common;
        autoindex on;
    }
}
```
After that, the web machine and proxy machine could both access these resources from relative path ```/common/xxx.jpg``` or ```/common/xxx.ico```, and we can just implement this relative path to any parts (like index.html) of the projects deployed on either machine.

### Independent Resources Proxy
We'll have the docker containers of project1 and project2 running on port 8000 and port 8888 on the web machine respectively. And through a reverse proxy similar to the last topic, we'd like the public to access these two websites from http://landingpage/project1 and http://landingpage/project2.

Pay attention to the secondary directory /project1 and /project2, it should be noticed that after reverse proxy, the proxy server machine wants to find all the independent resources files under the default nginx root/project1 or default nginx root/project2. So, in order to guide the proxy server to this path, we should first configure the ```vue.config.js``` file within the project.
Around line 13, change the public path to:
```
publicPath: process.env.NODE_ENV === 'production'
    ? '/project1'
    : '/project1',
```
or
```
publicPath: process.env.NODE_ENV === 'production'
    ? '/project2'
    : '/project2',
```

After web pack and docker deployment, make sure that we could ```wget webproject:8080``` and ```wget webproject:8888``` from the landingpage proxy server, along with the resources files under the projects.

Here is the configuration part for the proxy machine's ```nginx.conf``` to do the reverse proxy of the two projects from web machine's port 80 server:
```
location /project1 {
    proxy_pass http://webproject:8080/;
    autoindex on;
}

location /project2 {
    proxy_pass http://webproject:8888/;
    autoindex on;
}
```

For the landing page deployment parts, it's simpler. Just run the app docker to a random port (like 8000) on the proxy machine, and do a reverse proxy again. (Remember the landing page doesn't need public path change within ```vue.config.js``` because we want to visit this website without any secondary directory)
```
location / {
    proxy_pass http://localhost:8000/;
    autoindex on;
}
```
By doing this, we could access the landing page app from http://landingpage, and the two projects from http://landingpage/project1 and http://landingpage/project2.
