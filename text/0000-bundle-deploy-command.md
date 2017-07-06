- Start Date: 2017-07-06
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

This RFC proposes a new `bundle deploy` command. The idea behind it is to wrap up all of the common flags, behaviors, conventions, and best practices around using Bundler to deploy a Bundle to a "production-ish" server. I think it should consist of (more or less) `check || install --deployment`, as a single command.

# Motivation

I think we should do this for two reasons:

1. The `check` command is incredibly fast compared to a no-op `install` command. Automatically speeding up every no-op deploy in existence by several seconds would be pretty awesome. Doing it in the same process, so we don't even have to boot two ruby intepreters, or eval the Gemfile twice, would be even better.
2. Starting with Bundler 2.0, the `--deployment` option will not be remembered. As a result, we've been suggesting that users will want to run `bundle config deployment true`, followed by `bundle install`. I don't like (at all) having to tell users that they need to run a command that creates local state before running the actual install command anytime they want deployment mode.

My hoped for/expected outcome is that automated tools switch from running `bundle check || bundle install --deployment` to running `bundle deploy` instead.

# Detailed design

Add a new command, `deploy`, designed to provide the functionality of running `bundle check || bundle install --deployment` but without having to start two Ruby processes. This means running the `check` logic inside the command, and if the check fails, moving on to run the `install` logic. Just like when `install --deployment` is run, the install would include the deployment-only check for Gemfile changes.

The `deploy` command would likely also need to accept the same set of flags that the `install` command accepts. 

The new man page for `deploy` would document the specific functionality provided by deployment mode, including both the fast check and raising on Gemfile/lockfile mismatch. The current docs for `install --deployment` would be changed to simply refer to the `deploy` help.

# How We Teach This

I think it makes more sense to have a clear delineation between "installing for development" and "installing for deployment" by separating the commands, rather than making the `install` command serve both functions.

We can pitch it as a nicer alternative to `install --deployment` in the release announcement, and update the suggested workflow story on the website to explain that `bundle deploy` is the command for deploying to servers, and is run automatically by cloud deployment systems.

Ultimately, I think the impact on most devs will be small, because deployment processes generally so fully automated already that it will mean updating a deployment dependency or changing a configuration file one time.

# Drawbacks

It would be work to change, and the payoff might not be worth the effort.

The documentation will need to be updated, but it may be possible to simplify it because of this change.

Existing apps would need to change their deployment setup to take advantage of it.

We will likely need to maintain both `install --deployment` and `deploy` in parallel for at least the 2.x series. Speeding adoption of this change would probably also require reaching out to @kirs and @scheems with PRs for capistrano-bundler and the heroku ruby buildpack, respectively.


# Alternatives

We could stick with what we have now, `bundle install --deployment`, and tell everyone to run that instead. We can also probably integrate the logic of `check` into the `install` command for no-op cases, providing that speedup without needing a new `deploy` command.

Another option is to perhaps have a `deploy-check` command instead, that raises if the lock doesn't match, and nothing else. Then we could remove deployment mode entirely, and tell people to run two stateless commands instead of two stateful commands.

I think the ultimate impact of not doing this is not huge, but it is annoyance by a handful of papercuts every time anyone uses Bundler to deploy.

# Unresolved questions

It's unclear whether the `deploy` command would also set the config `deployment` to true.

We might even want to remove the stateful config entirely, and only allow "frozen" behavior during the `deploy` command. That would streamline a lot of things, I think.