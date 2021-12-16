## Setup NGINX, Authentication and Discord Alert for Full Nodes

Full nodes are always essential parts for building dApps. We can choose paid options or even setup a node on AWS or GCP for unlimited rate limit of access. For self-deployed full nodes, we expect that they have fixed addresses. However, static IPs are very limited, for example AWS only allows 6 Elastic IPs per region. We need a gateway and alert system for all our nodes!

## Install Nginx
I suggest to use a separate server to run Nginx, for example `t2.medium` on AWS with Ubuntu 20.04 LTS (remember the instance's IP address). Login server console using SSH and then update software packages:
```console
sudo apt update
sudo apt-get upgrade -y
sudo reboot
```
Wait some minutes until the instance finishes to reboot. Then install Nginx:
```console
sudo apt install nginx
systemctl status nginx
```

## Point domain and generate SSL certificate
Using the instance's IP address from previous section and point your subdomain, e.g. `fullnode`, to this address:
![Screen Shot 2021-09-21 at 12.47.02 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632203312020/IXq1IcfyS.png)

If you have not SSL certificate yet, you can generate a new one with Cloudflare (remember to save public and private key):

![Screen Shot 2021-09-21 at 1.30.57 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632205895238/F9HZpQnXc.png)

## Setup Nginx configuration
Add a config file:
```console
sudo touch /etc/nginx/conf.d/full-node.conf
sudo vi /etc/nginx/conf.d/full-node.conf
```

Update config as below
```nginx
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    server_name fullnode.hiliamtech.com;

    ssl_certificate /etc/nginx/conf.d/cert.pem;
    ssl_certificate_key /etc/nginx/conf.d/cert.key;

    auth_basic "protected full node";
    auth_basic_user_file /etc/nginx/conf.d/full-node-pass;

    location /eth-1 {
        proxy_pass http://33.33.33.33:8545/;
    }

    location /eth-2 {
        proxy_pass http://44.44.44.44:8545/;
    }
```

Save SSL private key as `ssl_certificate_key /etc/nginx/conf.d/cert.key`, and SSL public key as `/etc/nginx/conf.d/cert.pem`. This will enable SSL protection for your connection. To deal with authentication, I use basic auth. Generate user file with:
```console
printf "{username}:$(openssl passwd -crypt {password})" > full-node-pass
sudo mv full-node-pass /etc/nginx/conf.d
```
For example: `printf "fullnode:$(openssl passwd -crypt nodepas)" > full-node-pass`. The basic auth will be enabled in Nginx with:
```nginx
auth_basic "protected full node";
auth_basic_user_file /etc/nginx/conf.d/full-node-pass;
```

Test config file and restart Nginx:
```
sudo nginx -t
sudo systemctl restart nginx
```

Check connection from your local machine (require `geth`)
```
$ geth attach https://fullnode:nodepas@fullnode.hiliamtech.com/eth-1
Welcome to the Geth JavaScript console!
...
To exit, press ctrl-d
>
```

## Setup monitoring service
Create project:
```console
mkdir node-gateway
cd node-gateway
npm init -y
touch main.js
```
Install required dependencies:
```console
npm install web3
npm install discord.js
```
Read  [this](https://discordpy.readthedocs.io/en/stable/discord.html) and create a Discord bot to get the bot token. Then, update `main.js` with:
```js
const Web3 = require('web3');
const axios = require('axios');
const { Client, Intents } = require('discord.js');

const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));

const channelID = '884022210142936081'
const token = 'your.bot.discord.token'

const rpcs = [
    "https://fullnode:nodepas@fullnode.hiliamtech.com/eth-1",
    "https://fullnode:nodepas@fullnode.hiliamtech.com/eth-2"
]

const getReportFunc = async function () {
    const client = new Client({ intents: [Intents.FLAGS.GUILDS] });
    client.once('ready', () => {
        console.log('discord bot is ready!');
    });
    await client.login(token);
    const channel = await client.channels.fetch(channelID);

    return async function (rpc, isOk, lastStatus) {
        const paths = rpc.split('/');
        const name = paths[paths.length - 1];

        console.log("syncing:", name, isOk);
        if (lastStatus[name] !== isOk) {
            let msg = ":scream: **" + name + "** is down!"
            if (isOk === true) {
                msg = ":yum: **" + name + "** is up!"
            }
            await channel.send(msg);
        }

        lastStatus[name] = isOk;
        return lastStatus
    }
}

const start = async function () {
    let report = await getReportFunc();
    let lastStatus = {};

    while (true) {
        console.log('checking syncing status ...');

        for (let id in gethRPCs) {
            try {
                let web3 = new Web3(new Web3.providers.HttpProvider(gethRPCs[id]));
                const syncing = await web3.eth.isSyncing();
                lastStatus = await report(gethRPCs[id], syncing === false, lastStatus);
            } catch (e) {
                lastStatus = await report(gethRPCs[id], false, lastStatus);
            }
        }
    }
}

start();
```
Finally, start the program with `npm start`. The result should be:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1632208859997/6qZw95ppt.png)

Yay, that's all. Thank you for your reading.
