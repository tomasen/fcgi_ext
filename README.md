##Migrating from php(LEMP) to Golang in large scale

In modern age, many sites are built on what we call - [LEMP](http://en.wikipedia.org/wiki/LAMP_\(software_bundle\)) solution. Which typically include nginx + php + mysql(or NoSQL databases) now.   
For system administrators of the web site with large traffic, should know the struggles of fighting with concurrency and availability of php. Facebook even developed a monster project as [HipHop](https://developers.facebook.com/blog/post/2010/02/02/hiphop-for-php--move-fast/)!
Even for websites or web applications much smaller than the Facebook, the php source code amount still too huge and too complex to rewrite to Golang.
But the concurrency and efficiency of Golang is so attempting that I hereby propose a solution to migrating from php to Golang one step a time.

###The advantage of Golang
* High concurrency
* Fast performance and memory efficiency
* Easy to learn


###Problem that we are facing under php

* the max concurrent requests that php can handle is limited to the maximum number of php process can be running same time in the system.
* when there is bottleneck happened in the backend, for example: unstable connection to database of slow query, large amount of traffic will jammed in front of web server because php don't have the resource to process more requests.
* Golang still lack of many library support with massive used in php.

###Solution

####Basic principle
Only use Golang to replace part of php application process which Golang is good at handling(long polling and concurrency). eg.   
Only use PHP to process data instead i/o waiting operation like slow query or connect to remote service.

####Steps
* Put a Golang service between web server and php.   
  + use Golang package like [fcgi_ext](https://bitbucket.org/PinIdea/fcgi_ext) and [fcgi_client](https://bitbucket.org/PinIdea/fcgi_client) to build a [daemon](https://bitbucket.org/PinIdea/zero-downtime-daemon)
  + typical flow will be:    
  nginx+fcgi_ext(golang)+fcgi_client(golang)+php_fcgi

* Use Golang to do the heavy lifting, and still use the php code that we current have, to take care the rest.

####The advantages of this solution
* Very easy to get started, less than one hundred lines of Golang codes is enough to get the migration up and running
* Can execute step by step, not even need to be module by module. Just few lines here and there you will get huge improve if the plan is right
* Don't have the pressure to rewrite the whole system or application

###Practice
Here we have an old structure include: Nginx, search.php to process the search request to program understandable term and send to a solr(search engine). And when solr response the search results, search.php will also query database then render the results to more beautiful html language and output to nginx. We notice php process will jammed the system if solr become unstable when search traffic increased.    
For start, we write a Golang daemon use fcgi_ext to handle fcgi request from nginx. The Golang daemon in charge of communication with solr. After the Golang daemon received the result, it send fcgi request to php_fcgi(post_search.php) to process data and render the html. post_search.php will output to the Golang daemon, then Golang daemon output to nginx.   
The result is promising of cause, Golang is so good of handling concurrent long polling, you will see significant decrease of system resource usage. And for the best part, again, you don't have to face the challenge of rewriting or converting all you php source code to Golang in short period.