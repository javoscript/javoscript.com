+++
title = "How I setup my own mail server in DigitalOcean (and almost fail at the very last step)"
path = "blog/post/how-i-setup-my-own-mail-server-in-digitalocean-and-almost-fail-at-the-very-last-step"

[taxonomies]
categories = ["blog"]
tags = ["devops", "docker", "mailing"]
+++

> TL;DR, Fully functional mail server with web client configured with Docker in DigitalOcean, using Mailgun as a relay server

**Don't feel like reading? Check out the Github repo [here](https://github.com/javoscript/mail-server)!**

<!-- more -->

(This is not intended to be a super detailed step by step guide to setting your own mailserver, but to share my own experience with this project.)

There comes a time for every little project when a domain name for it's website has to be chosen. And after that, it's time to setup your nice and unique email account with the shiny new domain, right?

Setting up a landing page with a pretty and responsive design is really easy nowadays, with a lot of templates to chose from if design is not your thing. Firing up a virtual machine in any of the popular hosting services and configuring the DNS records is straight forward.

But when wanting to implement mailing with your own domain, lot's of options suddenly appear, each with it's benefits and drawbacks, different unique offerings, pricing, etc... Postfix, Sendmail, Sendgrid, Mailgun, your own SMTP, third party options, APIs, mailing as a service, web clients. How to pick what's best for you?

Luckily for me, as a developer myself, the choice was really easy! Don't pay for it.. Implement it! So out in the quest for the easiest and most feature-packed solution I was. Having used Docker for some time in the web development and server setup space, I knew there had to be a pre-baked solution for my situation. After some digging, I came across an open source project that claimed to be what I was looking for. As they put it on [their website](https://mailu.io/index.html):

> Mailu is a simple yet full-featured mail server as a set of Docker images. It is free software (both as in free beer and as in free speech), open to suggestions and external contributions. The project aims at providing people with an easily setup, easily maintained and full-featured mail server while not shipping proprietary software nor unrelated features often found in popular groupware.

So, the only thing left was to get a server within the internet's reach where to put this Docker containers to work and start sending mails from my own server. I've used DigitalOcean to host other web projects in the past, with low prices and reasonably good performance and support. So I spun up the most basic standard droplet [for $5/month](https://www.digitalocean.com/pricing), and on I went to configure and tie everything up.

### First, NGINX as a reverse proxy
The default and easy way to setup Mailu on your server binds all the necessary ports of the host machine directly into the dockerized services. This was an issue for me, as I wanted to also host some websites on the same server. So I was forced into either using some unconventional ports and tinkering with more configuration or putting a reverse proxy as the gatekeeper to my server.

I've done this before, using [this awesome dockerized NGINX reverse proxy](https://github.com/jwilder/nginx-proxy). I also used the [Letsencrypt NGINX proxy companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion) for automatic SSL certificates creation and renewal. Both are really easy to set up with a `docker-compose.yml` file. You can find the configuration I used on my [github](https://github.com/javoscript/mail-server).

When you run the containers, with `docker-compose up -d` (to leave the containers running in `detached` mode, in the background), it binds ports 80 and 443 (standard HTTP and HTTPS ports) to the NGINX reverse proxy container, which then does it's magic.

Once I had that running, I can fire up other containers with some special environment variables so that they automagically *"register"* themselves to the reverse proxy. The ones I used were `VIRTUAL_HOST`, `LETSENCRYPT_HOST` and `LETSENCRYPT_EMAIL`.

### Initial DNS configuration
With the web server listening on ports 80 and 443 on my DigitalOcean droplet, now we have to configure the DNS records so that web traffic reaches our VM.
Doing this is pretty straight forward:
* **Point to DigitalOcean nameservers from your domain registrar** ([quick guide](https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars)). Basically, just set up `ns1.digitalocean.com`, `ns2.digitalocean.com`, `ns3.digitalocean.com` as the name servers for your domain.
* **Setup your domain inside DigitalOcean to direct to your droplet** in [your DO panel](https://cloud.digitalocean.com/networking/domains), registering your domain and creating an A record ponting to your droplet's IP.

Once this is done, and the DNS configuration propagates (may take up to some hours) you should be able to open your domain on the browser and see an NGINX error page. This is fine, and means that you are now reaching your VM through the web!

![nginx503.png](/images/nginx503.png)*This means everything is working fine so far*

### Running Mailu
Getting all the Mailu containers running is simple. Copy their docker-compose.yml and .env files from their [documentation](https://mailu.io/compose/setup.html), configure all the environment variables, and fire them up. It’s tunning these environment variables which can be a little tricky. Here are the tricky ones I had to set up.

```
# /root/mailu/.env

TLS_FLAVOR=mail

ADMIN=true

RELAYNETS=172.18.0.0/16 # Docker's subnet were the containers are running
RELAYHOST=[smtp.mailgun.org]:587
RELAY_LOGIN=postmaster@your-domain.com
RELAY_PASSWORD=yourSecretKey # Get it from mailgun's panel
```

All the  `RELAY` stuff is just necessary for the last step, when I set up Mailgun as a relay server for mail sending. If you decide to host this in another cloud provider, it may not be necessary. The other important variable is `TLS_FLAVOR`. This, in addition to other configuration in the `docker-compose.yml` file, sets up the necessary certificates.

The other important bits of configuration are in the `docker-compose.yml` file. This is pretty much the default one, except for the port binding in the `front` service, and some volume mounting. The important bits are here:

```yml
# /root/reverse-proxy/docker-compose.yml

version: '2'

services:
  front:
    image: mailu/nginx:$VERSION
    container_name: mailu_front
    restart: always
    env_file: .env
    ports:
    - "110:110"
    - "143:143"
    - "993:993"
    - "995:995"
    - "25:25"
    - "465:465"
    - "587:587"
    expose:
      - "80"
      - "443"
    volumes:
      - /root/reverse-proxy/certs/mail.your-domain.com/key.pem:/certs/key.pem
      - /root/reverse-proxy/certs/mail.your-domain.com/fullchain.pem:/certs/cert.pem
      - /root/reverse-proxy/certs/mail.your-domain.com/dhparam.pem:/certs/dhparam.pem
    environment:
      - VIRTUAL_HOST=mail.your-domain.com
      - LETSENCRYPT_HOST=mail.your-domain.com
      - LETSENCRYPT_EMAIL=your@email.com
    networks:
      - default

...

networks:
  default:
    external:
      name: reverse-proxy_default
```

The volume mounting configuration is **really** important. Here, I'm sharing the certificates that were automatically generated by the [Letsencrypt NGINX proxy companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion) with the Mailu containers, so that they can use them for encryption configuration.

It's also important to attach the *front* service to the reverse proxy network (in my case, called `reverse-proxy_default`). Without this, the reverse proxy running in front of all these containers won't be able to route traffic to them.

###  Using Mailgun as relay host
Once I got up to this point (without the `RELAY` variables set up in the `.env` file for the Mailu containers), I was ready to try my mail server. I was receiving mails perfectly from other accounts (@gmail. com, for example). But my outgoing emails were not sent successfully.

After debugging a little, and inspecting the logs (specifically the logs from the `smtp` service), I narrowed the problem to this:
```
smtp_1_3d8b20f9756e | postfix/smtp[121]: connect to gmail-smtp-in.l.google.com[172.217.197.27]:25: Operation timed out
smtp_1_3d8b20f9756e | postfix/smtp[121]: connect to alt1.gmail-smtp-in.l.google.com[2800:3f0:4003:c02::1b]:25: Address not available
smtp_1_3d8b20f9756e | postfix/smtp[121]: connect to alt1.gmail-smtp-in.l.google.com[172.217.192.26]:25: Operation timed out
smtp_1_3d8b20f9756e | postfix/smtp[121]: connect to alt2.gmail-smtp-in.l.google.com[74.125.193.27]:25: Operation timed out
smtp_1_3d8b20f9756e | postfix/smtp[121]: 4D8BD101050: to=, relay=none, delay=544, delays=454/0.01/90/0, dsn=4.4.1, status=deferred (connect to alt2.gmail-smtp-in.l.google.com[74.125.193.27]:25: Operation timed out)
```

The emails were being processed but got stuck once the communication with google (in this case) had to take place. I started to suspect of DigitalOcean blocking some ports by default in their droplets, so I wrote them a support ticket, asking if port 25 was blocked for outgoing traffic. This was their answer:

> Hello,
> Thank you for reaching out to us. We're very sorry that you are facing issues with SMTP.
>
> Stopping spam is a constant fight and due to this, your account has restrictions specifically on ports 25, 993, 995 and 465. However, you are be able to send out mail using port 587. You will need to open the port in your firewall.
>
> We realize this is inconvenient, but many customers in your position move their mailing activities to a third party service such as SendGrid or similar which processes such mail separately from their droplet. I'm sorry for the frustration but we're not able to lift these port restrictions at this time.
>
> In terms of a workaround, here are a few alternatives:
> 1. Utilize port 587 for SMTP relay via another mail provider, for example G Suite/Gmail, Mailgun, etc. We have a guide on doing so using Postfix [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-mail-relay-with-postfix-and-mailgun-on-ubuntu-16-04)
> 2. Configure your app or service to send mail directly using either a SMTP client connection (typically using port 587), or API call via another mail provider such as Sendgrid, Mailgun, Mandrill, etc.
>
> Please note that with this restriction in place on port 25, mail servers hosted here will be unable to directly relay email to other mail servers, as communication between mail servers typically takes place on port 25.
>
> We think the API is the best solution, as it is honestly more scalable and what we would use if we wanted to "future proof" the project.
>
> Please feel free to reach out to us via this ticket if you have further queries or concerns, we will be around to help you out!
>
> Please let us know if you have any additional questions, and have a wonderful day!

Later, I asked them if the block on port 25 was permanent, or if there was a way for me to unblock it. They said it will be blocked forever. **So there is no way of sending email directly from my own mail server hosted in DigitalOcean.**

I was not ready to give up on my quest. I've used [Mailgun](https://www.mailgun.com/)'s free tier for sending emails in the past. And, as DigitalOcean's support team recommended, I should be able to use it as a relay host for routing my ougoing emails. The only restriction with the free tier is that you only can send 10.000 emails per month (a lot more of what I'm planning to...). Also, in the free plan, you get a shared IP, which can get into any of the IP blacklists if any other using it starts spamming (it happened to me once, but a simple support ticket in the Mailgun site explaining that the IP they assigned to me was in a blacklist was enough for them to assign me a clean new one :) ).

I followed the [tutorial they pointed me to](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-mail-relay-with-postfix-and-mailgun-on-ubuntu-16-04), and after resolving some errors related to "*relay host login*" (copying and executing the /mailu/relay_conf.sh file in the `smtp` container), I was able to send emails from my mail server, using Mailgun as a relay host.

### Conclusion
I guess the final result is not the optimal one. I'm limited by Mailgun's free tier mail quota, and using a shared IP for mail sending. I could do the same thing in another provider (not DigitalOcean), where the necessary ports are not blocked, and get a dedicated IP. But, for now, this solution is enough for me. And I learned a little bit more of server and email configuration.
