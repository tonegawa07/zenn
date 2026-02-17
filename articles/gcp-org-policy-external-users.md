---
title: "Google Cloudで組織外ユーザーを招待する際の組織ポリシー設定"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud"]
published: true
---

## ドメイン制限で外部ユーザーが招待できない

Google Workspaceを使用している組織のGoogle Cloud環境で、外部のユーザー（組織外アカウント）にIAM権限を付与しようとしたところ、以下のエラーで弾かれました。

> ドメイン制限の組織のポリシーにより、ポリシーの更新中にエラーが発生しました。
> (IAM policy update failed due to the domain restriction organization policy.)

原因はGoogle Cloudの組織ポリシー `constraints/iam.allowedPolicyMemberDomains`（ドメインによるIDの制限）です。Google Workspaceの組織配下では、デフォルトで組織のCustomer IDのみが許可されているため、組織外のGmailアカウントや異なるGoogle Workspace組織のアカウントへの権限付与がブロックされます。

## 単純にドメイン制限を解除するとどうなるか

対象プロジェクトの組織ポリシーで `iam.allowedPolicyMemberDomains` を「すべて許可」にオーバーライドすれば、外部ユーザーの招待自体はできるようになります。

ただし、これだと開発者が誤って個人のGmailや無関係な外部アカウントに権限を付与することも可能になるため、セキュリティ上の懸念が残ります。

## 新しいマネージド制約で個別に制御する

`iam.allowedPolicyMemberDomains` は旧来の制約で、ドメイン単位でしか制御できません。これに対して `iam.managed.allowedPolicyMembers` はGoogleが後から追加した新しいマネージド制約で、「IAMポリシーに追加できるプリンシパル（ユーザーやグループ）」を個別に制御できます。dry-run（事前テスト）にも対応しており、Googleのドキュメントでもこちらが推奨されています。

構成としては以下の2段構えになります。

1. `iam.managed.allowedPolicyMembers`（IAMプリンシパルの制限）→ オンにして、許可したい対象を個別に指定する
2. `iam.allowedPolicyMemberDomains`（ドメイン制限）→ オフにしてドメイン単位のブロックを解除する

### 必要な権限

プロジェクトの組織ポリシーを変更するには、組織レベルでの **組織ポリシー管理者**（`roles/orgpolicy.policyAdmin`）が必要です。

組織ポリシーは組織 / フォルダ / プロジェクトの階層で継承されるため、どのレベルに設定するかはケースによります。今回は特定プロジェクトだけ外部ユーザーを受け入れる必要があったため、プロジェクト単位で設定しました。

### 許可リストを作成する

対象プロジェクトの「組織のポリシー」画面で `iam.managed.allowedPolicyMembers` を選択し、親のポリシーをオーバーライドして制約を有効化します。

この制約には `allowedPrincipalSets`（組織単位の許可）と `allowedMemberSubjects`（個別の許可）の2つのパラメータがあります。

```
# allowedPrincipalSets — 組織単位で許可する場合
//cloudresourcemanager.googleapis.com/organizations/YOUR_ORG_ID
```

```
# allowedMemberSubjects — 個別のユーザーやサービスアカウントを許可する場合
user:external-user@example.com
serviceAccount:external-sa@external-project.iam.gserviceaccount.com
```

### ドメイン制限を解除する

次に、同じ画面で `iam.allowedPolicyMemberDomains` を選択し、親のポリシーをオーバーライドしてポリシーの適用をオフにします。

この設定により、「旧来のドメイン制限は解除するが、IAM権限を付与できる相手は新しいマネージド制約でリストアップされた対象のみ」という状態になります。

なお、ドメイン制限を先に解除すると新しい制約が効くまでの間ガードがない状態になるため、`iam.managed.allowedPolicyMembers` を先に設定しておくのが安全です。

## まとめ

- 外部ユーザーへの権限付与エラーは `iam.allowedPolicyMemberDomains` によるドメイン制限が原因
- ドメイン制限を解除するだけだと任意の外部アカウントをIAMに追加できる状態になるため、セキュリティ上の懸念が残る
- 旧来のドメイン制限を解除したうえで、新しいマネージド制約 `iam.managed.allowedPolicyMembers` で許可したい外部ユーザーやグループを個別に指定するのがセキュア

## 参考

- https://docs.cloud.google.com/resource-manager/docs/organization-policy/restricting-domains?hl=ja
- https://dev.classmethod.jp/articles/organization-policy-domain-restriction-allowed-vs-managed-policy-members/
