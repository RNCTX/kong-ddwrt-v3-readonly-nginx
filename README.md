# kong-ddwrt-v3-readonly-nginx


Hi folks, I figured someone else might find this useful.

I have a Netgear R7000 running DD-WRT and was planning on using it for a reverse proxy, among other things, to host various virtual machines I use for web development in my house. Optimally, I should not be relying on the suggested solution of a USB stick to hold things, since the router has plenty of internal flash memory for a few packages and SSL keys.  But one would want to run that internal flash as read-only to not wear it out, obviously.

Lighttpd is included in Kong's build(s) of DD-WRT, but unfortunately it does not support SSL passthrough like Nginx does.  The problem with making the switch from Lighttpd is that Nginx as-compiled by the maintainer for the Entware DD-WRT Nginx package points to /opt for the log, pid, and lock files by default.  

https://github.com/Entware/entware-packages/blob/master/net/nginx/Makefile ...

```
define Build/Configure
	( cd $(PKG_BUILD_DIR) ; \
		$(if $(CONFIG_NGINX_LUA),LUA_INC=$(STAGING_DIR)/opt/include LUA_LIB=$(STAGING_DIR)/opt/lib) \
		./configure \
			--crossbuild=Linux::$(ARCH) \
			--prefix=/opt \
			--modules-path=/opt/lib/nginx \
			--conf-path=/opt/etc/nginx/nginx.conf \
			$(ADDITIONAL_MODULES) \
			--error-log-path=/opt/var/log/nginx/error.log \
			--pid-path=/opt/var/run/nginx.pid \
			--lock-path=/opt/var/lock/nginx.lock \
			--http-log-path=/opt/var/log/nginx/access.log \
			--http-client-body-temp-path=/opt/var/lib/nginx/body \
			--http-proxy-temp-path=/opt/var/lib/nginx/proxy \
			--http-fastcgi-temp-path=/opt/var/lib/nginx/fastcgi \
			--http-uwsgi-temp-path=/opt/var/lib/nginx/uwsgi \
			--http-scgi-temp-path=/opt/var/lib/nginx/scgi \
			--with-cc="$(TARGET_CC)" \
			--with-cc-opt="$(TARGET_CPPFLAGS) $(TARGET_CFLAGS)" \
			--with-ld-opt="$(TARGET_LDFLAGS)" \
			--without-http_upstream_zone_module \
	)
endef

define Package/nginx/install
	$(INSTALL_DIR) $(1)/opt/sbin
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/opt/sbin/nginx $(1)/opt/sbin/
	$(INSTALL_DIR) $(1)/opt/etc/nginx
	$(INSTALL_DATA) $(addprefix $(PKG_INSTALL_DIR)/opt/etc/nginx/,$(config_files)) $(1)/opt/etc/nginx/
	$(INSTALL_DIR) $(1)/opt/etc/init.d
	$(INSTALL_BIN) ./files/S80nginx $(1)/opt/etc/init.d/
	$(INSTALL_DIR) $(1)/opt/share/nginx/html
	$(INSTALL_DATA) $(PKG_INSTALL_DIR)/opt/html/{index,50x}.html $(1)/opt/share/nginx/html
	$(INSTALL_DIR) $(1)/opt/var/{run,lock} $(1)/opt/var/log/nginx $(1)/opt/lib/nginx
	$(INSTALL_DIR) $(1)/opt/var/lib/nginx/{body,proxy,fastcgi}
ifeq ($(CONFIG_NGINX_NAXSI),y)
	$(INSTALL_DIR) $(1)/opt/etc/nginx
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/nginx-naxsi/naxsi_config/naxsi_core.rules $(1)/opt/etc/nginx
	chmod 0640 $(1)/opt/etc/nginx/naxsi_core.rules
endif
```

This means that before Nginx will write a log anywhere else, it checks for its log folder under /opt to be writable.  If you wish to run packages on /opt mounted read-only to avoid wearing out the internal flash, as I do, then that won't work. Nginx never releases that default logfile location. As long as Nginx is running, you can't remount /opt as read-only.

If only Nginx were trying to put its default log under the ramdisk on /tmp, this would be fixed and Nginx would be totally usable with /opt mounted on the internal flash as read-only. You could then, say... redirect Nginx logs to a remote syslog on another machine, which makes way more sense than writing logs to a USB stick (and also eliminates the obvious physical security issue with putting SSL keys on a USB stick, too). Nginx supports syslog by default, so that would be a trivial (just config, no re-compile) change to make.

So this re-build of the Nginx binary does just that: points Nginx logs, lock, and pid to existing folders on the /tmp ramdisk.

How to use:


1. Enable jffs2 as /opt from Administration -> Management
2. [Install Entware](https://wiki.dd-wrt.com/wiki/index.php/Installing_Entware#Installation) to the jffs /opt
3. Install Nginx from optware like you normally would (opkg install nginx)
4. Replace the binary in /opt/sbin with the one linked here (rm -f /opt/sbin/nginx && scp user@192.168.1.xx:nginx /opt/sbin/nginx)
5. Modify your config file in /opt/etc/nginx to point the logs somewhere else, or disable the logs entirely, or whatever you want to do with them. **BIG FAT WARNING: if you do not do this, the logs will fill up all your RAM and (probably) crash your router, you must turn them off or send them somewhere else**.
6. Run nginx once so it can chown all of its directories on /opt to itself (/opt/etc/init.d/S80nginx start && /opt/etc/init.d/S80nginx stop)
7. In your Administration -> Commands -> Startup, add a command to read-only the internal flash at boot: (mount --bind /jffs/opt /opt && mount -o remount,ro /opt)
8. Reboot the router


Here's the Makefile diff if you wish to compile yourself...

```
ddwrt@debian:~/Entware/feeds/packages/net/nginx$ diff Makefile Makefile.1
232,235c232,235
< 			--error-log-path=/opt/var/log/nginx/error.log \
< 			--pid-path=/opt/var/run/nginx.pid \
< 			--lock-path=/opt/var/lock/nginx.lock \
< 			--http-log-path=/opt/var/log/nginx/access.log \
---
> 			--error-log-path=/tmp/var/log/nginx_error.log \
> 			--pid-path=/tmp/var/run/nginx.pid \
> 			--lock-path=/tmp/var/lock/nginx.lock \
> 			--http-log-path=/tmp/var/log/nginx_access.log \
257c257
< 	$(INSTALL_DIR) $(1)/opt/var/{run,lock} $(1)/opt/var/log/nginx $(1)/opt/lib/nginx
---
> 	$(INSTALL_DIR) $(1)/opt/lib/nginx
```

As of this writing, the current Entware build of Nginx is 1.12.2.  Far into the future, you probably want to use the diff to compile it yourself, as per the instructions on the Entware wiki for [Compiling From Sources](https://github.com/Entware/Entware/wiki/Compile-packages-from-sources)
