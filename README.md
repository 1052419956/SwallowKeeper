# SwallowKeeper
###
Easemob/SwallowKeeper is a simple and handy autoscale solution for micro service, it is composed of nginx (tengine), dyups and consul, it updates the upstream list by watching consul services, once any serivce changes found or timeouts, it will persistent the updated upstreams first in case of consul server crash and then update upstreams into memory, which will take effect in near real time without reloading nginx. SwallowKeeper can reduce much operation time by automating removing/adding upstream servers without reloading nginx when deploying app or restarting app for a configuration change. In our customer service project, all springboot micro services apply this solution, each springboot app auto registers itself into consul when starting, and deregisters itself when stopping. Honestly, this make our operation team's life much more easier.

This solution can apply to not just java language, also apply to python, ruby, php etc. The consul role in this solution can be replaced with etcd/zookeeper as well, this will be repleased in future.
###
## Advantage

 * __Users will not be aware of any breaks during upstream service restarts or service crash.__
 Without this solution, if any service crashes  all of a sudden, nginx will only be aware of this after retry_times * check interval, users will get 502 response durint this period. With consul watch mechanism, nginx can notice the service status change in near real time, and users will not aware of the service break.

* __This solution is not intrusive for application codes.__
  Application codes doesn't need to change, what is needed is only few nginx dyups module configs and a script for watching consul changes.

* __Nginx dyups module provides restful api to manage upstream servers, it's easy to implement our own tool to manage upsstream servers.__

## Architecture in our environment

  ![Kefu autoscale structure](https://github.com/easemob/SwallowKeeper/blob/master/images/dyups_consul_app.png)
  
  
## Components
 * __tengine with dyups module__
 
   We built tengine 2.1.2 with dyups, and it has been running ok in production for 1 year.
 * __consul cluster__
 
    __Consul Client__: We deploy consul client on each host, all micro services on that host will register their information into local consul agent (including service information, health check etc). Then Consul agent sync these information to consul servers.
    
   __Consul Server__: It's recommended to use 3 or 5 nodes to form the consul server cluster to guarantee high avalibility. Clients can get all registered services information via consul servers.
    
    
 * __update_nginx_upstream.py__
 
   One script reads upstreams information from consul server and updates them into tengine memory with dyups api, and this will take effect in near real time without reloading tengine.

## Install and Configure
 
 It's assumed consul cluster has been setup successfully in your environment and services are already registered.

  ```
   1. Build tengine with dyups and install it
   
   2. configure dyups configs in tengine
      
       server {
        listen  127.0.0.1:18882;
        location / {
            dyups_interface; # Define the dyups api interface
        }
       }
      
      Reference: https://github.com/yzprofile/ngx_http_dyups_module
 
   3. Install consul agent on tengine server so that script update_nginx_upstream.py can fetch all services information registered in consul via it.
   
   4. Change variables in scripts/update_nginx_upstream.py 
   
      eg:
         # NGINX_DYUPS_ADDR: Define dyups management url, it's configured on the same host with nginx
         NGINX_DYUPS_ADDR = "http://127.0.0.1:18882"
         
         #UPSTREAM_FILE: Define upstream config file to persist servers information from consul server, this config file will be updated
         # automatically once any service status changes or after LONG_POLLING_INTERVAL time
        UPSTREAM_FILE = "/home/dyups/apps/config/nginx/conf.d/dyups.upstream.com.conf"
        
        
  5. Run script update_nginx_upstream.py with supervisor
 
  ```
