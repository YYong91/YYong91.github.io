---
title: "CodeBuild webhook은 source.type=GITHUB일 때만 붙일 수 있다"
date: 2026-03-04
categories: ["til"]
tags: ["AWS", "CodeBuild", "CodePipeline", "CI/CD", "monorepo", "Terraform"]
project: "CUBIG Backend"
source_sessions: ["2026-03-04-stripe-test-fixes"]
summary: "Monorepo CI/CD 파이프라인 구성하면서 마주친 CodeBuild webhook source.type 제약과 GitHub Team plan 제한"
---

billing 서비스에 PR 테스트 게이트를 붙이는 작업을 했다. CodePipeline이 이미 있는 상황에서 PR 단위로 테스트를 돌리려면 CodeBuild webhook이 필요했는데, source.type 제약에서 한 번 막혔다.

## CodeBuild webhook은 source.type이 GITHUB일 때만 작동한다

처음에 CodeBuild 프로젝트를 CodePipeline 소스(`source.type = CODEPIPELINE`)로 만들어서 webhook을 붙이려 했다. Terraform 설정은 이런 식이었다.

```hcl
resource "aws_codebuild_project" "billing_test" {
  name = "billing-test"

  source {
    type = "CODEPIPELINE"  # 파이프라인 소스
  }
}

resource "aws_codebuild_webhook" "billing_test" {
  project_name = aws_codebuild_project.billing_test.name
  # ...
}
```

`terraform apply`는 성공했지만, webhook이 동작하지 않았다. AWS 문서를 확인하니 CodeBuild webhook은 `source.type`이 반드시 `GITHUB`, `BITBUCKET`, `GITHUB_ENTERPRISE` 중 하나여야 한다. `CODEPIPELINE` 소스에는 webhook을 붙일 수 없다.

PR 테스트용 CodeBuild 프로젝트는 별도로 만들어야 했다.

```hcl
resource "aws_codebuild_project" "billing_pr_test" {
  name = "billing-pr-test"

  source {
    type            = "GITHUB"  # webhook 붙이려면 GITHUB 필수
    location        = "https://github.com/cubigcorp/core_backend"
    buildspec       = "apps/billing/buildspec/test.yml"
  }
}
```

## Monorepo에서 webhook 필터링: FILE_PATH

monorepo에서 billing 관련 변경이 있을 때만 테스트를 트리거하려면 `FILE_PATH` 필터를 쓴다.

```hcl
resource "aws_codebuild_webhook" "billing_pr_test" {
  project_name  = aws_codebuild_project.billing_pr_test.name
  build_type    = "BUILD"

  filter_group {
    filter {
      type    = "EVENT"
      pattern = "PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED, PULL_REQUEST_REOPENED"
    }

    filter {
      type    = "FILE_PATH"
      pattern = "^apps/billing/.*"  # billing 디렉토리 하위 변경만
    }
  }
}
```

`FILE_PATH` 필터는 PR에서 변경된 파일 경로 기준으로 트리거 여부를 결정한다. `packages/shared/` 변경은 여러 서비스에 영향을 주므로 별도 필터 그룹으로 추가하거나, 공유 패키지 변경은 별도 파이프라인에서 다루는 게 낫다.

서비스가 여러 개면 서비스 수만큼 CodeBuild + webhook 쌍이 필요하다. 3개 서비스 = 3개 CodeBuild 프로젝트 + 3개 GitHub 웹훅이다.

## GitHub Team plan: Terraform이 webhook을 만들어도 수동 확인이 필요하다

`aws_codebuild_webhook` Terraform 리소스가 apply에 성공하더라도, GitHub 저장소 설정에서 webhook이 실제로 등록됐는지 확인해야 한다.

GitHub Free plan에서는 CodeBuild가 API를 통해 webhook을 자동으로 등록하지 못하는 경우가 있다. GitHub Team plan 이상에서는 자동 등록이 되지만, 조직 보안 정책에 따라 수동 승인이 필요할 수도 있다.

GitHub 저장소 → Settings → Webhooks에서 CodeBuild endpoint가 등록됐는지, 최근 delivery가 성공했는지 직접 확인하는 습관을 들이는 게 좋다. Terraform apply 성공이 webhook 동작 성공을 보장하지는 않는다.

## Dev(CD) vs Prod(CI/CD) 파이프라인 명명

파이프라인 구조를 정리하면서 네이밍도 명확하게 잡았다.

| 파이프라인 | 환경 | 스테이지 수 | 역할 |
|-----------|------|------------|------|
| `billing-cd-dev` | Dev | 4단계 | Source → Build → Migrate → Deploy |
| `billing-cicd-prod` | Prod | 5단계 | Source → Test → Build → Migrate → Deploy |

Dev 파이프라인은 PR gate에서 이미 테스트를 통과했으니 파이프라인에서 다시 돌릴 필요가 없다(CD 중심). Prod 파이프라인은 develop → main 릴리즈 머지이므로 안전망으로 테스트를 포함한다(CI/CD).

## ECS one-off task 실행 시 IAM 역할과 보안 그룹을 명시해야 한다

DB 마이그레이션이나 배치 작업을 ECS one-off task로 실행할 때 자주 놓치는 부분이다.

```bash
aws ecs run-task \
  --cluster cubig-ecs-cluster-dev \
  --task-definition billing-migration \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-xxx],
    securityGroups=[sg-xxx],
    assignPublicIp=DISABLED
  }"
```

`securityGroups`를 생략하면 기본 VPC 보안 그룹이 붙어서 DB 접근이 막히고, task가 시작 직후 종료된다. CloudWatch 로그에 연결 에러도 안 남는 경우가 있어서 원인 파악이 늦어진다. 항상 보안 그룹을 명시하는 게 안전하다.
