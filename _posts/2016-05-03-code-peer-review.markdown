---
layout: post
title: "Team Peer Review"
date: 2016-05-03 08:00:00
categories: development peer-review
---

I have mentioned before in previous blog postings how much I _love_ testing.  I am fairly half serious
about that statement.  I do love what I achieve after code is tested.  The feeling of satisfaction
that only comes with validation that what you have created actually works the way you think it does.
Recently my team was exclaiming how great the quality of our code is because we have on average 92%
LOC coverage in our unit test suite.  With great timing our boss said, well that is great, you don't
know what 8% of your code is doing. :/

In my prior post about getting [external coverage][external-coverage] from behavior testing I went into
depth about how to get real numbers for LOC coverage from external testing suites, such as lettuce or 
cucumber or behave.

## Started from a discussion

I recently asked in a team meeting if we should set the threshold of coverage to 100%, as our current
threshold for "acceptable" unit test coverage was 80%.  That is right, our services we develop require
at least 80% unit test coverage in order for our build system, [drone.io][drone], to compile our services.

That was quite the discussion.  We talked probably for a full 45 minutes about what that means, and people
started finding reasons not to in particular cases.  For example, "Some of the logic that isn't covered currently
would take a lot of time to unit test, cost out weighs benefit." Another example, "Coverage doesn't equal 
quality."

It was pretty amazing seeing how everyone reacted to the concept of having all of our code 100% unit tested, 
when just 6 months ago everyone on the team agreed unanimously that we would maintain over 80% coverage in 
order for our software to build.  What is the difference?  Is it more reasonably easy to 80% versus 100%.  Is
the 80/20 adage real here?

## Continued with Options

We all know that the real problem we are trying to solve is the code quality problem.  To solve the quality
problem we need deterministic and objective mechanisms in place, which is where unit testing and code
coverage was created.  We can deterministic-ally run our tests, which either end in pass or fail state, and
we can objectively visualize through reports the exercised code branches.

This is great in theory, but often times developers create broad tests that cover large swaths of logic without
following that test run up with conditionals to see if the code worked properly... This is the way to cheat
the LOC coverage quality score.

Moreover, In golang we follow [this][mocking-go] model for unit test mocking in golang, which works great. It 
is a simple way to mock structs via interfaces, as well as monkey patch functions.  But, someone who isn't 
familiar with this testing technique for golang would look at our code and say, that is some ugly code...
Why on earth are you assigning stdlib functions to variables?? It just makes for some hard to follow code.

Though it is hard to follow, it allows us to expressly test every single conditional statement in our code, 
which increases quality, right?  Or does it diminish quality by the confusing mocking technique.

Looking at a few options other than a 100% mandate on code coverage, we came across [coveralls][coveralls].  I
use coveralls for my open source project [vestigo][vestigo] and I have been very impressed with it.  It basically
is a ratcheting unit test increasing platform.  You can have exactly the same LOC coverage, or more in your branch
to pass this gating mechanism.  This forces developers to write coverage for all new things created, which
in the end will approach 100% coverage.

## Another Option

In python there is a tool called [diff_cover][diff_cover], which will overlay a coverage report on top of a branch
or commit diff from git.  This is *fantastic* because you can see within the diff what the submitter missed
coverage wise with the commit.  

It would be sweet to one day run `go cover -diff=coverage.out` to get the same functionality, which is why I am going
to start a coverage/diff overlay project for golang shortly.  Marry that concept with the ability to perform a golint
and govet on the diff as well and you would have a great peer review tool.  This tool will tell you precisely what
the submitter is adding to the project, and if the code the submitter wrote is "good" based on the linting and coverage
of the diff.  

I feel like armed with this information, and being able to have a concise report of quality of the submission to affix to 
a pull request/merge request, we can have peers more likely to talk to each other about their submissions 
as opposed to blindly saying "LGTM" in the comment of the pull/merge request.


[external-coverage]: https://husobee.github.io/golang/test/coverage/2015/11/17/external-test-coverage.html
[drone]: http://drone.io
[mocking-go]: https://husobee.github.io/golang/testing/unit-test/2015/06/08/golang-unit-testing.html
[coveralls]: https://coveralls.io/
[vestigo]: https://github.com/husobee/vestigo
[diff-cover]: https://pypi.python.org/pypi/diff_cover
