- Start Date: 04/03/2017
- RFC PR: 
- Bundler Issue: 

#### Note: This was taken from @indirect at [bundler/bundler#4925](https://github.com/bundler/bundler/issues/4925)

# Summary

I'm proposing that we create a report-issue command to provide more help for end-users who are having problems, and completely automating the information that we have to ask for when investigating and diagnosing issues.

The command can act as an automated instances of the [ISSUES troubleshooting guide], suggesting common fixes and then collecting all of the required information into a gist and posting a new issue.

# Motivation

Today we have [the exception reporting template] and [the `bundle env` command]. As evidenced by issues like [#4920](https://github.com/bundler/bundler/issues/4920) (among countless others) we usually don't get enough information. Repetitive task? Computers can do this for us. 😅

# Detailed design

Create a new command (with a name like `report-issue`, although it could be something else). 
- expand the data collected by `env`
  - collect the full path to `ruby`
  - collect `$LOAD_PATH` and `Gem.path`
  - collect the OS name and version (or linux distro name)
  - collect installed ruby packages for common package managers (brew, apt, yum, ???)
  - simplify user-facing exception reports
  - default to showing no backtrace
  - write the last backtrace into a file
  - print a generic error message like “Sorry, something unexpected happened, and Bundler was unable to continue.”
  -  suggest running `report-issue` if the problem continues
  - when `report-issue` is run, show a troubleshooting guide
  - run through the suggested troubleshooting steps
  - suggest updating bundler, suggest updating rubygems
  - maybe offer to run the commands automatically?
  - if none of that works, suggest running `report-issue --nothing-helped`
  - when `report-issue --nothing-helped` issue creator
  - collects the output from `bundle env`
  - creates an anonymous gist out of all of this stuff
  - tells the user that there are some required questions for an issue report, and then asks them one at a time:
    - summary of the problem (5-10 words)
    - what did you do?
    - what did you expect to happen?
    - what actually happened?
  - creates a new issue with the summary as the subject, the answers to the questions as the body, and a link to the gist containing all of the additional information
  
# Documentation changes
- add a man page for `env`
  - add a man page for `report-issue`
  - describe using the `report-issue` command directly in `README.md`
  - describe using `report-issue` on the bundler.io homepage
  
# How We Teach This

# Drawbacks

# Alternatives

It’s not the worst thing to stick with the status quo. We end up having to tell users the same things over and over, and I’m sure that in the end no amount of fancy commands is going to save us from that. This may take more work to implement than it provides benefit, which is most of why I’m writing this RFC.

# Unresolved questions

Is it a good idea to do this at all? Is it a good idea to automatically create a gist and open an issue? Should this entire idea be scrapped in favor of `bundle env --gist`?
