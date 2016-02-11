I found AWS OpsWorks when I was looking for a way to orchestrate the applications containers that I had just created. At that moment I was using shell script and puppet. 

Just after starting using OpsWorks, I realised it should be my key tool for containerisation because of its features, like auto-healing to EC2 instances, auto-scale, and time-based EC2 instances. Of course there are other solutions available out there, and I'm still trying them, and I have to say that I’m kind of tending to kubernetes or deis.

Anyway, I thought that I could use the custom json OpsWorks feature to translate the docker-compose.yml that our developers already use to tell the docker hosts which container images they should run and how.

OpsWorks is a tool that makes it easy to provision and manage hosts (EC2 instances) and integrate them with EC2 ELBs for instance. Basically, you create application stacks that have application layers (instances, load balancers, RDSs…) and configure them using custom chef cookbooks. You can also provide a "custom JSON" on stack creation time that will be passed to chef. 

The custom JSON is the key feature here, as it can be used to to configure, tag and set properties of EC2 instances that will run chef recipes.

Some of the things I did (with the contribution of [@tnache](https://bitbucket.org/tnache/opsworks-recipes)) were: 

>- Change the EC2 instances tags on boot time
>- Update a specific Route53 IN A entry based on the current public ip of an EC2 instance
>- Mount NFS exports

But the major goal of this article is to tell you how I did the docker
container orchestration with docker-compose.

Docker-compose uses yaml files to declare how containers should run. And
yaml files can be easily translated to json.

So, things like this:

```
db:
 image: postgres
web:
 build: my-djangoapp
 command: python manage.py runserver 0.0.0.0:8000
 volumes:
   - /data:/data
 ports:
   - "8000:8000"
 links:
   - db
```

When represented in json, should look like:

```
{
 "db": {
   "image": "postgres"
 },
 "web": {
   "image": "my-djangoapp",
   "command": "python manage.py runserver 0.0.0.0:8000",
   "volumes": [
     "/data:/data"
   ],
   "ports": [
     "8000:8000"
   ],
   "links": [
     "db"
   ]
 }
}
```

Easy....


So, the idea here is to have the YAML translated to JSON so we can use
it inside the custom JSON field on the OpsWorks Stack's creation,
allowing the json to be passed to chef.

Of course that docker-compose as I said uses yaml files, so we would
have to translate it from json back to yaml using chef, and it is pretty
easy using chef templates.

Because I would like to use the custom JSON to things other than simply
write the docker-compose.yml, I decided to put the yaml translated to
json inside a new json key. So the json would look like this:

```
{
   "applications":
   {
       "db": {
           "image": "postgres"
       },
       "web": {
           "image": "my-djangoapp",
           "command": "python manage.py runserver 0.0.0.0:8000",
           "volumes": [
               "/data:/data"
               ],
           "ports": [
               "8000:8000"
               ],
           "links": [
               "db"
               ]
       }
   }
}
```

This way, I can use specific keys to specific chef recipes, for
instance tags, route53, nfs... 

One last thing. As the OpsWorks Stacks supports 'Layers' of EC2
Instances, it would be nice if I could specify the applications json's
key to each layer.


```
{
   "layers": {
       "djangoapp":{
           "applications":
           {
               "db": {
                   "image": "postgres"
               },
                   "web": {
                       "image": "my-djangoapp",
                       "command": "python manage.py runserver 0.0.0.0:8000",
                       "volumes": [
                           "/data:/data"
                           ],
                       "ports": [
                           "8000:8000"
                           ],
                       "links": [
                           "db"
                           ]
                   }
           }
       }
   }
}
```

On the example above, the chef template file would translate to yaml to all the EC2
Instances inside the OpsWorks Stacks's layer called 'djangoapp' the json
starting on the 'applications' key.

We can "translate" one docker-compose.yml file to each OpsWorks Stack's
layers creating more keys in the "layers" json's level. And also, we can
create a "global" "applications" key which would be merged to the
specific layer's "applications". In that case the more long json would
be like this:

```
{
   "applications":{
       "postfix" {
           "image": "postfix"
       }
   },
       "layers": {
           "djangoapp":{
               "applications":
               {
                   "db": {
                       "image": "postgres"
                   },
                   "web": {
                       "image": "my-djangoapp",
                       "command": "python manage.py runserver 0.0.0.0:8000",
                       "volumes": [
                           "/data:/data"
                           ],
                       "ports": [
                           "8000:8000"
                           ],
                       "links": [
                           "db"
                           ]
                   }
               }
           }
       }
}
```

With the json above, every single EC2 Instance  on
the OpsWorks Stack would run the container postfix, and only the EC2 Instances on the layer called 'djangoapp' would run the postgres, and
the my-djangoapp images.

This "feature" also extends itself to the other recipes that I
mentioned (changing tags, route53 entries, nfs mounts...), so a really
complete custom JSON on the OpsWorks Stack's would look like this: 

```
{
   "tags"{
       "Product": "Django example"
   },
   "applications":{
       "postfix" {
           "image": "postfix"
       }
   },
   "layers": {
       "djangoapp":{
           "tags": {
               "Service": "the application"
           },
           "applications":
           {
               "db": {
                   "image": "postgres"
               },
               "web": {
                   "image": "my-djangoapp",
                   "command": "python manage.py runserver 0.0.0.0:8000",
                   "volumes": [
                       "/data:/data"
                       ],
                   "ports": [
                       "8000:8000"
                       ],
                   "links": [
                       "db"
                       ]
               }
           }
       }
   }
}
```

I will soon prepare a Youtube video describing step by step
the process to get this working for those that have no OpsWorks and Chef knowledge. 

Meanwhile, Thiago Nache is making available the docker-compose recipes that
we wrote on [his bitbucket](https://bitbucket.org/tnache/opsworks-recipes). You can also can find a working fork on [mine](https://bitbucket.org/fbueno/opsworks-recipes).
