---
title: "[生成AI利用] GITHUB_TOKEN で動かした検査の「次」が起動せず、auto-merge が止まった話"
emoji: "🔁"
type: "tech"
topics: ["githubactions", "github", "cicd", "gitleaks"]
published: true
---

## はじめに

GitHub Actions で作った自動マージが、**検査は緑なのに `main` に入らない**状態で止まった。エラーは出ない。

原因は `GITHUB_TOKEN` の仕様だった。本記事では、止まった仕組みと、`GITHUB_TOKEN` のまま直した方法をまとめる。

## 何が起きたか

個人のリポジトリで、作業ブランチを自動で `main` に取り込む仕組みを GitHub Actions で作っていた。流れは次のとおり。

1. 作業ブランチに push する
2. `on: push` で秘密情報の検査（gitleaks）が走る
3. 検査が終わると `on: workflow_run` で auto-merge が起動する
4. `main` を fast-forward で進める

`main` はルールセットで検査の成功を必須にしているので、検査済みのコミットしか入らない。

問題は `main` が先に進んでいたときだ。fast-forward できないので、auto-merge が `main` を取り込んだコミットを `auto/<ブランチ>` に作り、検査してから `main` に入れようとした。

ここで止まった。

## 原因

GitHub のドキュメントに、そのまま書いてあった。`GITHUB_TOKEN` によるイベントでは、基本的に新しいワークフローの実行が作られない。再帰的な実行を防ぐためだ。例外は `workflow_dispatch` と `repository_dispatch`。

今回はこれに2回当たっていた。

| 段階 | 動き |
|---|---|
| `auto/` ブランチへの push（`GITHUB_TOKEN`） | `on: push` の検査が起動しない |
| `gh workflow run` で検査を起動 | `workflow_dispatch` は例外なので起動する |
| その検査が終わる | `on: workflow_run` の auto-merge が起動しない ← ここで止まる |

1つ目には気づいて `workflow_dispatch` で回避していたが、2つ目を見落としていた。毎朝ブランチを見直すワークフローも同じ理由で、検査をかけ直すだけで取り込みまで進まなかった。

## 直し方：起動を諦めて、同じジョブの中で待つ

次のワークフローに引き継ぐのをやめて、検査を起動したジョブがそのまま結果を待ち、自分で `main` に入れるようにした。

```bash
# 検査を起動する（workflow_dispatch なので GITHUB_TOKEN でも動く）
gh workflow run gitleaks.yml --ref "$REF"

# 起動した実行を探す（一覧に出るまで少しかかる）
run_id=""
for _ in $(seq 1 24); do
  sleep 5
  run_id=$(gh run list --workflow gitleaks.yml --branch "$REF" \
    --event workflow_dispatch --limit 10 \
    --json databaseId,headSha \
    --jq "map(select(.headSha == \"$SHA\")) | .[0].databaseId // empty")
  [ -n "$run_id" ] && break
done
[ -n "$run_id" ] || exit 1

# 終わるまで待つ。失敗なら止まる
gh run watch "$run_id" --exit-status --interval 10
```

ポイントは3つ。

- **コミットで実行を見分ける：** 同じブランチの別の実行を拾わないように、`headSha` で絞る
- **権限を足す：** `gh workflow run` には `actions: write` が要る
- **待っているあいだの変化に備える：** 検査中にまた `main` が進むことがあるので、数回やり直して、それでもだめなら止める

## ほかの直し方

ドキュメントが勧めているのは、`GITHUB_TOKEN` の代わりに GitHub App のトークンや個人用アクセストークン（PAT）を使う方法だ。これなら次のワークフローも起動する。

今回はそうせず、待つ方法を選んだ。長く使う秘密を増やさなくて済むからだ。代わりに、検査のあいだジョブが動き続けて Actions の時間を使う。秘密を増やしてもいいなら、そちらのほうが素直だと思う。

## 確かめ方

後から取り込むブランチが `auto/` を通るように、2本のブランチを同時に push して試した。検査を待って `main` に入り、ブランチも削除された。

## まとめ

- `GITHUB_TOKEN` で起こしたことからは、次のワークフローは基本的に起動しない
- 例外の `workflow_dispatch` で検査を起動できても、その完了を `workflow_run` で受けられるとは限らない
- 緑なのに進まないときは、ここを疑う

## 参考文献

- [GITHUB_TOKEN - GitHub Docs](https://docs.github.com/en/actions/concepts/security/github_token)
- [Triggering a workflow - GitHub Docs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow)

---

:::message
この記事は生成AI（Claude）を使用して作成されました。内容の正確性には注意を払っていますが、誤りがある可能性があります。お気づきの点があれば、コメントで教えてください。
:::
