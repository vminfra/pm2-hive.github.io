---
layout: docs
title: Graceful restart/reload/stop
description: How signals are handled in PM2
permalink: /docs/usage/signals-clean-restart/
---

To allow gracefull restart/reload/stop processes, make sure you intercept the **SIGINT** signal and clear everything needed (like database connections, processing jobs...) before letting your application exit. 

```javascript
process.on('SIGINT', function() {
   db.stop(function(err) {
     process.exit(err ? 1 : 0);
   });
});
```

Now `pm2 reload` will become a gracefulReload.

## Explanation: Signals flow

When a process is stopped/restarted by PM2, some system signals are sent to your process in a given order.

First a **SIGINT** a signal is sent to your processes, signal you can catch to know that your process is going to be stopped. If your application does not exit by itself before 1.6s *([customizable](http://pm2.keymetrics.io/docs/usage/signals-clean-restart/#customize-exit-delay))* it will receive a **SIGKILL** signal to force the process exit.

## Cleaning states and jobs

As stated before, if you need to clean some stuff (stop intervals, stop connection to database...) you can intercept the SIGINT signal to prepare your application to exit.

Here is a sample on how to intercept the SIGINT signal with Node.js:

```javascript
//[...]

process.on('SIGINT', function() {
  // My process has received a SIGINT signal
  // Meaning PM2 is now trying to stop the process

  // So I can clean some stuff before the final stop
  clearInterval(my_interval);
  my_db_connection.close();
  current_jobs.save();

  setTimeout(function() {
    // 300ms later the process kill it self to allow a restart
    process.exit(0);
  }, 300);
});
```

## Customize exit delay

Via CLI, this will lengthen the timeout to 3000ms:

```bash
$ pm2 start app.js --kill-timeout 3000
```

Via [JSON declaration](http://pm2.keymetrics.io/docs/usage/application-declaration/):

```json
{
  "apps" : [{
    "name"         : "api",
    "script"       : "app.js",
    "kill_timeout" : 3000
  }]
}
```

## Note about SIGKILL

The **SIGKILL** event cannot be intercepted in your application (newer Node.js version will throw an error if you are trying to install a signal handler on SIGKILL or SIGSTOP)

## Note about Cluster mode and Gracefull Actions

For some unknown reason, in cluster mode (if someone has an explanation we are interested) Node will not call the callback for SIGINT event in some situation. Here you have some example because it isnt easy to understand :

This callback here will be called **ONLY if your server has recevied one request**:

```javascript
var http = require('http');

http.createServer( (req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('okay');
}).listen(8080);

process.on('SIGINT', function() {
   console.log('SIGINT received');
});
```

This callback here will be called:

```javascript
var http = require('http');

http.createServer( (req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('okay');
}).listen(8080);

setInterval(function () {
 // even empty we do not care
}, 1000);

process.on('SIGINT', function() {
   console.log('SIGINT received');
});
```
