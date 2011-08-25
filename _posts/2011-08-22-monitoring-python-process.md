---
layout: post
title: "Monitoring Python Processes"
categories: 
  - python
  - monitoring
date: Aug 24, 2011
time: "08:45"
date_time: "24 Aug, 2011 // 20:45 PM"
---
Its often common to see some python process like a webserver process or a python
script running for a long time and you have no idea what its doing currently.
Although you have logging and print statements it does not help much to get a
sense of what the latency is all about. I personally encounter this everyday.
You brought in a new database or algorithm which worked well in you local / dev
/ staging environment, but as soon as you push to production, DANG... stuck..
You try a number of ways in shell etc and take a guess of what might be wrong
but you cant for sure find where exactly its getting stuck.

For these situations we came up with a couple of ways to know whats happening 
in productions.

### setproctitle
[Setproctitle](http://pypi.python.org/pypi/setproctitle) is a python library 
which allows you to change the name of the process which running. Projects like
postgres, celery, gunicorn use this to show which process is master and which
of them are workers. But we used this to a new level to know which worker is
serving what request and whats the latency in a each request.

In your WSGI middleware set a timer on your request object and change the process
title to the request.path and in the process_response set the request's title
to idle and add the time spent in the request. 

    #!python
    # some custom middleware
    def process_request(self, request):
        ...
        request.start_time = time.time()
        setproctitle.setproctitle("gunicorn -> "+request.path)
        ...

    def process_response(self, request, response):
        ...
        setproctitle.setproctitle("gunicorn -> [idle] [%0.3fs]" \
                            %(time.time()-request.start))

Now **ps -aux | grep gunicorn** will show you the current request being handled
and the time taken by a worker to process the last request. With this you can know
which workers are like stuck. The next method will tell you where there are stuck.


### Register SIGUSR1 signal
Another way to get a traceback of the python process is to register a SIGUSR1
signal to dump a traceback of the current execution in a file. All you need to
do is to call _register\_signals_ in any python script.

    #!python
    import os, signal, sys, traceback

    def sigusr1_handler(signum, frame):
        print "Received SIGUSR1 -- Printing stack trace..."

        f = open('/tmp/current.traceback.txt', 'a')
        traceback.print_stack(file=f)
        f.close()

    def register_signals():
        signal.signal(signal.SIGUSR1, sigusr1_handler)

Now call register_signals in settings.py or views.py or any python script. Now
do a ps -aux and get the pid of the process you want to know about and send a 
SIGUSR1 signal to it.

    ps -aux
    kill -SIGUSR1 8932
    cat /tmp/current.traceback.txt

This will give you a full traceback of where the process is stuck there by
giving you a clean idea of why its stuck.





