---
layout: post
title: "Rust CI with GitHub Actions"
date: 2021-05-09 16:38:20 +0900
categories: rust githubactions ci devops 
lang: en
---

CI(Continuous Integration) is a concept that would interest most developers in a collaborative environment. CI automates the post-coding process of building and testing rather than doing it manually.

Through CI, developers can smoothly integrate their code with other people’s code. This allows developers to focus on coding without wasting their time on integration, which in turn increases productivity.

However, setting up CI can be troublesome and time-consuming, so developers have to decide the extent their CI process will cover.

In our case, we implemented a CI workflow for the Rust projects on GitHub Actions which includes linting, testing, code coverage reporting on CodeCov, and building for release.

Through this article, we hope you can have an easier time implementing CI for your Rust projects.

# Related Resources
- [GitHub Actions](https://github.com/features/actions): A feature on GitHub that allows users to *act(run a script)* on event triggers.
- [action-rs](https://github.com/actions-rs): A collection of commonly used GitHub Actions for Rust projects
- [CodeCov](https://www.codecov.io): A service that visualizes code coverage reports

# Project Structure
Example project using the workflow is uploaded in: [GitHub](github BamPeers/rust-ci-github-actions-workflow no-readme)

## Root Directory
- .gitignore: A list of files for git to ignore
- Cargo.toml: The manifest file for this cargo package
- README.md: The project manual

## .github/workflows/
This directory contains the workflow files.
- check-and-lint.yaml : Workflow for linting the code by running cargo check, fmt, and clippy
- release-packaging.yaml : Workflow for building the file and uploading the result as a downloadable artifact
- test.yaml : Workflow for running tests, measuring code coverage, and uploading respective results to GitHub and CodeCov

## src/
- lib.rs : A stub library file that includes an example function and test code
- main.rs : The main executable that runs and prints the result of the function in lib.rs


# Workflows
The following is a simple explanation of the included workflows. Check out our project [README](https://github.com/BamPeers/rust-ci-github-actions-workflow) for a more detailed explanation.

The **Check and Lint** and **Test with Code Coverage** workflows run on pull requests and on pushing to the main branch. The **Release Packaging** workflow runs on pushing to the main branch.

## Check and Lint (check-and-lint.yaml)
This workflow checks for compiler errors and code style inconsistencies.

The **Check job** runs `cargo check`.

The **Rustfmt job** runs `cargo fmt --check`. You can add a `rustfmt.toml` or `.rustfmt.toml` to configure the code style.

The **Clippy job** runs [clippy](https://github.com/rust-lang/rust-clippy) through [actions-rs/clippy-check@v1](https://github.com/actions-rs/clippy-check).
You can add a `clippy.toml` or `.clippy.toml` to configure the style.


## Test with Code Coverage (test.yaml)
This workflow runs tests, outputs test results, and publishes code coverage results on [CodeCov](https://codecov.io/).
Publishing test results and code coverage data is done in one job to avoid running the tests twice.

The environment variables for the job are set as shown below:
```yaml
env:
    PROJECT_NAME_UNDERSCORE: rust_ci_github_actions_workflow
    CARGO_INCREMENTAL: 0
    RUSTFLAGS: -Zprofile -Ccodegen-units=1 -Copt-level=0 -Clink-dead-code -Coverflow-checks=off -Zpanic_abort_tests -Cpanic=abort
    RUSTDOCFLAGS: -Cpanic=abort
```
`PROJECT_NAME_UNDERSCORE` environment variable should be replaced with your project name with - as _. Other environment variables are for code coverage.

The **Test job**, first, looks for a cache of the dependencies based on the hash of `Cargo.lock`.

Then, it runs `cargo test` on nightly Rust and uses `cargo2junit` to generate a JUnit format test result. And, it runs `grcov` and `rust-covfix` to generate proper code coverage data:

```yaml
- name: Generate test result and coverage report
    run: |
        cargo install cargo2junit grcov rust-covfix;
        cargo test --features coverage $CARGO_OPTIONS -- -Z unstable-options --format json | cargo2junit > results.xml;
        zip -0 ccov.zip `find . \( -name "$PROJECT_NAME_UNDERSCORE*.gc*" \) -print`;
        grcov ccov.zip -s . -t lcov --llvm --branch --ignore-not-existing --ignore "/*" --ignore "tests/*" -o lcov.info;
        rust-covfix -o lcov_correct.info lcov.info;
```

Test results are uploaded through [EnricoMi/publish-unit-test-result-action@v1](https://github.com/EnricoMi/publish-unit-test-result-action).

Code coverage results are uploaded to CodeCov through [codecov/codecov-action@v1](https://github.com/codecov/codecov-action). For private repositories, add your token from CodeCov repository setting to GitHub Secrets and uncomment the line: `token: ${{ secrets.CODECOV_TOKEN }}`.

## Release Packaging (release-packaging.yaml)
This workflow builds the package in release mode and uploads the resulting file as a GitHub artifact.

The included job uploads the project binary in `target/release` as an artifact through [actions/upload-artifact@v2](https://github.com/actions/upload-artifact).
You can configure which files to upload.

# Outcome
[Example pull request](https://github.com/BamPeers/rust-ci-github-actions-workflow/pull/1) with a failing test and clippy warning can be found in our repository.
Any failing job will block merging

## Clippy
- The action outputs result (**Clippy Output** added to a random workflow).
  ![Screen Shot 2021-05-01 at 6.06.28 PM](/assets/rust-ci-with-github-actions/clippy-output-into-github.png)
- For pull requests, it adds annotations on the diff.
  ![Screen Shot 2021-05-01 at 7.43.44 PM](/assets/rust-ci-with-github-actions/pr-annotation-by-clippy.png)

## Test Result
- The action outputs the test result (**Test Results** added to a random workflow).
  ![Screen Shot 2021-05-01 at 6.05.25 PM](/assets/rust-ci-with-github-actions/test-result-in-github.png)
- For pull requests, the action adds a comment containing the test results.
  ![Screen Shot 2021-05-01 at 7.00.21 PM](/assets/rust-ci-with-github-actions/comment-of-test-in-pr.png)

## Code coverage
- Code coverage results can be seen on your CodeCov repository.
  ![Screen Shot 2021-05-01 at 6.56.49 PM](/assets/rust-ci-with-github-actions/report-in-codecov.png)
- For pull requests, the action adds a comment containing the code coverage report.
  ![Screen Shot 2021-05-01 at 7.00.33 PM](/assets/rust-ci-with-github-actions/comment-of-codecov-report-in-pr.png)
- You can also add a CodeCov badge on your README to display the coverage percentage like we did on ours. It can be found in the `Setting > Badge` section of your CodeCov repository.

## Release Packaging
- Artifacts can be downloaded from the Summary tab of the workflow.

## GitHub Pull Request Checks
You can set status checks as required for merging in `Settings > Branches > Branch protection rules`.
![Screen Shot 2021-05-01 at 7.47.38 PM](/assets/rust-ci-with-github-actions/setting-for-github-pr.png)
When one or more jobs fail, the PR merge box will look something like below:
![Screen Shot 2021-05-01 at 7.46.21 PM](/assets/rust-ci-with-github-actions/pr-merge-box-in-github.png)

# Conclusion
We introduced CI for our Rust project through GitHub Actions. With this, the lint-build-test(+code coverage) process starts automatically when we push our code.

Questions and suggestions are welcome in the comment section.
