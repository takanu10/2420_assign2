# 2420_assign2

This readme will cover:
- Digital Ocean configuration for the Virtual Private Cloud (VPC), load balancer, and firewall for 2 droplet servers
- Caddy web server installation on droplets 
- Utilizing Fastify framework to display web apps 

### 1. Create Digital Ocean Infrastructure

Create a custom VPC and this will ensure optimal security/safety for your 2 droplets.

Create the 2 droplets and connect them to the VPC by using the drop-down bar in the droplet setup menu.

Create the load balancer and this will ensure optimal web traffic management when your servers go live.

Create the firewall.

### 2. Create Regular User on Both Droplets

SSH into a droplet by specifying its allocated IP address.
*Note: be mindful of adding your server's IP addresses and usernames

![image](https://user-images.githubusercontent.com/98194516/204706520-836f96ac-5d1a-4596-a225-7ec6ed9ee49d.png)

Once in the droplet, add the new user.

![image](https://user-images.githubusercontent.com/98194516/204707062-8d798352-fb99-4cd8-98b7-5ae6f5d7d0be.png)

Update group privileges for the new user to sudo.

![image](https://user-images.githubusercontent.com/98194516/204707114-fc305486-c8a8-4518-bba0-670dcc3ae965.png)

Move SSH key pair from root to the new user's home directory.

![image](https://user-images.githubusercontent.com/98194516/204707176-ec38eed7-3332-44f2-af6a-b82fdf29c424.png)

Change password of new user.

![image](https://user-images.githubusercontent.com/98194516/204707206-b3667bd5-4013-4480-aac0-78c0ae8ba03b.png)

Go into ssh configuration file and set rootPrivileges to "no" by using insert mode via `i` -> `:wq` to save + exit 
![image](https://user-images.githubusercontent.com/98194516/204707324-70517680-849c-444e-a5cf-146b36c1b73a.png)

When changes are done, restart service

![image](https://user-images.githubusercontent.com/98194516/204707416-61d7ef3c-a44a-4672-b5ce-0bb200f1cc92.png)

### 3. Install Web Server on Droplets

Install caddy by using the command below.

![image](https://user-images.githubusercontent.com/98194516/204707801-59b5494b-446f-49be-b955-5f640aec7463.png)

This will download the binary in an archived .tar.gz format. Unarchive the .tar.gz file.

![image](https://user-images.githubusercontent.com/98194516/204707969-40348d5f-c785-44d3-a19c-0b67b898de6e.png)

Once unarchived, change file ownership and group to root

![image](https://user-images.githubusercontent.com/98194516/204708036-a35dcc60-f205-4a75-81f3-f43deea4dad6.png)

Then copy the caddy file to the bin directory

![image](https://user-images.githubusercontent.com/98194516/204708078-4fb857ab-6d04-443c-9737-fc8bf9616d49.png)

### 4. Write Web App

This tutorial will prioritize a basic configuration level for the web application, so feel free to add anything you'd want to the html files. 

For our purposes, a simple `hello world` will suffice.

Sample Code Below:
```
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Greetings</title>
</head>

<body>
    <div>
        <h1>Hello world!</h1>
    </div>
</body>

</html>
```

On your local machine in WSL, create a new directory called `2420-assign-two`.

Within this directory, create 2 new directories `html` and `src`.

Inside the `html` directory, create an `index.html` file. The contents of this html file can be anything you want.

Now enter `src` directory.

Create a new node project via `npm init` and use `npm i fastify` to install fastify.

Create an `index.js` file and copy the code below into there.

```
// Require the framework and instantiate it
const fastify = require('fastify')({ logger: true })

// Declare a route
fastify.get('/api', async (request, reply) => {
  return { hello: 'Server 1/2' }
})

// Run the server!
const start = async () => {
  try {
    await fastify.listen({ port: 5050 })
  } catch (err) {
    fastify.log.error(err)
    process.exit(1)
  }
}
start()
```
Once complete, use sftp to move the `html` and `src` directories to both server droplets.

```
sftp -i ~/.ssh/{key_file_name} {user}@{droplet_ip}
put -r src
put -r html
```
The directories are now on the other server. 

### 5. Create Caddyfile on WSL

Create a caddyfile on WSL and paste this into it.

```
http://137.184.247.221 {
	root * /var/www
	file_server
	reverse_proxy /api localhost:3030
	file_server
}
```

Now use sftp to transfer it over to your 2 droplets.

### 6. Install node and npm with Volta

`source ~/.bashrc will allow you to continue without restarting your VM environment.

```
curl https://get.volta.sh | bash
source ~/.bashrc
volta install node
```

### 7. Create service file 

On WSL, create service file named `hello_web.service` that will start your node application.

```
[Unit]
Description=hello world web app that restarts service on failure + requires configured network 
After=network.target
Wants=network-online.target

[Service]
Restart=on-failure
ExecStart=/home/darren/.volta/bin/node /var/www/src/index.js
User=darren
Group=darren 
SyslogIdentifier=hello_app

[Install]
WantedBy=multi-user.target
```

### 8. Upload and run web app

Using sftp, upload `caddyfile` and `hello_web.service` to servers.

```
sftp -i ~/.ssh/{key_file_name} {user}@{droplet_ip}
put -r caddyfile
put -r hello_web.service
```

Now ssh into each server and ensure files are in the proper locations.
- service file goes in `/etc/systemd/system`; `sudo mv hello_web.service /etc/systemd/system`
- caddyfile server block needs a couple more steps:

```
sudo mkdir /etc/caddy
sudo mv caddyfile /etc/caddy
```
Now that everything's in the right place, start and enable the `hello_web.service`.

```
sudo systemctl start hello_web.service
sudo systemctl enable hello_web.service
```


