##Migrating from php(LEMP) to Golang in large scale

In modern age, many sites are built on what we call - [LEMP](http://en.wikipedia.org/wiki/LAMP_\(software_bundle\)) solution. Which typically include nginx + php + mysql(or NoSQL databases) now.   
For system administrators of the web site with large traffic, should know the struggles of fighting with concurrency and availability of php. Facebook developed a monster project as [HipHop](https://developers.facebook.com/blog/post/2010/02/02/hiphop-for-php--move-fast/) just for such reason!    
As a Golang supporter, even in my point of view, even for smaller scale websites or web applications, rewriting the website in Golang still not seems to be a reasonable option. Because most of them already have a huge PHP codebase and way too complex frameworks. Which is an absolute deal-breaker.   
But the concurrency and efficiency of Golang is so attempting that I hereby propose a solution of migrating method, from php to Golang, just one step a time.

###The advantage of Golang
* High concurrency
* Fast performance and memory efficiency
* Easy to learn


###Problem that we are facing

* the max concurrent requests that php can handle is limited to the maximum number of php process can be running at the same time in the system. And php process is resource consuming.
* when there is bottleneck happened in the backend, for example: unstable connection to database or slow query, large amount of traffic will jammed in front of web server because php don't have the resource to process next requests anymore.
* Golang still lack of many library support, which are already used in php massively.

###Solution

####Basic principle
Only use Golang to replace part of php application process that Golang is good at handling(long polling and concurrency etc.).    
Only use PHP to process data instead i/o operation that may require waiting, like slow query or connect to remote service etc.

####Steps
* Put a Golang service between web server and php.   
  + use Golang package [fcgi\_ext](https://bitbucket.org/PinIdea/fcgi_ext) and [fcgi\_client](https://bitbucket.org/PinIdea/fcgi_client) to build a [daemon](https://bitbucket.org/PinIdea/zero-downtime-daemon)
  + typical flow will be:    
  nginx <-> fcgi\_ext+fcgi\_client(golang) <-> php\_fcgi
* Use Golang to do the heavy lifting. And use the php code that we current have, to take care the rest.

####The advantages of this solution
* Very easy to get started, less than one hundred lines of Golang codes is enough to get the migration up and running
* Can execute step by step, not even need to be module by module. Just few lines here and there, you will get tremendous improve if plan it right
* Don't have the pressure or the risk of rewriting the whole system or application may bring.

###Practice
Here we have an old system design include: Nginx, search.php to process the search request to programed  term and send them to a solr(search engine) service. When solr response the search results, search.php will also query database. After that, php render the results to more beautiful html language and output to nginx.    
We notice php process will jammed the system if solr become unstable when search traffic increased.  
  
For a start, we write a Golang daemon that use fcgi\_ext to handle fcgi request from nginx. The Golang daemon in charge of communication with solr. Only when the Golang daemon received the result, it send fcgi request to php\_fcgi(post\_search.php) to process data and render. post_search.php will output html to the Golang daemon, then Golang daemon flush the output to nginx.

The result is promising, of cause. Golang is so good of handling concurrent long polling request, therefore you will see significant decrease of system resource usage. And for the best part, again, you don't have to face the challenge of rewriting or converting all you php source code to Golang at once. The migration will be under control and very stable for production service.