---
layout: post
title: Ubuntu Email
description: Sending email from the command-line in Ubuntu is a breeze --- but there's just too many choices
<!-- modified: 2015-02-20 -->
tags: [ubuntu, email, command line]
image:
  feature: ubuntu-email.jpg
  credit:
  creditlink: 
---

# Setting Ubuntu to send email

After setting up RAID on an Ubuntu server at home, I wanted the server to email me if there were any problems with the drives.

Rather than installing and configuring postfix or some other MTA, I settled for `ssmtp`.

Installation and testing is as simple as:

{% highlight bash %}
$ sudo apt-get install mailutils ssmtp
$ sudo dpkg-reconfigure ssmtp
$ echo "This is a test" | mail -s test your-address@gmail.com
{% endhighlight %}

Check your spam folder for the messages though!

Configuration lives in `/etc/ssmtp/ssmtp.conf`.

#### References

* ['Ask Ubuntu' forum post](https://askubuntu.com/questions/7993/how-can-i-setup-a-mail-transfer-agent)
* [ssmtp config documentation](http://manpages.ubuntu.com/manpages/jaunty/man5/ssmtp.conf.5.html)
