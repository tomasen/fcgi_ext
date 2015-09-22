### Migrating from php(LEMP) to Golang in large scale

[![Build Status](https://travis-ci.org/tomasen/fcgi_ext.svg?branch=master)](https://travis-ci.org/tomasen/fcgi_ext)
[![GoDoc](https://godoc.org/github.com/tomasen/fcgi_ext?status.svg)](http://godoc.org/github.com/tomasen/fcgi_ext)

In modern age, many sites are built on what we call - [LEMP](http://en.wikipedia.org/wiki/LAMP_\(software_bundle\)) solution. Which typically include nginx + php + mysql(or NoSQL databases) now.   
For system administrators of the web site with large traffic, should know the struggles of fighting with concurrency and availability of php. Facebook developed a monster project as [HipHop](https://developers.facebook.com/blog/post/2010/02/02/hiphop-for-php--move-fast/) just for such reason!    
As a Golang fan, even in my point of view, even for smaller scale websites or web applications, rewriting the website in Golang still not seems to be a reasonable option. Because most of them already have a huge PHP codebase and way too complicated frameworks. Which is an absolute deal-breaker.   
But the concurrency and efficiency of Golang is so attempting that I hereby propose a solution of migrating method, from php to Golang, just one step a time.


#### The advantage of Golang
* High concurrency
* Fast performance and memory efficiency
* Easy to learn


#### Problem that we are facing

* the max concurrent requests that php can handle is limited to the maximum number of php process can be running at the same time in the system. And php process is resource consuming.
* when there is bottleneck happened in the backend, for example: unstable connection to database or slow query, large amount of traffic will jammed in front of web server because php don't have the resource to process next requests anymore.
* Golang still lack of many library support, which are already used in php massively.

###Solution

####Basic principle
Only use Golang to replace part of php application process that Golang is good at handling(long polling and concurrency etc.).    
Only use PHP to process data instead i/o operation that may require waiting, like slow query or connect to remote service etc.

####Steps
* Put a Golang service between web server and php, connect them by fast-cgi.   
  + use Golang package [fcgi\_ext](https://bitbucket.org/PinIdea/fcgi_ext) and [fcgi\_client](https://bitbucket.org/PinIdea/fcgi_client) to build a [daemon](https://bitbucket.org/PinIdea/zero-downtime-daemon)
  + typical flow will be:    
  nginx <-> fcgi\_ext+fcgi\_client(golang) <-> php\_fcgi
* Use Golang to do the heavy lifting. And use the php code that we current have, to take care the rest.

####Example

```go
    package main

    import (
     "net/http"
     "strings"
     "log"
     "flag"
     "net"
     "bitbucket.org/PinIdea/fcgi_ext"
     "bitbucket.org/PinIdea/fcgi_client"
     "bitbucket.org/PinIdea/zero-downtime-daemon"
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
* Can execute step by step, not even need to be module by module. Just few lines here and there, you will get tremendous improve if plan it right
* Don't have the pressure or the risk of rewriting the whole system or application may bring.

###Practice
Here we have an old system design include nginx and search.php to process search request to term that can be sent to a solr(search engine) service. When solr response with the search results, search.php will render the results to more beautiful html language and output to nginx. We notice php process will jammed the system if solr become unstable during search traffic increase.
  
For a start, we write a Golang daemon that use fcgi\_ext to handle fcgi request from nginx. The Golang daemon will be in charge of communication with nginx and solr. Only when the Golang daemon received the results from solr, it initial another fcgi request to php\_fcgi(post\_search.php) to process data and render the web page. Then post_search.php should output html back to the Golang daemon, and the Golang daemon flush the output to nginx.

The result is promising, of cause. Golang is so good of handling concurrent long polling request, therefore you will see significant decrease of system resource usage. And for the best part, again, you don't have to face the challenge of rewriting or converting all you php source code to Golang at once. The migration will be under control and very stable for production service.
