## 概要
HACARUS 学生インターン生の山下です。今回は [HACARUS Check](https://check.hacarus.com/ja/) というプロダクトを開発する上で環境整備のため実装した、Gitlab における変更履歴の自動生成システムについて紹介します。

## 全体の構成
初めに、実装した自動生成機能の構成を載せておきます。

![flow]("/images/4-changelog-auto-generation-in-gitlab/flow.png")

開発者がリポジトリにタグ付きのコミットをプッシュすると、GitLab CI が動作します。  
CI により Python スクリプトが実行され、スクリプトは GitLab API からリポジトリの情報を受け取って changelog を出力します。  
作成された changelog の内容は CI を経由して GitLab Release CLI tool に渡され、リリースノートに反映されます。  
開発者は自動生成された changelog の内容を含むリリースノートをブラウザで閲覧できます。生成した changelog のファイル自体は、リポジトリにコミットされません。

## 背景
HACARUS Check のリポジトリでは、タグ付きのコミットをプッシュすると Windows 上で動作する GUI アプリ のインストーラーが自動でビルド・配布されるように設定されています。  
インストーラーの配布に伴って GitLab 上のリリースノートも自動で作成されますが、前バージョンからの変更内容は自動で記載されないため、どんな変更があったかが分かりにくいという問題点がありました。 
変更履歴を自動生成するツールは色々と存在するようですが、今回の環境（GitLab CI を Windows マシンで実行）で簡単に使えるものが見つからなかったため、自前で作成することにしました。

## GitLab CI/CD での処理
GitLab CI/CD は GitLab が提供する継続的インテグレーション/継続的デリバリーのためのツールです（詳しくはHACARUS技術ブログの[他記事](Issue3の記事リンク)でも紹介しています）。

設定ファイル `.gitlab-ci.yml` を編集することで、リポジトリへの push ごとに自動実行される動作などを設定できます。  
動作が複雑な場合は `.gitlab-ci.yml` に全ての内容を書くのではなく、以下のように処理の内容や使用する image に応じてステージを分割し、ステージごとに yaml ファイルを作成して include する形式にしておくと整理しやすいです。
```yaml:.gitlab-ci.yml
stages:
  - changelog
  - release

include:
  - local: /.gitlab/changelog.gitlab-ci.yml
  - local: /.gitlab/release.gitlab-ci.yml
```
今回は Python で changelog を作成するステージ（`changelog` ステージ）と、GitLab Release CLI tool でリリースノートを作成するステージ（`release` ステージ）に分割しています。  
ディレクトリ構造は以下のようになっています。
```
repositroy_root
├ .gitlab
│ ├ changelog.gitlab-ci.yml
│ └ release.gitlab-ci.yml
├ .gitlab-ci.yml
├ ChangelogGenerator.py
└ (changelog.md)
```

### `changelog` ステージの内容
```yaml:/.gitlab/changelog.gitlab-ci.yml
changelog:
  image: python:3.11
  stage: changelog
  before_script:
    - pip install python-gitlab
  script:
    - echo "$(python ChangelogGenerator.py)" > changelog.md
  artifacts:
    paths:
      - changelog.md
```
Pythonを使用するため、`image` は `python:3.11` としています。GitLab API は `urllib` モジュールや `request` モジュールから利用することもできますが、[`python-gitlab`](https://python-gitlab.readthedocs.io/en/stable/) モジュールを使うとより簡潔に書けるため、`before-script` でインストールしておきます。

`ChangelogGenerator.py` （後述）の出力を `changelog.md` としてファイル化し、ファイルを `release` ステージに引き継ぐため `artifacts` に登録します。

### `release` ステージの内容
```yaml:/.gitlab/release.gitlab-ci.yml
release:
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  stage: release
  needs: ["changelog"]
  before_script:
    - echo start release
  script:
    - CHANGELOG="$(cat changelog.md)"
    - release-cli create --name "Release $CI_COMMIT_TAG" --tag-name $CI_COMMIT_TAG --description="$CHANGELOG" --ref "$CI_COMMIT_SHA"
  only:
    - tags
```
GitLab Release CLI tool を使用するため、`image`は`registry.gitlab.com/gitlab-org/release-cli:latest`としています。`changelog` ステージで作成されたファイルに依存するため `needs` に `changelog` を指定しておきます。

変数 `CHANGELOG` に先ほど作成した `changelog.md` の内容を読み出しておき、`release-cli create` コマンドでリリースノートを作成する際に `description` オプションへ渡すことで、リリースノートに反映させます。`release-cli`の詳しい使い方については今回は割愛させていただきます。  
また、`only:` を指定することで動作するタイミングを「タグ付きのコミットがプッシュされたとき」のみに限定しています。

作成されたリリースノートは、GitLab の Web UI 上から `Deploy`->`Releases` を見れば確認できます。

![release_note_navigation]("/images/4-changelog-auto-generation-in-gitlab/GitLabRelease.png")

実際に作成されたリリースノートの例が以下のとおりです。前半は元から用意されていた HACARUS Check の配布部分で、今回追加した changelog は「20230728_QA - 2023-07-28」と書かれた以降の部分です。  
前回リリース以降にマージされたマージリクエストのタイトル（画像ではモザイクで隠されています）とIDがリスト化されており、IDはマージリクエストページへのリンクになっています。  
詳細が気になるものがあれば、リンク先へジャンプすることで簡単に確認できます。

![release_note_example]("/images
4-changelog-auto-generation-in-gitlab
ReleaseNote_sample_filtered.png")

補足として、リリースノートの前半部分を作成するには `changelog` と `release` 以外のステージも必要ですが、今回は changelog 作成の記事のため、省略しています。

## Python を使った changelog の作成
Hacarus Check は主に C# で開発されていますが、changelog の作成には個人的に使い慣れた python を使うことにしました。
CI では image によって環境を簡単に用意できるため、ちょっとした用途のために別言語を導入しても大した手間がかからないのは嬉しいポイントです。

changelog を作成するにはリポジトリの情報を得る必要があり、今回は前述の `python-gitlab` モジュール経由で GitLab API を利用します。  
なお GitLab API には標準の changelog 生成機能があるため、それを利用できれば python スクリプトを書くまでもなかったのですが、利用可能な条件に合わず断念しました。

:::details GitLab 標準の changelog 生成機能について
GitLab には CLI ツールや REST API が用意されており、それらを通じて標準の [changelog 生成機能](https://docs.gitlab.com/ee/user/project/changelogs.html)を利用できます。ただしこの機能を利用するにはいくつかの条件があります。

条件の1つ目として、changelog に含めたいコミットのコミットメッセージに [trailer](https://git-scm.com/docs/git-interpret-trailers) と呼ばれるフッターを適切に付与しておく必要があります。具体的には
```
（任意のコミットメッセージ）

Changelog: feature
```
というような形で、[こちら](https://docs.gitlab.com/ee/development/changelog.html)に記載された値（`added`, `fixed`, `changed`, `deprecated`, `removed`, `security`, `performance`, `other`）から適当なものを選び、メッセージの末尾に追加します。デフォルトでは`Changelog:`の trailer が対象となりますが、changelog 生成時に別の trailer を対象とするよう設定もできます。

条件の2つ目として、[semantic versioning](https://semver.org/) に準拠したタグをコミットに付与しておく必要があります。
タグの書式はある程度カスタマイズできますが、`major`, `minor`, `patch` に対応する数値が含まれていなければなりません。これらの数値に基づき、changelog 生成時に自動的に前のバージョンを特定したり、オプションでの範囲指定を可能にしているようです。

「これからリポジトリやコミットメッセージ規約を作る」という場合はこの機能を利用できるかもしれませんが、今回は既に開発が進んでいるリポジトリに機能を追加するケースで条件に合わなかったため、他の手段を検討しました。
:::

自前で changelog を生成するにあたり、changelog の内容を決めておく必要があります。  
今回は次のような書式で、「前回のタグ付けされたコミット以降にマージされたマージリクエスト（以下、MR と略記）一覧」を作成し、changelog とすることにしました。

```
# {タグ名} - {更新年月日}

- {MR1のタイトル} from {MR1へのリンク}
- {MR2のタイトル} from {MR2へのリンク}
- ・・・
```

実際に作成した `ChangelogGenerator.py` の内容は以下のとおりです。

```python: ChangelogGenerator.py
import datetime
import gitlab
from os import getenv


URL = "GitLabのURL"
TOKEN = {"private_token": getenv("GITLAB_TOKEN")}
PROJECT_ID = getenv("CI_PROJECT_ID")

def generate_changelog_text():
    gl = gitlab.Gitlab(URL, **TOKEN)
    project = gl.projects.get(id=PROJECT_ID)
    tags = project.tags.list(get_all=True)
    newest_tag = tags[0]
    second_newest_tag = tags[1]

    compare = project.repository_compare(second_newest_tag.name, newest_tag.name)
    commits_since_last_tag = compare["commits"]
    related_merged_requests = {}
    for commit in commits_since_last_tag:
        merge_requests = project.commits.get(commit["id"]).merge_requests()
        if merge_requests is not None:
            for mr in merge_requests:
                if mr["state"] == "merged" and mr["id"] not in related_merged_requests:
                    related_merged_requests[mr["id"]] = mr

    if len(related_merged_requests) == 0:
        merged_log = "- No merge requests are merged since last tagged commit."
    else:
        logs = []
        for mr in related_merged_requests.values():
            logs.append(f"- {mr['title']} from [!{mr['iid']}]({mr['web_url']})")
        merged_log = "\n".join(logs)

    date = datetime.datetime.fromisoformat(newest_tag.commit["committed_date"]).date()
    header = f"## {newest_tag.name} - {date}\n\n"
    text = f"{header}{merged_log}"
    print(text)


if __name__ == "__main__":
    generate_changelog_text()
```

上から順に、要点を補足していきます。

```python
import datetime
import gitlab
from os import getenv
```
`python-gitlab` の import 文は`import gitlab` となります。  

```python
URL = "GitLabのURL"
TOKEN = {"private_token": getenv("GITLAB_TOKEN")}
PROJECT_ID = getenv("CI_PROJECT_ID")
```
URLはダミー文字列として `GitLabのURL` にしていますが、通常のSaaS版なら`https://gitlab.com/`、Self-Managed版なら各自が設定したURLとなります。  
CI に環境変数 `GITLAB_TOKEN` として設定しておいたプライベートアクセストークンを取得し、API リクエスト時に使うように設定しています。  
GitLab CI/CD には [job token](https://docs.gitlab.com/ee/ci/jobs/ci_job_token.html)というものがあり、通常のプライベートアクセストークンよりも安全に使用できるそうですが、今回の用途ではエラーが出て使用できませんでした。  
また、プライベートアクセストークンを発行する際、`read_repository`では権限不足のため、`read_api`で発行する必要があります。ご注意ください。

`CI_PROJECT_ID` は CI 使用時に自動で設定される環境変数で、CI を動かしているプロジェクトの ID が格納されています。

```python
def generate_changelog_text():
    gl = gitlab.Gitlab(URL, **TOKEN)
```
`gitlab.Gitlab` クラスにアクセス先の URL と アクセストークンなどの認証情報を渡してインスタンス化することで、プライベートリポジトリへの API リクエストができるようになります。

```python
    project = gl.projects.get(id=PROJECT_ID)
    tags = project.tags.list(get_all=True)
    newest_tag = tags[0]
    second_newest_tag = tags[1]

    compare = project.repository_compare(second_newest_tag.name, newest_tag.name)
```
プロジェクトを指定し、全てのタグを取得します。  
さらに、最新のタグと1つ前のタグで比較した結果を取得します。

```python
    commits_since_last_tag = compare["commits"]
    related_merged_requests = {}
    for commit in commits_since_last_tag:
        merge_requests = project.commits.get(commit["id"]).merge_requests()
        if merge_requests is not None:
            for mr in merge_requests:
                if mr["state"] == "merged" and mr["id"] not in related_merged_requests:
                    related_merged_requests[mr["id"]] = mr
```
比較結果から、タグ間の全てのコミットについて関連する MR があるかを問い合わせ、マージ済みの MR があれば重複を避けて辞書に格納していきます。  
なお Hacarus Check リポジトリでは develop ブランチでのみタグをつけてリリースを行うため、この実装で develop ブランチのタグ間の MR が取得されますが、他のブランチでもタグがつけられる可能性がある場合は対応が必要かもしれません。

```python
    if len(related_merged_requests) == 0:
        merged_log = "- No merge requests are merged since last tagged commit."
    else:
        logs = []
        for mr in related_merged_requests.values():
            logs.append(f"- {mr['title']} from [!{mr['iid']}]({mr['web_url']})")
        merged_log = "\n".join(logs)
```
タグ間にマージされた MR が無ければ無い旨をメッセージとし、MR があれば書式に合わせてメッセージを作成します。

```python
    date = datetime.datetime.fromisoformat(newest_tag.commit["committed_date"]).date()
    header = f"## {newest_tag.name} - {date}\n\n"
    text = f"{header}{merged_log}"
    print(text)
```
最後に書式に合わせたヘッダーを付与して changelog の内容を完成させ、標準出力として出力します。

ここで changelog を直接ファイルとして書き込んでもよいですが、デバッグが楽になること、ファイル名が yaml 上で確認しやすいこと、ファイルではなく変数への格納やパイプでの受け渡しなどにも対応できることから、敢えて標準出力を選んでいます。  
（変数への格納は一度試しましたが、改行を含んでいるため扱いが難しそうでした。）

以上のような python スクリプトを実行することで、CI 上で changelog を自動作成できます。

## まとめ
GitLab CI/CD を利用して、変更履歴の記載を自動化しました。  
Python から GitLab API を利用することで、プロジェクトやリポジトリの様々な情報を取得して適当な書式に整えることが簡単にできます。ぜひ試してみてください。