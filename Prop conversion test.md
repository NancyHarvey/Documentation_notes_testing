<table>
  <tr>
    <td>A replacement for the Kubernetes submit queue: Tide
shared with kubernetes-dev and kubernetes-sig-testing</td>
    <td>Summary
We will replace the Kubernetes submit queue with a program called tide. It will solve several key problems that are difficult to solve with the submit queue.
Author: spxtr
Contributors: fejta, colew
Status: Draft
Created: 2017-07-28</td>
  </tr>
  <tr>
    <td>Spacer row, do not delete :-)</td>
    <td></td>
  </tr>
</table>


# Background

## The current solution: mungegithub’s submit queue

The most basic purpose of the submit queue is to trigger tests for and then quickly merge approved pull requests that have passed tests against master. Any replacement must at the very least satisfy this same purpose.

### Trigger tests

Currently the submit queue triggers tests in two ways. The serial way is by saying /test all on the head PR. The batch way is to expose the queue to prow, which will periodically trigger a batch run. For the serial way, it will wait for a success result on the GitHub context lines themselves. For the batch way, it will ask prow for successful batch results.

### Quickly

Our tests take up to an hour to run, and they are flaky. It would be unacceptable to only merge twenty or thirty PRs daily, so we implemented batch merging. We currently test a handful of PRs at a time and merge them all if tests pass.

### Approved

We have an often-changing set of labels and processes that determine what PRs can be merged. At the very least, the PRs need lgtm, approved, and cncf-cla: yes labels. Different users may require different labels. Note that the actual application of these labels is not handled by the submit queue, but instead by either prow or other mungers. The submit queue passively consumes them and then clicks the merge button.

### Passed tests against master

It’s possible for two PRs that have no merge conflicts and which pass tests individually against master to fail tests when merged together.

## Problems with mungegithub’s submit queue

![image alt text](image_0.jpg)

* It is stateful and does not gracefully restart. Because it’s just another munger, we need to restart it any time any of the mungegithub code changes. The most important state is the queue itself, the current tests, and also the history. On restart, it needs to retrigger tests, so it effectively loses one merge cycle. It keeps a cache of GitHub objects in order to lower API token usage.

* It is inherently tied to an individual repository. There are those who want to split up Kubernetes into many repositories. Only a few of the current repos have submit queues leading to an inconsistent experience across them. It is not trivial to set up a new submit queue for a new repo. Testing across multiple branches in the same repository can be confusing, especially with special cherrypick labels.

* The code is dangerously complex.

    * It’s a munger. Mungegithub lists issues and munges them. It has some nice caching features so that it only looks at changed issues, and is pretty decent at what it was originally written to do. However, munging issues is a much broader  process than the ideal submit queue, which lists PRs to merge, compares that list with tests, triggers tests, and merges the PRs. The extra functionality specific to munging issues makes for difficult code.

    * It has a long and interesting history. When it was first written it had to trigger jobs on Jenkins by saying @k8s-bot test this, or trigger jobs on Travis by closing and reopening the PR. Now it triggers jobs passively by simply existing, since prow reads its state to start batch jobs. It used to care about only merging if post-submit jobs are passing, but it no longer does. It used to read GitHub statuses for test results, but now it reads GCS. GitHub’s API has also changed dramatically in the last few years. History is nice when you want to study history, but it’s not nice when you want a predictable, reliable system.

* Queue-hopping mechanisms are annoying and slow down merges. A concerned coworker asked me today, "my PR has been in the queue for six hours but it has actually moved down in the queue. How is this possible?" That’s a great question, and the answer is that the queue is sorted by priority and time-of-first-lgtm. This is because when PRs flake they drop off the queue, and when they pass retests they get put back in the same place. This scheme indicates an important property: that the longer a PR is on the queue, the sooner it will be merged. Ideally only critical fixes can hop the queue, and those should be manually merged. There’s all this complex logic in the submit queue that tries to order PRs in the queue when really we should be optimizing to empty the queue. Nobody cares about queue order if the queue is empty enough that one or two batch merges clear the whole thing. However, when the queue does get backed up due to flakes and end-of-quarter pressure, we may want some weak prioritization.

Some of these problems can be improved without a rewrite. For instance, we can mitigate the pain of restarts by restarting less often, which we can accomplish by running the submit queue as a separate munger from the rest (DONE). We can also use a tool like helm to mitigate the cost of spinning up new submit queues for new repos, but the fact still remains that we do need a submit queue per repo. We can make the submit queue ignore queue-hopping if we’ve already triggered a batch run and only consider it when triggering the next.

# Implementation: the tide pool

We will implement a program called cmd/tide that runs in the prow cluster as an independent binary.  It will use GitHub’s search API to generate a set of PRs ready for merge across multiple repositories which we will call the "tide pool." It will query the prow cluster for test statuses, then decide to merge PRs or trigger tests based on these two state sources.

Note that this will not be a stateful priority queue. It will **not** be structured in three phases: trigger tests, wait for tests to complete, then merge PRs, even though this is the expected order of events that will actually happen. The tide program will instead look a little more like a Kubernetes controller: it will inspect the state of the world, and then trigger tests and/or merge PRs based on it.

## Core loop

It will do the following every few minutes:

1. Do a search for pull requests matching a query specified in the config, and add all resulting PRs to the tide pool. An example query for kubernetes/test-infra is type:pr state:open org:kubernetes repo:test-infra label:lgtm label:"cncf-cla: yes”. For kubernetes/kubernetes, we will need to also tack on label:approved. A few facts about GitHub’s search API:

    1. Queries can cover multiple orgs or repos. If we use the same query string for the entire kubernetes org then we can use one query string to list all merge-ready pull requests in the org. Our code can take advantage of this.

    2. GitHub’s GraphQL API lets us run searches. They are [paginated](http://graphql.org/learn/pagination/) and return a maximum of 100 results per query. However, we get 5000 points per hour separate from the REST v3 API, which is nice. The docs are [here](https://developer.github.com/v4/guides/resource-limitations/). I don’t expect this to be a problem even if we scale up our testing by a few orders of magnitude.

2. Examine prow’s state by listing ProwJobs. Each ProwJob is a Kubernetes resource that records the status of a single test run. For instance, a ProwJob might say "pull-kubernetes-e2e-gce passed for PR #123 (commit beef567) merged into the master branch (commit abc123)." For each repository and branch represented in the tide pool, there are a few important cases to consider. In this table, I describe these cases. Note that the test results are only for tests and batches that are against the current branch head. I consider no result the same as a failed result.

<table>
  <tr>
    <td>Single Test Result</td>
    <td>Batch Test Result</td>
    <td>Action</td>
  </tr>
  <tr>
    <td>none</td>
    <td>none</td>
    <td>Start PR test, start batch.</td>
  </tr>
  <tr>
    <td>pending</td>
    <td>none</td>
    <td>Start batch.</td>
  </tr>
  <tr>
    <td>success</td>
    <td>none</td>
    <td>Merge the PR. Transposes to (none, none).</td>
  </tr>
  <tr>
    <td>none</td>
    <td>pending</td>
    <td>Wait.</td>
  </tr>
  <tr>
    <td>pending</td>
    <td>pending</td>
    <td>Wait.</td>
  </tr>
  <tr>
    <td>success</td>
    <td>pending</td>
    <td>Wait.</td>
  </tr>
  <tr>
    <td>none</td>
    <td>success</td>
    <td>Merge all PRs. Transposes to (none, none).</td>
  </tr>
  <tr>
    <td>pending</td>
    <td>success</td>
    <td>Merge all PRs. Transposes to (none, none).</td>
  </tr>
  <tr>
    <td>success</td>
    <td>success</td>
    <td>Merge all PRs. Transposes to (none, none).</td>
  </tr>
</table>


For example assume we have PRs #2-10 in the pool. Master’s SHA is abc123, and PR #1 just merged.  We have a pending batch with PRs 1, 3, 5, and 6. We have a passing test for PR #2 against abc123. This is the (none, pending) case. In this case we must not merge PR #2, since that will invalidate our whole batch. Instead we simply wait for the batch to complete. Now assume that the batch failed. This is the (success, none) case, since PR #2 passed against the latest master. In this case we merge PR #2 right away, which puts us in (none, none), and so on.

There is some pretty tricky logic here that will require careful testing. One important note is that we don’t store any state across loop iterations. Another important note is that in the trivial case where one PR is in the pool for a repo, if it has up-to-date tests already then we do not need to test it again. We can just merge it.

We can consider giving priority to PRs with a label such as queue/critical-fix. These labels should be considered only when triggering new tests, so that we do not invalidate old test results. We can consider several priority schemes aside from the labels, such as PR age and time to first LGTM. The latter is currently used by the submit queue, and has the nice property that it is hard to game and it feels fair. Whatever schemes we use should be hard to game and should prefer merging PRs that have been mergeable for longer. This is difficult to solve precisely, so we will need to use a heuristic.

## Configuration

In our prow config, we can specify a list of queries whose union will be the tide pool. The following example represents a fairly simple idea. kubernetes/kubernetes PRs to the master branch need a few labels. kubernetes/kubernetes PRs to release branches need cherrypick-approved, plus those labels. All other kubernetes repos don’t need approved.

<table>
  <tr>
    <td>tide:
- type:pr state:open org:kubernetes -repo:kubernetes/kubernetes label:lgtm -label:do-not-merge
- type:pr state:open repo:kubernetes/kubernetes label:lgtm label:cherrypick-approved label:approved -label:do-not-merge -base:master
- type:pr state:open repo:kubernetes/kubernetes label:lgtm label:approved -label:do-not-merge base:master</td>
  </tr>
</table>


We will need to clearly document that it is our desire to drive the world towards a simpler config. Ideally we only need one query for the entire org, because that would mean a consistent experience for all kubernetes repos. Certain repos may wish to be exempt because they have already established different processes. Eventually we should attempt to bring about consistency.

## User interface

Currently, going to [https://submit-queue.k8s.io/](https://submit-queue.k8s.io/) reveals something like this:

![image alt text](image_1.png)

That’s pretty nice. It’s easy for a PR author to see that their PR is on-track to merge. We don’t want to lose that nice behavior.

cmd/tide will sit behind an internal service and it will serve up the results from its latest search as a map from org -> repo -> []PR. Every minute, when cmd/deck updates its cache of ProwJobs, it will also update its cache by talking to cmd/tide. Then, we can expose an endpoint such as /pool that will nicely show the data. It can link directly to the test runs, too, which is nice.

I’m not interested in making an equivalent of the history page. However, I am interested in naturally answering the following:

1. Whether a given PR is in the tide pool and what is currently testing: just look at the dashboard described above.

2. What are the requirements for getting into the pool: we will provide the query used for each repo, possibly prettyfied so that labels and such are clear.

3. Why a given PR is not in the tide pool: the submit queue sticks a status context on the PR saying things like "this PR needs rebase," or “this required context isn’t passing: bla.” I don’t actually like this. When something is needed, we generally have a comment explaining how to satisfy the need. For instance, we have a big “this PR is NOT APPROVED” comment telling you exactly how to add the approved label. These comments are provided by the various plugins that add the labels, and they should be descriptive enough to get PRs into the pool. To be clear, the tide program will **not** have any special logic telling you the next step to getting your PR into the pool. The plugins that add the labels need to be individually excellent and usable.

4. How quickly we expect to merge things. This won’t be exact, but we can use the last day of data to estimate job pass rates. Then, given the pool size, we should be able to give merge time estimates for a PR in the pool at 50% and 80% confidence intervals. Explicit ordering schemes break these estimates. If we decide to honor priority labels or prefer testing certain PRs then it will be possible for a PR to sit in the pool for a long time without making progress.

## Implementation Details

The binary itself will have the same overall look and feel as cmd/plank. The deployment YAML will look like [plank_deployment.yaml](https://github.com/kubernetes/test-infra/blob/f0808c2479723047bf5afbd3b0eec08539405622/prow/cluster/plank_deployment.yaml). The main function and dockerfile will look like [plank’s](https://github.com/kubernetes/test-infra/tree/f0808c2479723047bf5afbd3b0eec08539405622/prow/cmd/plank) as well.

The main control loop will look something like this. The queries function returns the queries from the config. The splitPool function splits the pool into subpools for each repo and branch. The syncSubpool function implements the table from the core loop.

<table>
  <tr>
    <td>func (c *Controller) Sync() error {
	pjs, err := c.kc.ListProwJobs(nil)	if err != nil {		return err	}
	var pool []github.PullRequest
	for _, q := range queries(c.ca.Config()) {
		prs, err := c.ghc.SearchPulls(q)
		if err != nil {
			return err
		}
		pool = append(pool, prs...)
	}
	for repo, subpool := range dividePool(pool) {
		if err := c.syncSubpool(pjs, repo, subpool); err != nil {
			return err
		}
	}
	return nil
}</td>
  </tr>
</table>


The code that determines if a batch is valid will be copied from the [submit queue](https://github.com/kubernetes/test-infra/blob/10c38f41c5d4ef376bcce2f5a066b01bfd903410/mungegithub/mungers/submit-queue-batch.go). The code that determines how to start a batch will be copied from [cmd/splice](https://github.com/kubernetes/test-infra/blob/10c38f41c5d4ef376bcce2f5a066b01bfd903410/prow/cmd/splice/main.go), although I will use the git client since we will be checking out many repos.

We will use GitHub’s [GraphQL search API](https://developer.github.com/v4/reference/query/#connections), the [merge button](https://developer.github.com/v3/pulls/#merge-a-pull-request-merge-button), and probably the [issue comment button](https://developer.github.com/v3/issues/comments/#create-a-comment). We can consider merging the PRs locally and simply pushing the merged result, rather than clicking the merge button.

## Using GraphQL to make searches return the data we need

GitHub’s v3 search API is pretty terrible. It’s missing all sorts of important fields, which means we’ll have to do GET requests to fetch more information on the PRs in the pool. Instead, we can use their [Grap](https://developer.github.com/v4/)[hQL API](https://developer.github.com/v4/). Here is an example search:

<table>
  <tr>
    <td>{
  rateLimit {
    limit
    cost
    remaining
    resetAt
  }
  search(first: 100, query: "org:kubernetes label:lgtm state:open is:pr", type: ISSUE) {
    repositoryCount
    issueCount
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      ... on PullRequest {
        number
        baseRefName
        repository {
          name
          owner {
            login
          }
        }
        commits(last: 1) {
          nodes {
            commit {
              oid
            }
          }
        }
      }
    }
  }
}</td>
  </tr>
</table>


It looks intimidating, but actually this is really nice.  It returns PR numbers, head commits, and repository information. If we find that we need more information we simply add it to the query. Neat!

The only downside is that it’s not a simple JSON API, which means our code will feel a little special-cased at times. We can consider using [shurcooL/githubql](https://github.com/shurcooL/githubql), which seems well thought-out.

