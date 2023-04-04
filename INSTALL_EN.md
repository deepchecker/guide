# Deepchecker

DeepChecker is an application that consumes a lot of resources. All operations take place in dozens of separate processes, and therefore a VPS with at least 6 vCPUs and 6GB of RAM is required for the application to work correctly. OS: Ubuntu 20.04
You can buy VPS here:

- https://www.cherryservers.com/
- https://www.digitalocean.com/
- https://www.ovhcloud.com/
- https://www.hetzner.com/

# Domain name

For DeepChecker to work correctly, you need a domain name with an installed SSL certificate. The software is also able to work on the IP address, but in this case, you will be deprived of the opportunity to receive notifications from the telegram bot, since the telegram API only works using the https protocol.

You need to buy a domain from any registrar convenient for you and add an A-record from the IP of your VPS server. Specify the following settings:

- Host: **@**
- IP: **1.1.1.1** _(IP address of your VPS server)_

## [1.1] Installing DeepChecker on a clean Ubuntu 20.04 image

All of the commands below must be entered in the console of your server.

Create a deepchecker folder
```
sudo mkdir /home/deepchecker
```
Now you need to upload the archive with deepchecker to the server in `/home/deepchecker`.

Unzip the archive
```
sudo apt-get update
sudo apt-get install unzip
unzip deepchecker.zip
```

Open `.env`.
```
sudo nano .env
```

Insert in `APP_URL=` the domain where you plan to run the checker in the format https://domain.com/ (required via https), or in the format http://yourserver/ (if you plan to deploy the checker on an IP address without a domain)

Close nano with `ctrl + x`, then press `Y` to save the changes.

### [1.2] If the domain is present
**Run the following commands, replacing 'domain.com' with your domain**

```
cd /home/deepchecker
chmod +x scripts/install.sh
./scripts/install.sh domain.com //replace domain.com with your domain
```

### [1.2] If the domain is missing

```
cd /home/deepchecker
chmod +x scripts/install.sh
./scripts/install.sh
```

### Checker update (if a new version is released)

Download the archive with the new version to your device.
Open `.env` in the archive with the new version of deepchecker and compare it with the existing `.env` of the old version. If there are new lines in the first one, add them to `.env`.
Upload the archive with the new version of deepchecker to `/home/deepchecker`.
Unpack all files from the new archive except `.env` to `/home/deepchecker` replacing old files.

Enter in the console:
```
cd /home/deepchecker
chmod +x scripts/remove.sh
./scripts/remove.sh
```

Further, the process is similar to the installation.
Follow step 1.2 (see above)

# Authorization

After the first installation, the following administrator account is available to you:

Login: user@site.ru
Password: password

# Beginning of work

## Proxy
**We strongly recommend to use about 60 ipv4 http proxies (preferably private ones).**

You can buy proxies here:
- https://proxyline.net/ipv4/ (you can pay with crypto, fast issuance)
- https://proxy-seller.io/

Add proxies on the **/proxies** page.

## Telegram - bot
- Find **@botfather** in telegram, create your bot, get the bot token in the format `111111111:DASKDI2109290DWR10-4013389120-32`
- Go to the checker admin panel using the link **/settings**, fill in the fields **'Bot name (specify without @)'** and **'Token'**, click the **'Update'** button.
- Click the button **'Link Telegram webhook'**.
- Write **/start** in private messages to your bot.
- Return to the checker admin panel to the **/settings** page. Fill in the field 'Notify me if the wallet balance exceeds' - specify the threshold amount in $.
- Copy the text from the field **/pair 0000000000000000000000000** and send it to your df in the bot.
- If everything is correct - the bot will send you a message 'Linking successfully completed' and from that moment on it will send you notifications.

**You can start working**
