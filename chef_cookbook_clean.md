## part one

In a Chef development group with a high rate of cookbook churn, you may eventually find that your Chef server is timing out as the dependency solver (depsolver) works out the correct cookbook version to send to clients based on environment constraints and cookbook constraints. This gets ugly pretty quickly, since the Chef workers tied up doing depsolving aren't available for servicing other clients. At $WORK, we'd see lots of failed chef client runs, and usually at the most inconvenient of times. 

How did things go wrong? Well, we'd have a number of internal cookbooks with `metadata.rb` include constraints such as:

    depends 'java', '< 1.14.0'
    depends 'apt', '>= 1.8.2'
    depends 'yum', '>= 3.0'
    depends 'python'
    depends 'runit', '>= 1.5.0'
    depends 'bar', '~> 1.1'
    depends 'baz', '~> 2.0'


And when `bar` has versions 1.1.0, 1.1.1, 1.1.2, and `baz` has all its versions, and the upstream cookbooks have all their version iterations, all with their own constraints, the depsolver problem space grows exponentially. Eventually, the chef server will kill the long-running depsolvers, where long-running means about five seconds.

## Buying yourself time with longer depsolver timeout.

In a pinch, you can throw more resources at the problem, increasing the timeouts and the threads available to the chef server. This is only a short-term stop-gap. As you update more cookbooks, you'll soon be back to your earlier pain.

Good luck finding where to change those settings, as the `omnibus-chef-server` project (https://github.com/opscode/omnibus-chef-server) has an attribute defined for `default['chef_server']['erchef']['depsolver_timeout']`, but that attibute isn't used anywhere \*.

What you need to do is edit `/var/opt/chef-server/erchef/etc/app.config` to change the `depsolver_timeout` (under the `chef_objects` key) and the `max_count` under the `pooler` key, as show in this diff where the timeout goes to 10,000 ms, and the worker count is bumped to 12:

    --- app.config	2014-07-23 20:57:06.714838003 +0000
    +++ app.config~	2014-07-23 21:24:09.674838002 +0000
    @@ -114,7 +114,8 @@
                       {s3_external_url, host_header},
                       {s3_url_ttl, 900},
                       {s3_parallel_ops_timeout, 5000},
    -                  {s3_parallel_ops_fanout, 20}
    +                  {s3_parallel_ops_fanout, 20},
    +                  {depsolver_timeout, 10000}
                      ]},
       {stats_hero, [
                    {udp_socket_pool_size, 1 },
    @@ -148,7 +149,7 @@
                          {init_count, 20},
                          {start_mfa, {sqerl_client, start_link, []}} ],
                         [{name, chef_depsolver},
    -                     {max_count, 5},
    +                     {max_count, 12},
                          {init_count, 5},
                          {start_mfa, {chef_depsolver_worker, start_link, []}}]
                      ]},


Then restart the chef-server.

In the next part, I'll cover how to fix this with a safe cookbook clean-up.

\* I submitted a fix to the missing attribute problem with this pull request: https://github.com/opscode/omnibus-chef-server/pull/79

## Part Two: `knife cleanup versions`

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




## part three

How to apply constraints in depsolver-friendlier way


Why does Chef not cache the depsolver results?
See also: http://railsware.com/blog/2013/02/21/chef-dos-and-donts/ (or not)
http://stackoverflow.com/questions/20717804/chef-versioning-is-there-an-order-of-precedence


This commit:
https://github.com/opscode/chef_objects/commit/a3133ced037d1e508ff18723ad9a6f2b94dea1ea

removes depsolver and adds depselector, git => 'git://github.com/opscode/dep-selector'

 	+ timeout_ms = data[:timeout_ms]
  	78 	+ selector = DepSelector::Selector.new(graph, (timeout_ms / 1000.0))
  	79 	+
  	80 	+ answer = begin
  	81 	+ solution = selector.find_solution(run_list, all_versions)

removes depsolver.git and adds pooler.git


* Why does the depsolver not cache results?
  * The inputs are ....
* Confused by the _right way_ to do version numbering?  All questions answered at http://semver.orb
* Didn't Opscode/GetChef fix these depsolver timeout issues in 11.X? 
  * I don't think so.  11.0 introduced an Erlang-based depsolver, then that was rolled back in 11.TKTK
* What errors are indicative of the depsolver exhaustion issue?

* How does one keep track of all the dependency matching symbols?
  * I go to the source and refer to the chef [version_constraint unit tests at https://github.com/opscode/chef/blob/master/spec/unit/version_constraint_spec.rb](https://github.com/opscode/chef/blob/master/spec/unit/version_constraint_spec.rb)