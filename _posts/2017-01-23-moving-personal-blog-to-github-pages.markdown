---
layout: post
title:  "Moving Personal Blog to Github Pages"
date:   2017-01-23 19:59:03 -0800
---
Previously I used Github as poor man's blog site. After a while, I wanted to have a real blog with comment support. Thus, I turned to Github pages for help.
Github pages come to rescue. It has good support for Jekyll, a blog building ruby service.
After some investigation, I decided to move to Github pages.

## Why Github Pages?
1. Reliable hosting - Github pages are hosted by Github. The blog comes directly from your Github repo.
2. Sharing with community - I can put the whole blog in the Github repo. Coders love sharing!
3. Easy to use - The blogger repo is just another normal Github repo. You can add, commit, push and done!
4. Close integration with Jekyll, the Blog Generator.

## Migration In Simple Steps
1. Create a repo "username,github.io" in your github account. For simplicity, let's assume username is "abc".
2. On your local macine, run 

```shell
git clone git@github.com:abc/abc.github.io.git
```

3. Assume you alreayd have Ruby installed, now install Jekyll and bundler in your local machine.

```shell
sudo gem install jekyll bundler
```

4. Enter your github.io directory, and generate the skeleton for your blog under your github.io directory.

```shell
cd abc.github.io && jekyll new .
```

5. Now start serving the blog! You will see a "Welcome" page. Check the server address to show the log locally. Usually the address is "http://127.0.0.1:4000/
".

```shell
 bundle exec jekyll serve
```

6. It is time for the migration - you can copy the old markdown blogs under the "_post" directory, Name your markdown as "YYYY-MM-DD-blog-name.md" so that Jekyll can understand it.

## Adding Images and Comments
We have a basic blog running, but it is not good enough - we want to embed images in our blog and let readers comment on it. To set up the images, you can first put the images under "_assets" directory. Then, in your blog, you can refer the image as "{{ site.url }}/assets/captcha.gif".

You also want to set up comments for your blog. Fortunately, Disqus can support the [user comments](http://stackoverflow.com/a/22201969) easily. All you need to do are:
1. Create an account in Disqus.
2. Create a _layout_directory, and add a post.html inside that directory.

```html
---
layout: default
---

<h2 class="post_title">
  {{ page.title }}
</h2>

{{ content }}

{% include disqus.html %}
```

3. Create disqus.html inside _includes directory_

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

You can find all the examples in [my blog](https://wang-ye.github.io/).

## What's Next?
I am thinking about adding static comments and host the comments in github pages too. Fortunately, there have been [articles](https://github.com/mpalmer/jekyll-static-comments) about it.
