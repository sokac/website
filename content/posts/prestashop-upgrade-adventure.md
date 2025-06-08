+++
title = 'Prestashop 1.7.8.5 to Prestashop 8.2.0 Upgrade Adventure'
date = 2024-11-18T20:15:00-07:00
draft = false
description = "A detailed account of upgrading Prestashop e-commerce sites from version 1.7.8.5 to 8.2.0, including PHP compatibility challenges and the upgrade process."
+++

Over the weekend, I finally upgraded this server from Debian Buster to Debian
Bookworm. I also host two small Prestashop e-commerce webshops on this server.
Since the uptime is not the biggest concern, I can take a few hours of downtime
if needed, especially while it's night in Europe. So, I created a full backup,
took a cowboy hat, and did something I couldn't do at my day job: instructed
the server to perform the upgrade and, in the process, intermittently took all
hosted websites down.

# Issue #1 - Prestashop 1.7,8.5 doesn't work well with PHP 8.2

The Debian upgrade process was smooth - two reboots, and less than a minute of
downtime each time. The first issue showed up quickly - a bug in my patch to
show dual currency (EUR <> HRK conversion). But then, I realized that
the administration panel doesn't work. The logs files showed the following:

```
stderr: PHP Fatal error:  Uncaught Error: Failed opening required '/home/<redacted>/public_html/' (include_path='<redacted>') in /home/<redacted>/public_html/vendor/prestashop/autoload/src/Autoloader.php:103
stderr: Stack trace:
stderr: #0 /home/<redacted>/public_html/config/config.inc.php(83): PrestaShop\\Autoload\\Autoloader->load()
stderr: #1 /home/<redacted>/public_html/index.php(27): require('...')
stderr: #2 {main}
stderr:   thrown in /home/<redacted>/public_html/vendor/prestashop/autoload/src/Autoloader.php on line 103
stderr: PHP Warning:  require_once(/home/<redacted>/public_html): Failed to open stream: No such file or directory in /home/<redacted>/public_html/vendor/prestashop/autoload/src/Autoloader.php on line 103
```

Quickly glancing through documentation, I realized that I need to either
upgrade Prestashop to 8+ version, or downgrade PHP. I decided to upgrade it.
Before moving forward, I made another backup for the first webshop I was
upgrading - that way, I can easily restore it if I need to (hint: yes, I had to).

# Issue #2 - Upgrade Prestashop 1.7.8.5 using PHP 8.2

I attempted to do a CLI upgrade using PHP8.2, but I had to hack a few things
first - put the webshop in the maintenance mode, disable cache, etc.
Unfortunately, the upgrade process failed so I gave up and restored the backup.

I ended up installing php-7.4 alongside php-8.2, and instructed webshop to use
php-7.4 while I did the upgrade. That php downgrade allowed me to enter back to
admin panel, where I was able to update all the plugins, and prepare the
webshop for upgrade without fiddling with database directly. After a few
minutes, the upgrade process was completed. There were a few touch-ups needed
post-upgrade.

Once I tested end-to-end flows, I switched PHP back to php-8.2,, and the webshop
continue to work just fine. I repeated the same process for the other webshop
and didn't encounter any issues there.

# Conclusion

I was pleasantly suprised how smooth the experience was considering I was
running Debian that reached EOL a few months ago (I shouldn't really wait that
long!), and that I also had to upgrade Prestashop. Prestashop upgrade
experience using the 1-Click-Upgrade plugin was suprisingly smooth and overall
well designed. Kudos to the Prestashop maintainers!
