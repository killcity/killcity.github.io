---
comments: true
disqus_identifier: 393493
---
I'm always on the pursuit for decent podcasts. Somehow, I stumbled onto <a href="http://devopscafe.org/"> The Devops Cafe Podcast</a>, which is hosted by John Willis from Docker and Damon Edwards from Rundeck. They have some really great interviews with the likes of Kelsey Hightower and James Turnbull. Great stuff!

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
