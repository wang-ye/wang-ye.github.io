---
layout: post
title:  "Moving Personal Blog to Github Pages"
date:   2017-01-23 19:59:03 -0800
---
I used to have Github to host my blogs in markdown format. After some struggling, I really wanted to have a real blog with good theme, index page and user comment support. After some research I decided to move to [Github pages](https://pages.github.com/), and I am quite happy with it. This blog discusses the rationale of my transition and the detailed transition steps.

### Why Github Pages?
Github page offers many advantages compared to a vanilia github markdown blog.

1. Reliable hosting - Github pages are hosted by Github. The blog comes directly from your Github repo.
2. Sharing with community - I can put the whole blog in the Github repo. Coders love sharing!
3. Easy to use - The blogger repo is just another normal Github repo. You can add, commit, push and done!
4. Close integration with Jekyll, the ruby blog generator. This brings me theme, index page and some other goodies.

### Migration In Simple Steps
1. Create a repo "github_name,github.io" in your github account. For simplicity, let's assume your github user name is *abc*.
2. Clone your github.io repo to local envionment. On your local macine, run 
```
git clone git@github.com:abc/abc.github.io.git
```
3. Assuming you already have Ruby installed, now install Jekyll and bundler locally. Run
```
sudo gem install jekyll bundler
```
4. Enter your github.io directory, and generate the skeleton for your blog under your github.io directory. Run
```
cd abc.github.io && jekyll new .
```
5. Now start serving the blog locally!  Run
```
 bundle exec jekyll serve
```
, and type the server address to the browser.  Usually the address is http://127.0.0.1:4000/. You will be able to see blog index page.
6. It is time for the migration - you can copy your old markdown files under the "_post" directory. Name your markdown as "YYYY-MM-DD-blog-name.md" so that Jekyll can understand and serve it.

### Adding Images and Comments
Now we have a basic blog running, but it is not good enough - we want to embed images in our blog, also, we want to interact with readers through comments. 

To serve your image "image.gif" in your blog, you first put it under "_assets" directory. Then in your blog, just refer it as {% raw %} "{{ site.url }}/assets/image.gif" {% endraw %}.

As to comments, Disqus is a nice tool for supporting the [user comments](http://stackoverflow.com/a/22201969). All you need to do are:

1. Create an account in Disqus.
2. Create a _layout_directory, and add a [post.html](https://github.com/wang-ye/wang-ye.github.io/blob/master/_layouts/post.html) inside that directory.
3. Create [disqus.html](https://github.com/wang-ye/wang-ye.github.io/blob/master/_includes/disqus.html) inside _includes directory_

```html
<div id="disqus_thread"></div>

<script type="text/javascript">
    var disqus_shortname = 'star-dust';

    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>

<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>2
```

You can find the files in [my blog](https://wang-ye.github.io/).

### What's Next?
I am thinking about adding static comments and host the comments in Github pages too. Fortunately, there have been [articles](https://github.com/mpalmer/jekyll-static-comments) about it.
