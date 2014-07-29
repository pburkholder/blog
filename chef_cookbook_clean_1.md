## Part One

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

    --- app.config  2014-07-23 20:57:06.714838003 +0000
    +++ app.config~ 2014-07-23 21:24:09.674838002 +0000
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

In the [next part](http://www.pburkholder.com/post/93221514172/clearing-the-counter-pt-ii-knife-cleanup-tweaks-chef), I'll cover how to fix this with a safe cookbook clean-up.

\* I submitted a fix to the missing attribute problem with this pull request: https://github.com/opscode/omnibus-chef-server/pull/79
