Running Consul inside my containers, along with the apps was really getting to me. I would constantly be checking the Registrator Slack channels and forum postings...to no avail.

However, a day or two ago, I was sifting through the many PRs that are queued and noticed <a href="https://github.com/gliderlabs/registrator/pull/268">one</a> that had been merged to master. The includes a small chunk of code that allows ```-internal``` to actually work with Macvlan. The one caveat, is that you now have to ```EXPOSE``` your ports in your Dockerfiles. It's been working great. No more bundled Consul!

The entire time, this would have worked if I had used the ```gliderlabs/registrator:master``` image and NOT the ```gliderlabs/registrator:latest``` image. They are NOT the same.

Good luck!
