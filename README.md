⚡️ `travis-trigger-build.sh`
====

`travis-trigger-build.sh` is a short, self-contained Bash script for launching Travis build jobs from inside Travis build jobs.

## Example Usage
Say you have two projects, `Example/Downstream`, and `Example/Upstream`.

Imagine that you wish to trigger a build of the `Example/Downstream` project at the end of every `Example/Upstream` build.

First, you need to obtain an authentication token. You can do so with the [`travis` gem](https://github.com/travis-ci/travis.rb).

```bash
$ travis login --org
$ travis token --org  # Save this token
```

In your Travis build admin for `Example/Upstream`, you can then insert the generated token as a global secret environment variable named `EXAMPLE_DOWNSTREAM_TRAVIS_TOKEN`.

Finally, something like the following in the `.travis.yml` of the `Example/Upstream` would suffice for triggering builds of `Example/Downstream`:

```yaml
after_script:
- ./travis-trigger-build.sh "Example/Downstream" "${EXAMPLE_DOWNSTREAM_TRAVIS_TOKEN}"
```

And that's it! Builds of `Example/Upstream` will then wait on the corresponding `Example/Downstream` builds to complete, and fail themselves if the change caused the downstream build to fail.

## Environment Injection
`travis-trigger-build.sh` will inject build information from the triggering build instance into the configuration of the triggered build. In particular, the following global environment variables
are injected into `env.global`:

* `${UPSTREAM_PROJECT}_SLUG` -- the slug of the repository that contains the change.
* `${UPSTREAM_PROJECT}_BRANCH` -- the branch the change is on.
* `${UPSTREAM_PROJECT}_COMMIT_SHA` -- the specific commit hash of the change.

...where `UPSTREAM_PROJECT` is defined as the triggering build's repository slug, uppercased and with `-` replaced by `_`. That is, the slug of `some-org/some_Project` is forwarded as `SOME_ORG_SOME_PROJECT_SLUG`.

This can be useful when a downstream build needs perform special steps while setting up the upstream project.

For instance, in the DMOJ project, we maintain an integration testsuite under [**DMOJ/judge-testsuite**](https://github.com/DMOJ/judge-testsuite). When commits or pull requests to the testsuite are made, they are
automatically (via `travis-trigger-build.sh`) tested against the [**DMOJ/judge**](https://github.com/DMOJ/judge) project, which has something like this in its `.travis.yml`:

```yaml
install:
  - >
    git clone --depth 25 \
              --single-branch \
              --branch ${DMOJ_JUDGE_TESTSUITE_BRANCH:-master} \
              https://github.com/${DMOJ_JUDGE_TESTSUITE_SLUG:-DMOJ/judge-testsuite} testsuite &&
    git -C testsuite reset --hard ${DMOJ_JUDGE_TESTSUITE_COMMIT_SHA:-HEAD}
```

This ensures that if a pull request to the testsuite would cause the dependent project to fail, the pull request would be marked as broken in Github.
