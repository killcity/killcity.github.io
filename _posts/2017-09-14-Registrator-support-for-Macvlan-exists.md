---
comments: true
tags:
  - docker
  - swarm
  - macvlan
  - networking
  - consul
  - registrator
  - gliderlabs
---

Running Consul inside my containers, along with the apps was really getting to me. I would constantly be checking the <a href="https://github.com/gliderlabs/registrator">Registrator</a> Slack channels and forum postings...to no avail.

However, a day or two ago, I was sifting through the many PRs that are queued and noticed <a href="https://github.com/gliderlabs/registrator/pull/268">one</a> that had been merged to master. The includes a small chunk of code that allows ```-internal``` to actually work with Macvlan. The one caveat, is that you now have to ```EXPOSE``` your ports in your Dockerfiles. It's been working great. No more bundled Consul!

The entire time, this would have worked if I had used the ```gliderlabs/registrator:master``` image and NOT the ```gliderlabs/registrator:latest``` image. They are NOT the same.

Good luck!

{% if page.comments %}
<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
var disqus_identifier = "{{ page.url }}";
var disqus_url = '{{ site.url }}{{ page.url }}';
this.page.url = disqus_url;
this.page.identifier = disqus_identifier;
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
