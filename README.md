GitHub Action for Continuous Benchmarking
=========================================
[![Build Status][build-badge]][ci]

[This repository][proj] provides a [GitHub Action][github-action] for continuous benchmarking.
If your project has some benchmark suites, this action collects data from the benchmark outputs
and monitor the results on GitHub Actions workflow.

- This action stores collected benchmark results in [GitHub pages][gh-pages] branch and provides
  a chart view. You can see the visualized benchmark results on the GitHub pages of your project.
- This action can detect possible performance regressions by comparing benchmark results. When
  benchmark results are worse than previous exceeding specified threshold, it can raise an alert
  via commit comment or workflow failure.

This action currently supports following tools:

- [`cargo bench`][cargo-bench] for Rust projects
- `go test -bench` for Go projects
- [benchmark.js][benchmarkjs] for JavaScript/TypeScript projects
- [pytest-benchmark][] for Python projects with [pytest][]

Multiple languages in the same repository is supported for polyglot projects.

[Japanese Blog post](https://rhysd.hatenablog.com/entry/2019/11/11/131505)

## Examples

Example projects for each languages are in [examples/](./examples) directory. Live example workflow
definitions are in [.github/workflows/](./.github/workflows) directory. Live workflows are:

| Language   | Workflow                                                                                | Example Project                                |
|------------|-----------------------------------------------------------------------------------------|------------------------------------------------|
| Rust       | [![Rust Example Workflow][rust-badge]][rust-workflow-example]                           | [examples/rust](./examples/rust)               |
| Go         | [![Go Example Workflow][go-badge]][go-workflow-example]                                 | [examples/go](./examples/go)                   |
| JavaScript | [![JavaScript Example Workflow][benchmarkjs-badge]][benchmarkjs-workflow-example]       | [examples/benchmarkjs](./examples/benchmarkjs) |
| Python     | [![pytest-benchmark Example Workflow][pytest-benchmark-badge]][pytest-workflow-example] | [examples/pytest](./examples/pytest)           |

All benchmark charts from above workflows are gathered in GitHub pages:

https://rhysd.github.io/github-action-benchmark/dev/bench/

## Screenshots

### Charts on GitHub Pages

![page screenshot](https://github.com/rhysd/ss/blob/master/github-action-benchmark/main.png?raw=true)

Mouse over on data point shows a tooltip. It includes

- Commit hash
- Commit message
- Date and committer
- Benchmark value

Clicking data point in chart opens the commit page on GitHub repository.

![tooltip](https://github.com/rhysd/ss/blob/master/github-action-benchmark/tooltip.png?raw=true)

At bottom of the page, download button is available for downloading benchmark results as JSON file.

![download button](https://github.com/rhysd/ss/blob/master/github-action-benchmark/download.png?raw=true)

### Alert comment on commit page

This action can raise [an alert comment][alert-comment-example]. to the commit when its benchmark
results are worse than previous exceeding specified threshold.

![alert comment](https://github.com/rhysd/ss/blob/master/github-action-benchmark/alert-comment.png?raw=true)

## Why?

Since performance is important. Writing benchmarks is a very popular and correct way to visualize
a software performance. Benchmarks help us to keep performance and confirm effects of optimizations.
For keeping the performance, it's key to monitor the benchmark results for changes to the software.
To notice performance regression quickly, it's useful to monitor benchmarking results continously.

However, there is no good free tool to watch the performance easily and continuously across languages
(as far as I looked). So I built a new tool on top of GitHub Actions.

## How to use

This action takes a file which contains benchmark output and outputs the results to GitHub Pages branch
and/or alert commit comment.

- [Use this action with charts on GitHub pages](#use-this-action-with-charts-on-github-pages)
- [Use this action with alert commit comment](#use-this-action-with-alert-commit-comment)

### Use this action with charts on GitHub pages

Run your benchmarks on your workflow and store the output to a file. `tee` command is useful to output
results to both console and file.

e.g.

```yaml
- name: Run benchmark
  run: go test -bench 'Benchmark' | tee output.txt
```

Please add `rhysd/github-action-benchmark@{ver}` to your workflow yaml file.

e.g.

```yaml
- name: Store benchmark result
  uses: rhysd/github-action-benchmark@v1
  with:
    # Your favorite benchmark name
    name: My Project Go Benchmark
    # What benchmark tool the output.txt came from
    tool: 'go'
    # Where the output from the benchmark tool is stored
    output-file-path: output.txt
```

In the example, `name`, `tool`, `output-file-path` inputs are set and other inputs are set to default
values. Please read next section to know each input.

Above action step updates your GitHub pages branch automatically. By default it assumes `gh-pages`
branch. Please make it on your remote repository in advance.

Then push the branch to your remote.

As of now, [deploying GitHub Pages branch fails with `$GITHUB_TOKEN` automatically generated for workflows](https://github.community/t5/GitHub-Actions/Github-action-not-triggering-gh-pages-upon-push/td-p/26869).
`$GITHUB_TOKEN` can push branch to remote, but building GitHub Pages fails. Please read [issue #1](https://github.com/rhysd/github-action-benchmark/issues/1)
for more details.

To avoid this issue for now, you need to create your personal access token.

1. Go to your user settings page
2. Enter 'Developer settings' tab
3. Enter 'Personal access tokens' tab
4. Click 'Generate new token' and enter your favorite token name
5. Check `public_repo` scope for `git push` and click 'Generate token' at bottom
6. Go to your repository settings page
7. Enter 'Secrets' tab
8. Create new `PERSONAL_GITHUB_TOKEN` secret with generated token string

This is a current limitation only for public repositories. For private repository, `secrets.GITHUB_TOKEN`
is available. In the future, this issue would be resolved and we could simply use `$GITHUB_TOKEN` to
deploy GitHub Pages branch. Let's back to workflow YAML file.

There are two options for pushing GitHub pages branch to remote from workflow. Please choose one of them.

1. Give your API token to `github-token` input and set `auto-push` to `true`
2. Add step for executing `git push` within workflow

#### 1. Give your API token to `github-token` input and set `auto-push` to `true`

e.g.

```yaml
- name: Store benchmark result
  uses: rhysd/github-action-benchmark@v1
  with:
    name: My Project Go Benchmark
    tool: 'go'
    output-file-path: output.txt
    # Personal access token to deploy GitHub Pages branch
    github-token: ${{ secrets.PERSONAL_GITHUB_TOKEN }}
    # Push and deploy GitHub pages branch automatically
    auto-push: true
```

Just after generating a commit to update benchmark results, github-action-benchmark pushes the commit
to GitHub Pages branch. As a bonus, this action pulls the branch before generating a commit. It helps
to avoid conflicts among multiple workflows which deploy GitHub pages branch.

#### 2. Add step for executing `git push` within workflow

e.g.

```yaml
- name: Push benchmark result
  run: git push 'https://you:${{ secrets.PERSONAL_GITHUB_TOKEN }}@github.com/you/repo-name.git' gh-pages:gh-pages
```

If you don't set `auto-push` input, this action does not push changes to remote automatically. This
might be an option if you don't want to give API token to this action. Instead, So you need to push
GitHub pages branch by your own.

Note that GitHub pages branch and a directory to put benchmark results are customizable by inputs.

After first job execution, `https://you.github.io/dev/bench` should be available like
[examples of this repository][examples-page].

### Use this action with alert commit comment

This section explains how to enable performance alerts. To simplify explanation, configuration in
this section only enables alerts. If you want to know how to both GitHub Pages and alerts, please
see [live workflow examples](.github/workflows) for [example projects](examples/) after reading
this section.

e.g.

```yaml
- name: Store benchmark result
  uses: rhysd/github-action-benchmark@v1
  with:
    name: My Project Go Benchmark
    tool: 'go'
    output-file-path: output.txt
    # Prepare dedicated branch to store benchmark results
    gh-pages-branch: benchmark-data
    benchmark-data-dir-path: ./
    # If you don't deploy GitHub Pages, using secrets.GITHUB_TOKEN is sufficient.
    # You don't need to create and manage personal access token.
    github-token: ${{ secrets.GITHUB_TOKEN }}
    auto-push: true
    # Send commit comment when alert happens
    comment-on-alert: true
    # Alert happens when benchmark result gets worse exceeding 200% threshold
    alert-threshold: 200%
    # Workflow will fail when alert happens
    fail-on-alert: true
    # Users mentioned in alert commit message
    alert-comment-cc-users: '@rhysd'
```

With above inputs, this action works as follows:

1. extracts benchmark results from `output.txt`
2. stores (commits and pushes) the results in `benchmark-data` branch
3. compares the results with previous results stored in `benchmark-data` branch
4. if some current result is worse than previous exceeding 200% threshold,
    1. raise an alert as commit comment
    2. marks the workflow fail with alert message

In above example, when some benchmark result of current commit gets worse than previous exceeding 200%
threshold specified with `alert-threshold` input, an alert happens. For example, if previous benchmark
result was 100 iter/ns and this time it is 230 iter/ns, it means 230% worse than previous and alert
will happen.

When `comment-on-alert` is set, a commit comment is generated for the alert [like this][alert-comment-example].
Please ensure to set `github-token` input as well to for sending commit comment with GitHub API.

When `fail-on-alert` is enabled, workflow will fail when alert happens. Since
workflow immediately stops when it fails, please set `auto-push` to `true` also. Otherwise the benchmark
result won't be pushed to remote.

Optional `alert-comment-cc-users` specifies users which will be mentioned in alert comment so that they
can notice the alert comment easily via notification. Pleaes note that this value must be quated like
`'@rhysd'` because [`@` is an indicator in YAML syntax](https://yaml.org/spec/1.2/spec.html#id2772075).

If you don't use GitHub Pages, please make a dedicated branch to store benchmark results (`benchmark-data`
in above example). Since the branch is not for GitHub Pages, it is not deployed even if this action pushes
auto generated commit to the branch. And a personal access token is not necessary. `secrets.GITHUB_TOKEN`
is sufficient for `git push` and sending a commit comment. If you only use alerts, you don't need to
create and manage a personal access token.

### Tool specific setup

Please read `README.md` files at each example directory. Basically take stdout from benchmark tool and
store it to file. Then specify the file path to `output-file-path` input.

- [`cargo bench` for Rust projects](./examples/rust/README.md)
- [`go test` for Go projects](./examples/go/README.md)
- [Benchmark.js for JavaScript/TypeScript projects](./examples/benchmarkjs/README.md)
- [pytest-benchmark for Python projects with pytest](./examples/pytest/README.md)

These examples are run in workflows of this repository as described in 'Examples' section above.

### Action inputs

Input definitions are written in [action.yml](./action.yml).

| Name                      | Description                                                                                                                   | Type                                                  | Required | Default       |
|---------------------------|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|----------|---------------|
| `name`                    | Name of the benchmark. This value must be identical across all benchmarks in your repository                                  | String                                                | Yes      | `"Benchmark"` |
| `tool`                    | Tool for running benchmark                                                                                                    | One of `"cargo"`, `"go"`, `"benchmarkjs"`, `"pytest"` | Yes      |               |
| `output-file-path`        | Path to file which contains the benchmark output. Relative to repository root                                                 | String                                                | Yes      |               |
| `gh-pages-branch`         | Name of your GitHub pages branch                                                                                              | String                                                | Yes      | `"gh-pages"`  |
| `benchmark-data-dir-path` | Path to directory which contains benchmark files on GitHub pages branch. Relative to repository root                          | String                                                | Yes      | `"dev/bench"` |
| `github-token`            | GitHub API token. For public repo, personal access token is necessary. Please see basic usage                                 | String                                                | No       |               |
| `auto-push`               | If set to `true`, this action automatically pushes generated commit to GitHub Pages branch                                    | Boolean                                               | No       | `false`       |
| `alert-threshold`         | Percentage value like `"150%"`. If current benchmark result is worse than previous exceeding the threshold, alert will happen | String                                                | No       | `"200%"`      |
| `comment-on-alert`        | If set to `true`, this action will leave a commit comment when alert happens. `github-token` is necessary as well             | Boolean                                               | No       | `false`       |
| `fail-on-alert`           | If set to `true`, workflow will fail when alert happens                                                                       | Boolean                                               | No       | `false`       |

`name` and `tool` must be specified in workflow at `uses` section of job step.

Other inputs have default values. By default, they assume that GitHub pages is hosted at `gh-pages`
branch and benchmark results are available at `https://you.github.io/repo-name/dev/bench`.

If you're using `docs/` directory of `master` branch for GitHub pages, please set `gh-pages-branch` to
`master` and `benchmark-data-dir-path` to directory under `docs` like `docs/dev/bench`.

### Caveats

#### Run only on your branches

Please ensure that your benchmark workflow runs only on your branches. Please avoid running it on
pull requests. If branch were pushed to GitHub pages branch on pull request, anyone who creates a
pull request on your repository could modify your GitHub pages branch.

For this, you can specify branch which runs your benchmark workflow on `on:` section. Or set proper
condition to `if:` section of step which pushes GitHub pages.

e.g. Runs on only `master` branch

```yaml
on:
  push:
    branches:
      - master
```

e.g. Push when not running for pull request

```yaml
- name: Push benchmark result
  run: git push ...
  if: github.event_name != 'pull_request'
```

#### Stability of Virtual Environment

As far as watching the benchmark results of examples in this repository, amplitude of the benchmarks
is about +- 10~20%. If your benchmarks use some resources such as networks or file I/O, the amplitude
might be bigger.

If the amplitude is not acceptable, please prepare a stable environment to run benchmarks.
GitHub action supports [self-hosted runners](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/about-self-hosted-runners).

### Customizing benchmarks result page

This action creates the default `index.html` in the directory specified with `benchmark-data-dir-path`
input. By default every benchmark test case has its own chart in the page. Charts are drawn with
[Chart.js](https://www.chartjs.org/).

If it does not fit to your use case, please modify the HTML file or replace it with your favorite one.
Every benchmark data is stored in `window.BENCHMARK_DATA` so you can create your favorite view.

### Versioning

This action conforms semantic versioning 2.0.

For example, `rhysd/github-action-benchmark@v1` means the latest version of `1.x.y`. And
`rhysd/github-action-benchmark@v1.0.2` always uses `v1.0.2` even if newer version is published.

`master` branch of this repository is for development and does not work as action.

### Track updates of this action

To notice new version releases, please [watch 'release only'][help-watch-release] at [this repository][proj].
Every release will appear in your GitHub notifications page.

## Future work

- Allow user defined benchmark tool
  - Accept benchmark result as an array of benchmark results as JSON. User can generate JSON file
    to integrate any benchmarking tool to this action
- Allow to upload results to metrics service such as [mackerel](https://mackerel.io/) instead of
  updating GitHub pages
- Support pull requests. Instead of updating GitHub pages, add comment to the pull request to explain
  benchmark result.
- Add an alert comment to commit page when the benchmark result of the commit is far worse than
  previous one.
- Add more benchmark tools:
  - [Google's C++ Benchmark framework](https://github.com/google/benchmark)
  - [airspeed-velocity Python benchmarking tool](https://github.com/airspeed-velocity/asv)

## License

[the MIT License](./LICENSE.txt)

[build-badge]: https://github.com/rhysd/github-action-benchmark/workflows/CI/badge.svg
[ci]: https://github.com/rhysd/github-action-benchmark/actions?query=workflow%3ACI
[proj]: https://github.com/rhysd/github-action-benchmark
[rust-badge]: https://github.com/rhysd/github-action-benchmark/workflows/Rust%20Example/badge.svg
[go-badge]: https://github.com/rhysd/github-action-benchmark/workflows/Go%20Example/badge.svg
[benchmarkjs-badge]: https://github.com/rhysd/github-action-benchmark/workflows/Benchmark.js%20Example/badge.svg
[pytest-benchmark-badge]: https://github.com/rhysd/github-action-benchmark/workflows/Python%20Example%20with%20pytest-benchmark/badge.svg
[github-action]: https://github.com/features/actions
[cargo-bench]: https://doc.rust-lang.org/cargo/commands/cargo-bench.html
[benchmarkjs]: https://benchmarkjs.com/
[gh-pages]: https://pages.github.com/
[examples-page]: https://rhysd.github.io/github-action-benchmark/dev/bench/
[pytest-benchmark]: https://pypi.org/project/pytest-benchmark/
[pytest]: https://pypi.org/project/pytest/
[alert-comment-example]: https://github.com/rhysd/github-action-benchmark/commit/077dde1c236baba9244caad4d9e82ea8399dae20#commitcomment-36047186
[rust-workflow-example]: https://github.com/rhysd/github-action-benchmark/actions?query=workflow%3A%22Rust+Example%22
[go-workflow-example]: https://github.com/rhysd/github-action-benchmark/actions?query=workflow%3A%22Go+Example%22
[benchmarkjs-workflow-example]: https://github.com/rhysd/github-action-benchmark/actions?query=workflow%3A%22Benchmark.js+Example%22
[pytest-workflow-example]: https://github.com/rhysd/github-action-benchmark/actions?query=workflow%3A%22Python+Example+with+pytest-benchmark%22
[help-watch-release]: https://help.github.com/en/github/receiving-notifications-about-activity-on-github/watching-and-unwatching-releases-for-a-repository
