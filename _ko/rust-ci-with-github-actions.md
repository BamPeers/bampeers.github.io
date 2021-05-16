---
layout: post
title: "GitHub Actions로 하는 Rust CI"
date: 2021-05-09 16:38:20 +0900
categories: rust githubactions ci devops
lang: ko
---

CI(continuous integration)는 협업 환경에서 일하는 개발자라면 모두가 관심을 가질만한 주제이다. CI는 개발 이후 과정인 빌드, 테스트까지의 과정을 사람이 수동으로 하는 것이 아니라 자동화된 프로세스에 의해서 동작하도록 한 것이다.
개발자들은 CI를 통하여 여러 사람들이 만든 개발 코드를 통합하는 과정을 자연스럽게 할 수 있다. 이를 통해서 개발자는 통합과정을 신경쓰지 않고 개발에만 집중할 수 있어 생산성을 증진시킬 수 있다.

한편, CI를 세팅하는 것은 매우 불편하고 시간이 많이 소모되는 작업이다. 그리고, 근무중인 조직의 개발 문화에 따라서 어느정도까지 세팅할 것인지 결정하는 것도 중요하게 고민해야할 사항이다.
이 포스팅에서는 Rust에서 많이 사용하는  Lint/fmt, Unit Test/Integration Test를 하고, 작성된 테스트 코드가 어느정도의 Code Coverage를 가지고 있는지를 측정하고 Codecov에 리포팅하는 기능을 CI로 구현한다. CI 플랫폼으로는 github-action을 사용하여 구현하였다.
독자분들은 이 글을 통해 Rust 프로젝트의 CI/CD를 쉽게 도입할 수 있을 것이다.

# 관련 요소
- [GitHub Actions](https://github.com/features/actions): 이벤트가 트리거 되었을 때 특정 *액션(스크립트)* 을 하도록 하는 기능
- [action-rs](https://github.com/actions-rs): Rust를 위한 github action 을 라이브러리화 시켜둔 툴킷
- [CodeCov](https://www.codecov.io): Code coverage 를 시각화해주는 툴 중 하나로, 다양한 CI툴과 연동하여 구현한 코드의 커버리지를 확인 가능한 솔루션

# 프로젝트 구조
관련된 예제 코드는 GitHub의 BamPeers/rust-ci-github-actions-workflow 에 업로드 되어 있다. [링크](BamPeers/rust-ci-github-actions-workflow no-readme)

## 루트 디렉토리
- .gitignore: git프로젝트의 무시 목록
- Cargo.toml: cargo package의 매니페스트 파일
- README.md: 프로젝트 설명서

## .github/workflows/
이 디렉토리는 github action workflow 파일이 저장되어 있다.
- check-and-lint.yaml : lint를 하기 위한 파일로써, rust에서 제공하는 cargo check, fmt, clippy 를 실행
- release-packaging.yaml : cargo build --release를 하고 빌드 후 실행파일을 업로드
- test.yaml : cargo test를 실행하고, 테스트 커버리지를 측정하고 결과를 github와 codecov에 업로드

## src/
- lib.rs : lib stub 파일로 예제 함수와 테스트 코드가 구현 되어 있는 파일
- main.rs : main 실행파일로 lib에 있는 함수를 호출하고 출력하는 파일


# 워크플로우
이 항목에서는 포함된 워크플로우의 실행방법에 대하여 간단히 설명한다. 해당 워크플로우의 자세한 실행방법은 README파일을 확인하면 좀 더 구체적으로 알 수 있다.

**Check and Lint** 와 **Test with Code Coverage** 는 `Pull Request`시 `Main` 브랜치로 푸시할 때 실행된다.

## 체크/린 (check-and-lint.yaml)
이 워크플로우는 컴파일러 에러와 코드 스타일이 맞지 않는 것을 확인한다.

**Check Job** 은 `cargo check` 를 실행한다.

**Rustfmt job** 은 `cargo fmt --check` 를 실행한다. `rustfmt.toml` 이나 `.rustfmt.toml` 를 통해 코드 스타일 세팅을 변경할 수 있다.

**Clippy job** 은 [actions-rs/clippy-check@v1](https://github.com/actions-rs/clippy-check) 을 통하여  [clippy](https://github.com/rust-lang/rust-clippy) 를 실행한다. `clippy.toml` 이나 `.clippy.toml` 을 추가하여 clippy 세팅을 변경할 수 있다.


## Test with Code Coverage (test.yaml)
이 워크플로우는 테스트를 실행하고, 테스트의 결과를 출력하며, 코드 결과를 [CodeCov](https://codecov.io/) 로 업로드 한다. 테스트 결과와 코드커버리지 데이터의 업로드가 끝나면 테스트가 두번 실행되는 것을 방지하기 위해 하나의 작업만 실행한다. 

Job의 환경변수는 아래와 같이 세팅한다.

```yaml
env:
    PROJECT_NAME_UNDERSCORE: rust_ci_github_actions_workflow
    CARGO_INCREMENTAL: 0
    RUSTFLAGS: -Zprofile -Ccodegen-units=1 -Copt-level=0 -Clink-dead-code -Coverflow-checks=off -Zpanic_abort_tests -Cpanic=abort
    RUSTDOCFLAGS: -Cpanic=abort
```
`PROJECT_NAME_UNDERSCORE` 환경 변수는 프로젝트 이름을 ‘-’에서 ‘_’로 바꿔서 지정한다. 그 외에 다른 환경변수는 코드 커버리지와 관련된 것이다.

테스트 작업은 먼저 `Cargo.lock` 파일의 해쉬를 기반으로 만들어진 디펜던시의 캐시를 확인한다.

아래의 코드는 나이틀리 기반의 `cargo test`를 실행하고, `cargo2junit` 을 사용하여 테스트 결과를 junit 형식으로 생성한다. 그리고 `grcov`와 `rust-covfix` 위에서 실행한 결과를 바탕으로 코드 커버리지 데이터를 생성한다.

```yaml
- name: Generate test result and coverage report
    run: |
        cargo install cargo2junit grcov rust-covfix;
        cargo test --features coverage $CARGO_OPTIONS -- -Z unstable-options --format json | cargo2junit > results.xml;
        zip -0 ccov.zip `find . \( -name "$PROJECT_NAME_UNDERSCORE*.gc*" \) -print`;
        grcov ccov.zip -s . -t lcov --llvm --branch --ignore-not-existing --ignore "/*" --ignore "tests/*" -o lcov.info;
        rust-covfix -o lcov_correct.info lcov.info;
```

테스트 결과는 [EnricoMi/publish-unit-test-result-action@v1](https://github.com/EnricoMi/publish-unit-test-result-action) 를 통하여 업로드 된다.

코드 커버리지 결과는 [codecov/codecov-action@v1](https://github.com/codecov/codecov-action) 를 통하여 CodeCov 로 업로드 된다. 사설 저장소의 경우에는 CodeCov에 Github 시크릿을 세팅하고 `token: ${{ secrets.CODECOV_TOKEN }}` 의 주석을 지운다.

## 릴리즈 패키징 (release-packaging.yaml)
이 워크플로우는 릴리즈 모드에서 패키지를 빌드하고 깃허브 아티팩트에 결과물을 업로드한다.

`target/release` 에 있는 프로젝트 바이너리를 [actions/upload-artifact@v2](https://github.com/actions/upload-artifact) 통해서 아티팩트로 업로드한다. 업로드하고자 하는 파일은 변경 가능하다.

# 작업결과
테스트 저장소에서 테스트 실패 및 Clippy의 warning의 [Pull Request 예제](https://github.com/BamPeers/rust-ci-github-actions-workflow/pull/1) 를 확인 가능하다. 
작업이 실패하면 Merge를 할 수 없도록 되어 있다.

## Clippy
- 이 액션은 결과물을 임위의 워크플로우에 **Clippy Output** 로 등록한다.
  ![Screen Shot 2021-05-01 at 6.06.28 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qmtp0h9an82orf366f5a.png)
- 풀 리퀘스트인 경우에 diff 항목에 어노테이션을 추가한다.
  ![Screen Shot 2021-05-01 at 7.43.44 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ld7ym1xfiibdqeob3otv.png)

## 테스트 결과
- 이 액션은 결과물을 임위의 워크플로우에 **Test Results** 로 등록한다.
  ![Screen Shot 2021-05-01 at 6.05.25 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e65gc2awy7pc5m9u7rm5.png)
- 풀 리퀘스트인 경우에 이 액션은 테스트 결과가 포함된 댓글을 단다.
  ![Screen Shot 2021-05-01 at 7.00.21 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kk2baz1fakm8exu1pqcr.png)

## 코드 커버리지
- 코드 커비리지 결과는 Codecov 저장소에서 확인할 수 있다.
  ![Screen Shot 2021-05-01 at 6.56.49 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/byh4punhp9vzvo45uj78.png)
- 풀 리퀘스트인 경우에 이 액션은 코드 커버리지 리포트를 댓글로 단다.
  ![Screen Shot 2021-05-01 at 7.00.33 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c2rwexgtiw5eala1hu0b.png)
- README 파일과 같은 곳에 커버리지 범위를 보여주는 CodeCov 뱃지를 달 수 있다.

## 릴리즈 패키징
- 업로드된 아티팩트는 워크플로우의 요약 탭에서 다운로드 가능하다.

## Github Pull Request 체크
`Settings > Branches > Branch protection rules` 에서 머지할 때 빌드 상태 체크를 필수로 설정할 수 있다.
![Screen Shot 2021-05-01 at 7.47.38 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/06j2mfk60rpz142jn70c.png)
하나 이상의 작업이 실패했을 때, PR merge box 는 아래와 같이 나오는 것을 확인할 수 있다.
![Screen Shot 2021-05-01 at 7.46.21 PM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6hhkf0batw7un4pbpzzh.png)

# 결론
지금까지 Rust 프로젝트를 github-actions을 활용하여 CI를 도입하였고, 이를 통해서 자동적으로 lint, build, test(w/codecov), artifact 생성 기능을 성공적으로 완료하였다.
이 글을 통해 처음 Rust에 CI를 도입하는 사람들이 쉽게 본인들의 프로젝트에 적용하여 퍼포먼스를 개선할 수 있을 것으로 기대한다.
