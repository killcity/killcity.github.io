---
comments:true
---
So you'd like to run a single collector on your dockerhosts. You want to send to that instance from each container running on the host. I spent a a couple hours trying to make this happen using environment variables and eventually gave up. I'm also using Macvlan. The same should apply for Ipvlan.

Here's what I did.

## Launch your statsd/dogstatsd container
Spin up statds, dogstats, whatever collector du jour, on your dockerhost. Run it with `--network="host"`. Make sure you also publish the ports its listening on (8125).

## Give /etc/hosts some love
Add an entry to `/etc/hosts` on your dockerhost.
```
mydockerhost# echo "`hostname -i`     dockerhost" >> /etc/hosts
```

## Mount it
When you launch your container, make sure you set a bind mount for your host's /etc/hosts to mount to your container's /host/etc/hosts. Obviously you can mount to whatever directory you want, but I just to toss everything in /host.
```
mydockerhost# docker service create --name myapp --mount type=bind,src=/etc/hosts,dst=/host/etc/hosts:ro myapp/myapp
```

## Smarten up your container's hosts file
In the entrypoint script for your container, toss in some logic that greps `/host/etc/hosts` for "dockerhost" and echos that one line, appending it to your container's `/etc/hosts` file.
```
mydockerhost# grep dockerhost /host/etc/hosts >> /etc/hosts
```

## Party
Make it rain. Now you can point your statsd client to `dockerhost`.

{% if page.comments %}
<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
this.page.url = PAGE_URL;  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = PAGE_IDENTIFIER; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://killcity.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>

{% endif %}
