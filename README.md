# macos-server5-websocket
How to get reverse proxies and WebSockets working on MacOS Server 5

In case someone finds themselves in a similar situation here is how I got ​Web​Socket connections working via Apache​ in MacOS Server 5.2​.
And the solution ​i​s simple.

-------------------------------------------
The short version:​

​I use MacOS Server 5.2 (ships with Apache 2.4.23) to run a python Django application via the mod_wsgi module. 

I had been trying to setup proxypass and wstunnel​ in MacOS 10.12 & Server 5.2 to handle websocket connections via an ASGI interface server called Daphne running on localhost on port 8001.​

I wanted to reverse proxy any WebSocket connection to wss://myapp.local/chat/stream/ to ws://localhost:8001/chat/stream/

​From what I had read on all the forums and mailing lists that I had scoured was to simply make some proxypass definitions in the appropriate virtual host and make sure that the mod_proxy and mod_proxy_wstunnel modules were loaded and it would work.

Long story short - from what I understand all ​of ​this trouble came down to MacOS Server 5 and one major change:
"A single instance of httpd runs as a reverse proxy, called the Service Proxy, and several additional instances of httpd run behind that proxy to support specific HTTP-based services, including an instance for the Websites service."

All I need​ed​ to do​ to proxy the websocket connection​ was the following:

in:
/Library/Server/Web/Config/Proxy/apache_serviceproxy.conf

​Add (around line 297 (in the section about user websites, webdav):
   ProxyPass / http://localhost:8001/
   ProxyPassReverse / http://localhost:8001/

   RewriteEngine on
   RewriteCond %{HTTP:UPGRADE} ^WebSocket$ [NC]
   RewriteCond %{HTTP:CONNECTION} ^Upgrade$ [NC]
   RewriteRule .* ws://localhost:8001%{REQUEST_URI} [P]


​I then kicked over the service proxy:
sudo launchctl unload -w /Applications/Server.app/Contents/ServerRoot/System/Library/LaunchDaemons/com.apple.serviceproxy.plist
sudo launchctl load -w /Applications/Server.app/Contents/ServerRoot/System/Library/LaunchDaemons/com.apple.serviceproxy.plist

​And the web socket connections ​were instantly working!


​The long version:​ 

For many weeks I have been trying to get WebSocket connections functioning with Apache and Andrew Godwin's Django Channels project in an app I am developing. 

Django Channels is "a project to make Django able to handle more than just plain HTTP requests, including WebSockets and HTTP2, as well as the ability to run code after a response has been sent for things like thumbnailing or background calculation."

My interest in Django Channels came from my requirement of a chat system in my webapp. After watching a several of Andrew's demos on youtube and having read through the docs and eventually installing Andrew's demo django channels project I figured I would be able to get this working in production on our MacOS Server.

The current release of MacOS 10.12 and Server 5.2 ships with Apache 2.4.23. This comes with the necessary mod_proxy_wstunnel module to be able to proxy WebSocket connections (ws:// and secure wss://) in Apache and is already loaded in the server config file:

/Library/Server/Web/Config/apache2/httpd_server_app.conf

Daphne is Andrew's ASGI interface server that supports WebSockets & long-poll HTTP requests. WSGI does not.


With Daphne running on localhost on a port that MacOS isn't occupying (i went with 8001) the idea was to get Apache to reverse proxy certain requests to Daphne. 

Daphne can be run on a specified port (8001 in tis example) like so (-v2 for more feedback):
daphne -p 8001 yourapp -v2

I wanted Daphne to handle only the web socket connections (as I use currently depend on some apache modules for serving media such as mod_xsendfile). In my case the websocket connection was via /chat/stream/ based on Andrew's demo project.

From what I had read in MacOS Server's implementation of Apache the idea is to declare these proxypass commands inside the virtual host files of your "sites" in:
/Library/Server/Web/Config/apache2/sites/
In config files such as:
0000_127.0.0.1_34543_.conf
I did also read that any customisation for web apps running on MacOS Server should be made to the plist file for the required web app in:
/Library/Server/Web/Config/apache2/webapps/
In a plist file such as:
com.apple.webapp.wsgi.plist 
Anyway...

I edited the 0000_127.0.0.1_34543_.conf file adding:
   ProxyPass /chat/stream/ ws://localhost:8001/
   ProxyPassReverse /chat/stream/ ws://localhost:8001/

Eager to test out my first web socket chat connection I refreshed the page only to see an error printed in the apache log:

No protocol handler was valid for the URL /chat/stream/. If you are using a DSO version of mod_proxy, make sure the proxy submodules are included in the configuration using LoadModule.

I had read of many people finding a solution at least with Apache on Ubuntu or a custom install on MacOS.
I even tried installing Apache using Brew and when that didn't work I almost proceeded to install nginx.

After countless hours/days of googling I reached to the Apache mailing list for some help with this error. Yann Ylavic was very generous with his time and offered me various ideas on how to get it going. After trying the following:

SetEnvIf Request_URI ^/chat/stream/ is_websocket
RequestHeader set Upgrade WebSocket env=is_websocket
ProxyPass /chat/stream/ ws://myserver.local:8001/chat/stream/

I noticed that the interface server on port 8001 Daphne was starting to receive ws connections!
However in the client browser it was logging:

"Error during WebSocket handshake: 'Upgrade' header is missing"

From what I could see mod_dumpio was logging that the "Connection: Upgrade" and "Upgrade: WebSocket" headers were being sent as part of the web socket handshake:

mod_dumpio:  dumpio_in (data-HEAP): HTTP/1.1 101 Switching Protocols\r\nServer: AutobahnPython/0.17.1\r\nUpgrade: WebSocket\r\nConnection: Upgrade\r\nSec-WebSocket-Accept: 17WYrMeMS8a4ImHpU0gS3/k0+Cg=\r\n\r\n
mod_dumpio.c(164): [client 127.0.0.1:63944] mod_dumpio: dumpio_out
mod_dumpio.c(58): [client 127.0.0.1:63944] mod_dumpio:  dumpio_out (data-TRANSIENT): 160 bytes
mod_dumpio.c(100): [client 127.0.0.1:63944] mod_dumpio:  dumpio_out (data-TRANSIENT): HTTP/1.1 101 Switching Protocols\r\nServer: AutobahnPython/0.17.1\r\nUpgrade: WebSocket\r\nConnection: Upgrade\r\nSec-WebSocket-Accept: 17WYrMeMS8a4ImHpU0gS3/k0+Cg=\r\n\r\n

However the client browser showed nothing in the response headers.

I was more stumped than ever. 

I explored the client side jQuery framework as well as the  the Django channels & autobahn module to see if perhaps something was amiss, and then revised my own app and various combinations of suggestions about Apache and it's module. But nothing stood out to me.

​Then I reread the ReadMe.txt inside the apache2 dir:
/Library/Server/Web/Config/apache2/ReadMe.txt

​"​Special notes about the web proxy architecture in Server application 5.0:

This version of Server application contains a revised architecture for all HTTP-based services. In previous versions there was a single instance of httpd acting as a reverse proxy for Wiki, Profile, and Calendar/Address services, and also scting as the Websites service.​º​ With this version, there is a major change: A single instance of httpd runs as a reverse proxy, called the Service Proxy, and several additional instances of httpd run behind that proxy to support specific HTTP-based services, including an instance for the Websites service.

Since the httpd instance for the Websites service is now behind a reverse proxy, or Service Proxy, note the following:
​...
 It is only the external Service Proxy httpd instance that listens on TCP ports 80 and 443; it proxies HTTP requests and responses to Websites and other HTTP-based services.
​...
​"​

​I wondered if thus ServiceProxy had somet​hing to do with it. I had a look over:
/Library/Server/Web/Config/Proxy/apache_serviceproxy.conf 
and noticed a comment - "# The user websites, and webdav"​. 
I figured it wouldn't hurt to try adding the proxypass definitions & rewrite rules that people had suggested on the forums as their solution. 

   ProxyPass / http://localhost:8001/
   ProxyPassReverse / http://localhost:8001/

   RewriteEngine on
   RewriteCond %{HTTP:UPGRADE} ^WebSocket$ [NC]
   RewriteCond %{HTTP:CONNECTION} ^Upgrade$ [NC]
   RewriteRule .* ws://localhost:8001%{REQUEST_URI} [P]

​Sure enough after restarting the ServiceProxy it all started to work!​
-----------------------------------------------------------


Adam
