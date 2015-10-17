### Migrating from php(LEMP) to Golang on a large scale

[![Build Status](https://travis-ci.org/tomasen/fcgi_ext.svg?branch=master)](https://travis-ci.org/tomasen/fcgi_ext)
[![GoDoc](https://godoc.org/github.com/tomasen/fcgi_ext?status.svg)](http://godoc.org/github.com/tomasen/fcgi_ext)

In the modern age, many sites are built on what we call - [LEMP](http://en.wikipedia.org/wiki/LAMP_\(software_bundle\)) stack. Which typically include nginx + php + mysql(or NoSQL databases).   
For system administrators of websites with large amount of traffic, should know the struggles of fighting with concurrency and availability of php is well known. Facebook developed a monster project known as [HipHop](https://developers.facebook.com/blog/post/2010/02/02/hiphop-for-php--move-fast/) just for such reason!    
Even as a Golang fan,  rewriting a website in Golang still does not seems to be a reasonable option even if they are small scale websites or web applications. Because most of them already have a huge PHP codebase and way too complicated framework. Which is an absolute deal-breaker.   
But the concurrency and efficiency of Golang is so tempting that I hereby propose a solution of method of migrating method, from php to Golang, which requires just one step a time.


#### The advantage of Golang
* High concurrency
* Faster performance and better memory efficiency
* Easy to learn and maintainance


#### The problem that we face

* the max concurrent requests that php can handle is limited to the maximum number of php process can be running at the same time on a single server. And the php process is resource consuming.
* when there is a bottleneck happening in the backend, for example: unstable connection to the database or slow queries, the large amount of traffic will jam front web server because php doesn't have the resources to process the next request anymore.
* Golang still lack many library, which are already used in php massively.

###The Solution

####The Basic principles

Only use Golang to replace the parts of php application process that Golang is good at handling(long polling and concurrency etc.).    
Only use PHP to process data instead of i/o operations that may require waiting, like slow queries or connecting to remote service etc.

####Steps
* put a Golang service handler between web server and php, and connect them by fast-cgi.   
  + use Golang package [fcgi\_ext](https://github.com/tomasen/fcgi_ext) and [fcgi\_client](https://github.com/tomasen/fcgi_client) to build a [daemon](https://github.com/tomasen/zero-downtime-daemon)
  + the typical flow will be:    
  nginx <-> fcgi\_ext+fcgi\_client(golang) <-> php\_fcgi
* Use Golang to do the heavy lifting. And use the php code that we currently have, to take care the rest.

####Example

```go
    package main

    import (
     "net/http"
     "strings"
     "log"
     "flag"
     "net"
     "github.com/tomasen/fcgi_ext"
     "github.com/tomasen/fcgi_client"
     "github.com/tomasen/zero-downtime-daemon"
    )

    const (
      socket_file = "/tmp/golang-fcgi.sock"
      socket_php  = "/tmp/php-fpm.sock"
    )

    type FastCGIServer struct{}

    var (
      optCommand  = flag.String("s","","send signal to a master process: stop, quit, reopen, reload")
      optConfPath = flag.String("c","","set configuration file" )
      optHelp     = flag.Bool("h",false,"this help")
    )

    func usage() {
      log.Println("[command] -conf=[config file]")
      flag.PrintDefaults()
    }


    func doFcgi(resp http.ResponseWriter, fcgi_params map[string]string, sock string, phpfile string) (content []byte) {

      fcgi_params["SCRIPT_FILENAME"] = fcgi_params["DOCUMENT_ROOT"] + phpfile
  
      fcgi, err := fcgiclient.New("unix", sock)
      if err != nil {
        log.Printf("err: %v\n", err)
        return
      }

      content, err = fcgi.Request(resp, fcgi_params, "")
      if err != nil {
        log.Printf("err: %v\n", err)
        return
      }

      fcgi.Close()
      return 
    }

    func (s FastCGIServer) ServeFCGI(resp http.ResponseWriter, req *http.Request, fcgi_params map[string]string) {
  
      req.ParseForm();
  
      switch{
        case strings.HasPrefix(strings.ToLower(req.RequestURI), "/search"):
      
          // call php to filter query: parse keyword
          keyword := doFcgi(nil, fcgi_params, socket_php, "/search2/search_pre.php")
      
          // do something golang is good at, eg. communicate with database, slow query, other remote backend
          // ...
      
      
          // call php to prepare output: render result
          // prepare some custom argument to php
          fcgi_params["GOARG_SEARCHWORD"] = string(keyword)
          doFcgi(resp, fcgi_params, socket_php, "/search2/search_post.php")
      
      }

    }

    func handleListners(cl chan net.Listener) {
      for v := range cl {
        go func(l net.Listener) {
          srv := new(FastCGIServer)
          fcgi.Serve(v, srv)
        }(v)
      }
    }

    func main() {
  
      // parse arguments
      flag.Parse()
  
      if (*optHelp) {
        usage()
        return
      }

      ctx  := gozd.Context{
        Hash:   "go-fcgi",
        Command:*optCommand,
       // Maxfds: 32767,
        User:   "www",
        Group:  "www",
        Logfile:"/var/log/go_daemon.log", 
        Directives:map[string]gozd.Server{
          "sock":gozd.Server{
            Network:"unix",
            Address:socket_file,
          },
        },
      }
  
      cl := make(chan net.Listener,1)
      go handleListners(cl)
      done, err := gozd.Daemonize(ctx, cl) // returns channel that connects with daemon
      if err != nil {
        log.Println("err: ", err)
      }
  
      if <- done {
        // wait current connect closed and exit

      }
    }
```

####The advantages of this solution
* Very easy to get started, less than one hundred lines of Golang codes is enough to get the migration up and running
* Can execute step by step, not even need to be module by module. Just a few lines here and there, you will get tremendous improvement if planned out right
* Don't have the pressure or the risk of rewriting the whole system or bringing the whole application.

###Practice
Here we have an old system design including nginx and search.php to process search request terms that will be sent to a solr(search engine) service. When solr responds with the search results, search.php will render the results to more higher level languag of html and output to nginx. We notice php process will jam up the system if solr become unstable during periods of high traffic.
  
For a start, we write a Golang daemon that uses fcgi\_ext to handle fcgi request from nginx. The Golang daemon will be in charge of all communication between nginx and solr. Only when the Golang daemons received the results from solr, will it initial another fcgi request to php\_fcgi(post\_search.php) to process the data and render the web page. Then post_search.php should then output html back to the Golang daemon, and the Golang daemon will flush the output to nginx.

The result is promising, of course. Golang is so good at handling concurrent long polling requests, therefore you will see significant decrease of system resource usage. And for the best part, again, you don't have to face the challenge of rewriting or converting all you php source code to Golang at once. The migration will be under control and very stable production service.
