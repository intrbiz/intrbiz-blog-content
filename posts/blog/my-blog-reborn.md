---
Author: Chris Ellis
Category: blog
Date: 2014-08-16
---
# My Blog, Reborn

This blog has been broken for a while and I've finally gotten around to fixing 
it.  However rather than just reviving the old blog, I thought it was time for a 
change.  Whilst it may look thhe same, under the hood its a total rewrite.

I didn't dislike the previous Wordpress incarnation, althought there were some 
nasty hacks in that.  But viven I've developed Balsa, a fast and lightweight Java 
web application framework, I figured I should use it for my own stuff.

So, this blog is now being served from a Balsa application, the content is 
written in Markdown and stored in Git.  It only took a day to knock up the 
application, Balsa has good support for rendering Markdown content.

Creating a post is now as simple as:

* Firing up Kate (a rather lovely text editor)
* Attempting to extract my toughts in a coherent manner (the hardest part for me)
* Finally the `git commit; git push origin`

Its refreshingly simple to add a post now, just write, commit and push.

I've even put the code behind my blog on [GitHub](https://github.com/intrbiz/intrbiz-blog) 
for people who are really interested.
 