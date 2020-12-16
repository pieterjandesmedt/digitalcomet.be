---
modalID: "ssh-and-git-on-one-dot-com"
title: "Using SSH and Git on One.com"
date: "2018-03-12T14:58:26+01:00"
img: "ssh-and-git-on-one-dot-com.jpg"
category: Web Development
weight: 0
---

Usually it's a breeze to deploy a website using git. Until I had to do it for a one.com account.

<!--more-->

Ever since reading [Using Git to manage a web site](http://toroid.org/git-website-howto), I try to apply this to all websites I manage. It's so much easier to just run my `npm run publish` and have it build, commit, push and remotely checkout the website than mucking around with Filezilla. _(And yes, you should totally use [npm as your build tool](https://www.keithcirkel.co.uk/how-to-use-npm-as-a-build-tool/) too)_

5 different support reps said it couldn't be done, so I did it anyway, just to show them.

First of all, none of their support reps know that one can log in via the terminal using ssh. Most of them don't know git either. This conversation was fairly typical:

> - **Me**: I'm trying to login via ssh using the publickey/privatekey method. But it keeps asking for my password. Where do I put my public key?
> - **Support**: You're working with Filezilla?
> - **Me**: No, I'm using ssh and git
> - **Support**: Yes, but which program do you use to log in via SSH?
> - **Me**: Ermm... just ssh, in Terminal :-/
> - **Support**: I advise you to use Filezilla


_Are you kidding me?_

Second, the English speaking support reps seem a little more knowledgeable than the local ones - or maybe they just ask for a technician's help faster.

So, using a lot of Google-fu and some very cryptic answers of the one.com support team, I was able to piece together a solution that not only connects git through ssh, but also checks out the website after receiving a git update.

Great success!


## Enable SSH

> - **Support**: ssh is not included in your package, sir
> - **Me**: that's strange, because I can enable it in my control panel and I can log in through ssh in terminal.

First, you need to enable ssh/sftp. You can do that in the one.com [control panel](https://login.one.com/cp/).

It will send you (in an email) a link to create your ssh password. If you're managing the account for someone else, you'll first need to change the email address so that it sends that mail to you. Follow the instructions and remember that password. (Why this needs to be done with the extra email step is beyond me).

Then ssh into your account and log in using your login and password.

```
ssh <your-login>@ssh.<your-domain>
```

Now create a **file** on the server `~/.ssh/authorized_keys`.

Create a key pair on your local machine, for example using
```
ssh-keygen -b 4096 -t rsa
```

Save it under `~/.ssh/my-one-com-key`. Don't use a passphrase. Then add it to your ssh-agent using

```
chmod 600 ~/.ssh/my-one-com-key*
ssh-add ~/.ssh/my-one-com-key
```

Finally, paste the contents of the **public** key (normally `~/.ssh/my-one-com-key.pub`) on the one.com server into the `authorized_keys` file.

Now log out, then log in again. It should not ask for the password anymore.


## Set up git

Git is mentioned [literally nowhere](https://help.one.com/hc/en-us/search?utf8=%E2%9C%93&query=git) on one.com's support pages. But although they don't want you to be playing with the cool toys, it's there and you can use it.

So set up your local and remote repositories like [described here](http://toroid.org/git-website-howto).

I set up the remote repo in `~/website.git`.

In the post-receive hook use `GIT_WORK_TREE=/www git checkout -f`.


## Create your own post-receive

Now, when you do a `git push`, it's pushing the changes alright, but you'll notice the post-receive hook is not firing.

> - **Support**: yeah we don't allow users to execute scripts.
> - **Me**: indeed, it gives me a permission denied error
> - **Me**: but when I wrap it like `sh hooks/post-receive`, it runs correctly
> - **Support**: ...ok then

So, it _is_ possible to execute something if you wrap it in `sh`.

Now we just need to find a way to run this after git has finished pushing the changes.

Turns out you can make the `authorized_keys` file also execute a command upon connection with a key. But when you try to `git push` with such a command, it will error. The solution is simple:

We need a second key.

So go generate a second key. You could call it `~/.ssh/my-one-com-post-receive-key`. Now in the authorized_keys file on the server, add this line:

```
command="cd ~/website.git && sh hooks/post-receive" <contents of my-one-com-post-receive-key.pub go here>
```

Now you can connect using these two separate identities. One for git, and the other for executing the post-receive hook.

```
# this pushes your site using your first key
GIT_SSH_COMMAND='ssh -i ~/.ssh/my-one-com-key' git push web --force

# this runs the post-receive hook upon connection with the other key, then disconnects
ssh -i ~/.ssh/my-one-com-post-receive-key <your-login>@ssh.<your-domain>
```

_Voilà!_

Happy deploying!
