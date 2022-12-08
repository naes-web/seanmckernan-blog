---
title: "Selfhosted Blog Using Nginx and Hugo"
date: 2022-12-07T21:09:50Z
slug: 2022-12-07-nginx-congig-for-hugo
draft: false
type: posts
categories:
  - tech
tags:
  - linux
  - selfhost
  - nginx
  - hugo
---

Let me start with a disclaimer. I am not a webdev, I know HTML and CSS (kinda), and I have a *basic* understanding of JavaScript. I was still able to pull this blog off through some trial and error. The process was a learning experience, and I hope to document it with this post, maybe someone will find it useful. I'm not the first person to write about this either, a quick google of something like "nginx hugo ssl" will pull up loads of other guides.

I originally considered selfhosting Drupal, but after some digging I found it didn't really fit my needs. I just needed a static website to serve mainly text, to document my learning process and share with others. I didn't want to get side tracked messing around with a LAMP or LEMP stack just yet. Then I stumbled across static site generators. I looked a Jekyll, but decided it required too many dependencies and finally, I settled on Hugo. 

Hugo can take simple markdown files and spit out a website like this one in fractions of a second, all with a single binary.

# Prerequisites
- This post assumes some level of familiarity with Linux
- You have a registered domain name.
- You have a VPS running Ubuntu 22.04LTS (config should be similar on other distros, or older versions of Ubuntu. Probably...).
- You can ssh into a non-root account on your VPS.
- Configuring DNS records, ssh-keygen, and other initial server setup is outside the scope of this guide.

# Installing Hugo
I only have Hugo installed on my main PC, meaning it does not need to be hosted on the VPS, which saves space. There are many ways to install the package, but the method I used (and the method recommended by [Hugo](https://gohugo.io/installation/linux/)) is the Snap package manager. To install Snap, simply run
```
sudo apt update
sudo apt install snapd
```
Then installing Hugo is as simple as running
```
sudo snap install hugo
```
- Note that this will install the extended edition, which has some extra features. [It is recommended to install the extended version](https://gohugo.io/installation/linux/#editions). 
# Making your Fist Site
## Adding a Theme
Make sure you are in the directory that you want to build the new site in, then run 
```
hugo new site <sitename>
``` 
This will generate a new directory, with the skeleton of your new site inside. The next step is to install a theme. Check out https://themes.gohugo.io/ and pick one you like. I went with a theme named [smol](https://github.com/colorchestra/smol), as it features no JavaScript, no tracking or analytics and no external dependencies. To install a theme, ```cd``` into the directory ```<sitename>/themes``` and run
```
git clone https://github.com/<random-user>/<your-theme>
```
Replace the URL as appropriate for your selected theme, and follow the configuration guide that comes with it. You will be editing the ```config.toml``` file to set parameters and make use of all your chosen theme has to offer. Most themes come with an example ```config.toml``` in their respective directory within the ```themes``` directory. 
### Adding a Post
Creating new posts is as simple as running
```
hugo new posts/first-post.md
```
This will create a new post in the ```./content/posts/``` directory. We edit these posts using [Markdown](https://www.markdownguide.org/getting-started/). This is a lightweight markup language, used to add formatting to plaintext documents. I personally have been using Markdown to keep notes for a number of years, its really easy to pick up with simple syntax, allowing us to focus on the content of our blog. You can edit ```.md``` files using the text editor of your choice. I personally use [ghostwriter](https://ghostwriter.kde.org/). 

At the top of the file you should see some automatically generated text, this is called the Front Matter, which is better explained by the [docs](https://gohugo.io/content-management/front-matter/). I believe it changes depending on what theme you have installed, but for example, heres mine
```YAML
---
title: "Selfhosted Blog Using Nginx and Hugo"
date: 2022-12-07T21:09:50Z
slug: 2022-12-07-nginx-congig-for-hugo
type: posts
draft: true
tags:
  - linux
  - selfhost
  - nginx
  - hugo
---
```
The Front Matter (in this case) is in YAML format, but keep in mind your config could be different, either TOML (like the config.toml we edited earlier) or JSON. It's pretty self explanatory; 

```title:```is the title printed at the top of the page.

```date:``` is the date and time when you created the file.

```slug:``` is the URL that will be used when you build the site

```type``` is where the file will sit under the ```<your-site>/content/``` directory. In this case ```<yoursite>/content/posts/```

```draft:``` Marks the post as a draft, meaning it will not get included in the build. Make sure to change this to ```false``` when you are ready to publish your site.

```tags:``` adds tags to the post, allowing users to browse your content based on tags
## Building the Site for Testing
After you're done writing your fist post, save it and ```cd``` to the root directory of the website. Then run
```
hugo server -D
```
With the default configuration, this will start up a webserver on ```localhost:1313```, the ```-D``` flag makes sure Hugo builds documents marked as drafts too, allowing you to preview changes to the site as you make them. I usually have my browser and Markdown editor side-by-side, so I can check formatting. You can edit anything in the root directory of the site and view the changes almost immediately this way.

# Nginx Config for SSL
## Deploying the Build to our VPS
The [official docs on hosting and deployment](https://gohugo.io/hosting-and-deployment/) cover most use cases. In my case I have SSH access to my server using a non-root account, so I followed the simple guide on creating a shell script using rsync. Replace the values in <> brackets as appropriate. Make sure to save this script to the root directory of your site. ```nano deploy``` to create the file, edit as needed, save it and run ```chmod -x deploy``` to make it executable.
```sh
#!/bin/sh
USER=<your-user>
HOST=<your-server>
DIR=/var/www/<website-name>

hugo && rsync -avs --delete public/ ${USER}@${HOST}:/${DIR}

exit 0
```

First off, the ```hugo``` command. This builds the site to the ```public``` directory by default, but you can edit this with the ```--destination``` flag.

Now for ```rsync```. For a great guide on how ```rsync``` works see [this document](https://rsync.samba.org/how-rsync-works.html). The flags ```rsync -avs --delete``` are ```--archive```, ```--verbose```, ```--compress```. This will recursively transfer all the files from the ```public/``` directory. The files are transferred in archive mode, meaning symbolic links, attributes, permissions, etc. are preserved in the transfer. And finally, the ```--delete``` flag. This deletes files in the destination directory if they do not exist at the source. 

So, all in, this script will build my site, take it from my PC, transfer it to my VPS at the specified directory, and delete any unnecessary files at the destination. **Now to set up Nginx to serve our static site.**

## Nginx Installation and Config
This post is getting long, so I'll try to keep this section brief. Serving static content with Nginx is actually really easy, took me a while to understand what the config files were doing. I found [this guide from freeCodeCamp](https://www.freecodecamp.org/news/the-nginx-handbook/#introduction-to-nginx) really helpful. 

Use ssh to connect to your server, then install Nginx. It's available on Ubuntu's default repository, so we can simply run ```sudo apt install nginx```. If you have a firewall enabled on your server, you need to configure it to allow access to Nginx. 

Next we need to create and edit the config file for our site. Config files for sites are stored in ```/etc/nginx/sites-available/``` and symlinked to ```/etc/nginx/sites-enabled```. Use the ```cp``` utility to make a copy of the default config in case you mess up.
```sh
cd /etc/nginx/sites-available/
cp default your-site
vim your-site
```
You should now see a basic server configuration with lots of comments, don't worry too much about them right now (delete them if you want), we want to get a config that looks something like this
```sh
server {
       listen 80;		# listen on port 80 for ipv4
       listen [::]:80; 	# listen on port 80 for ipv6

       server_name your-site.com;		# this should be the URL of your website

       root /var/www/your-site.com;	# path to the directory of hugo build
       index index.html;				# hugo will generate index.html

       location / {
			   #attempt to serve request as file, then dir, then 404
               try_files $uri $uri/ =404;	
       }
}
```
Enabling this site is as simple as symlinking the file we created to the `sites-enabled` directory. It is best practice to use the absolute file paths when creating symlinks so they don't get messed up. We also need to remove the `default` symlink from `sites-enabled`. 
```sh
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/your-site /etc/nginx/sites-enabled/your-site
```
Make sure Nginx is running and enable it at boot with
```
sudo systemctl start nginx
sudo systemctl enable nginx
```
Now, if you have your DNS records correct, you should be able to visit `your-site.com` and see what you've built.

## Adding SSL
For this example we are going to use Certbot, built by the EFF. Visit [this link](https://certbot.eff.org/) for specific instructions for your setup. For Ubuntu 20.04LTS; 
1) You need to install snapd on your server, if you don't already have it.
2) Update snapd.
2) Remove any old Certbot packages. 
3) Install Certbot using snap.
4) Move Certbot in order to run it.
5) Get and install the certificates, which automatically edits your Nginx config.

That looks something like this
```
sudo apt update
sudo apt install snapd
sudo snap install core; sudo snap refresh core
sudo apt remove certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```
Certbot should renew the certs automatically, run `sudo certbot renew --dry-run` to test this out.

# Conclusion
Finally, you should be able to visit `https://your-site.com` and browse your shiny new site. Thank you for reading this far, and I hope you find this guide useful. **Keep a look out for my next post about hardening your server and Nginx config** 