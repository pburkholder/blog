## Part Two: `knife cleanup versions`

In the [first part](http://www.pburkholder.com/post/93155061892/clearing-the-counter-cookbook-clutter-and-knife) I discussed a workaround for depsolver timeouts. The real fix is getting rid of unused cookbooks.

A colleague at $WORK discovered a plugin by [Marius Ducea](https://github.com/mdxp) which would remove unused cookbook versions from a chef server. The plugin provided you the option of dumping all cookbooks locally before deleting them. The problem we saw was that 'unused' meant 'not explicitly pinned' with equality; it disregarded `~>` and `>=`, etc. So we could not use the plugin without trashing many of our cookbooks in use.

I extended the plugin to take a `--runlist` parameter, so it would query the chef server for the cookbook versions needed to satisfy that runlist, and do so for each of the environments present on the server. At the time we were not using Chef's built-in roles, as some of us had mis-read/mis-understood work such as [The Berkshelf Way](http://www.slideshare.net/opscode/the-berkshelf-way-20882903) or [A Year with Chef](http://devopsanywhere.blogspot.com/2012/10/a-year-with-chef.html). We had one cookbook `work-roles`, which in turn had recipes such as `workroles::mongo` or `workroles::api`. To clean up cookbooks, I'd run:

    knife cleanup versions --runlist 'workroles::default'

and it would list all the cookbooks eligible for deletion. Then I'd run that again with the `-D` option to actually do the deletion. A few score cookbooks would _poof_ go away and our depsolver issues would be gone.

I have a [PR open](https://github.com/mdxp/knife-cleanup/pull/3) with Marius but I've not heard back since Christmas. 

Meanwhile, you can use it from [my GitHub runlist_feature branch](https://github.com/pburkholder/knife-cleanup/tree/runlist_feature). 

### What about roles?

Turns out that _one master roles cookbook_ is [an anti-pattern](https://tickets.opscode.com/browse/CHEF-4837). That's a story for another time. Chef roles are great if they're kept lightweight; they were unfairly maligned by abuses heaped upon them. 

My branch of `knife-cleanup` doesn't handle roles well, since it expects a runlist of recipes, so I'll need to address that. 

Unless cookbook clean up has been built into Chef via some other avenue, I'll fork the current `knife-cleanup` plugin into a `knife-scrub` plugin which will build up a runlist for all roles, and keep the versions in use in any environment. Or if Marius has time then we can work on this issue together.

I have a short [Part Three]() forthcoming with some other musings around depsolver and cookbook cleanup. Stay tuned.

*Notes*:

* Marius Ducea's [blog post on knife cleanup](http://www.ducea.com/2013/02/26/knife-cleanup/)
* Useful, to me, [Chef api reference](http://docs.getchef.com/api_chef_server.html#roles-name-environments-name)
* One of the posts lashing out at [Chef roles](http://devopsanywhere.blogspot.com/2012/10/a-year-with-chef.html)
* Jira ticket [Role Cookbooks are a Chef AntiPattern](https://tickets.opscode.com/browse/CHEF-4837)