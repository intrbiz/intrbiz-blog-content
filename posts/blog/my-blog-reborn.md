---
Author: Chris Ellis
Category: blog
Date: 2014-08-16
---
# My Blog, Reborn

This blog has been broken for a bit too long and I've finally gotten around to 
fixing it.  However rather than restoring the Wordpress VM which used to host 
this site.  Whilst it may look the same, under the hood its a total rewrite.

I didn't dislike the previous Wordpress incarnation, althought there were some 
nasty hacks in that.  Given I've developed Balsa, a fast and lightweight Java 
web application framework, I figured I should use it.

This blog is now being served from a Balsa application, the content is kept in 
Markdown and stored in Git.  Creating a post is now as simple as: firing up Kate 
(a very lovely text editor), attempting to extract my toughts in a coherent 
manner and the `git commit; git push origin`.  Its refreshingly simple to add a 
post now.

I've even put the code behind my blog on 
[GitHub](https://github.com/intrbiz/intrbiz-blog) for people who are really 
interested.

 