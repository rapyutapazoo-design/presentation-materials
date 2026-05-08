---
title: "fix: scratch.py 個人情報を git 履歴から完全削除"
type: fix
status: active
date: 2026-05-08
---

# fix: scratch.py 個人情報を git 履歴から完全削除

## Summary

公開リポジトリ `tablet-comparison-proposals` の git 履歴に、個人のローカルパス（`/Users/hiroki/...`）を含む開発用スクリプト `scratch.py` が混入している。`git filter-repo` で該当コミット（`f990345`）を履歴ごと書き換え、GitHub・Netlify の両経路からアクセス不能にする。あわせてローカルの不要ファイル（テスト用フォルダ）を整理し、親リポジトリのサブモジュール参照を最新化する。

---

## Problem Frame

`scratch.py` はタブレット比較 HTML の機種変更を自動化するために一時的に作成された開発用 Python スクリプトであり、`/Users/hiroki/Library/Mobile Documents/com~apple~CloudDocs/...` というユーザー名・iCloud パスを含む。このファイルが公開リポジトリの最新コミット（GitHub master の HEAD: `f990345`）に含まれており、以下の 3 経路から誰でもアクセス可能な状態にある。

| 経路 | URL | 現状 |
|------|-----|------|
| Netlify | `fascinating-bunny-f65218.netlify.app/…/scratch.py` | HTTP 200（公開中） |
| GitHub UI | `github.com/…/blob/master/scratch.py` | 全文表示（公開中） |
| GitHub Raw（履歴） | `raw.githubusercontent.com/…/f990345/scratch.py` | HTTP 200（公開中） |

通常の `git rm` + commit では経路③は永続するため、履歴の書き換えが必須。

---

## Requirements

- R1. `scratch.py` が Netlify URL 経由でアクセス不能になること（HTTP 404）
- R2. `scratch.py` が GitHub UI（master ブランチ）から消えること
- R3. `scratch.py` を含むコミット（`f990345`）が git 履歴から抹消され、Raw URL 経由もアクセス不能になること
- R4. `presentation.html` の内容は毀損しないこと（`f990345` で行われた機種変更の内容は維持）
- R5. 親リポジトリ（presentation-materials）のサブモジュール参照が正しいコミットを指すこと
- R6. Netlify が最新の状態で再デプロイされること
- R7. ローカルの追跡されていないテスト用ファイルが整理されること

---

## Scope Boundaries

- `mansion-digital-roadmap` リポジトリは今回の対象外（個人情報含むファイルなし）
- `presentation.html` の内容変更は行わない
- GitHub リポジトリの可視性（Public → Private 化）は今回行わない
- Netlify のサイト設定・ドメイン変更は行わない
- `assets/` フォルダ内の画像ファイルの扱いは今回対象外

### Deferred to Follow-Up Work

- サブモジュールのローカル版とGitHub版の内容差異（台数 1台 vs 2台）の整合：別途コンテンツ更新作業として実施
- 全リポジトリの Public → Private 化の検討：別途判断

---

## Context & Research

### コミット構造（tablet-comparison-proposals）

```
f990345  fix: align tablet rankings and update NEC T12N price  ← scratch.py あり（GitHub HEAD）
ef18c12  fix: replace Apple iPad with NEC Lavie ...            ← scratch.py なし
b12702a  タブレット運用台数を2台から1台に変更                    ← scratch.py なし
2504ba6  Completely remove residual duplicated cards ...       ← scratch.py なし（ローカル HEAD）
ff6a97c  Fix markup corruption in tab 3 and tab 4 ...
...（以下 scratch.py なし）
52738b3  Initial commit: Tablet comparison materials
```

**重要な事実：** `scratch.py` が存在するコミットは `f990345` の 1 つのみ。それ以前・以後のコミットには存在しない。`git filter-repo` はこの 1 コミットだけを書き換え、祖先コミットのハッシュは変化しない。

### サブモジュール参照の現状

| | 参照コミット |
|---|---|
| 親リポジトリ（Netlify デプロイ基準） | `f990345`（旧） |
| ローカルサブモジュール HEAD | `2504ba6`（`f990345` の 3 コミット前） |

### GitHub キャッシュの挙動

- force push 後、`f990345` は branch から unreachable になる
- ただし GitHub の git object cache は GC までの間（最大 90 日程度）残存する可能性がある
- GitHub Support への依頼でキャッシュの強制削除が可能

### External References

- [GitHub: 機密データをリポジトリから削除する](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [git-filter-repo 公式](https://github.com/newren/git-filter-repo)

---

## Key Technical Decisions

- **git filter-repo を使用**（BFG Repo-Cleaner でも可）：`f990345` の 1 コミットのみに `scratch.py` が存在するため、ファイル指定での絞り込みが確実。`git filter-branch` は非推奨のため不使用。
- **フレッシュクローンで実行**：filter-repo は既存ワーキングコピーへの実行を非推奨。別ディレクトリへのクローンで安全に実施する。
- **force push は master ブランチのみ**：他ブランチは存在しないため影響なし。
- **親リポジトリの更新先は filter-repo 後の新 HEAD**：`f990345` は書き換え後に新ハッシュになるため、そのハッシュを親リポジトリに記録する。
- **テスト用ファイルはローカル削除のみ**：git 未追跡のため、単純にフォルダごと削除する。

---

## Open Questions

### Resolved During Planning

- **Q: `f990345` の `presentation.html` の内容は保持できるか？**
  → filter-repo は `scratch.py` のみを除外し `presentation.html` はそのまま保持する。`f990345` で加えた機種変更の内容は失われない。

- **Q: ローカルの `2504ba6` はどうするか？**
  → `f990345` の書き換え後に生まれる新 HEAD と `2504ba6` は別ライン。親リポジトリは filter-repo 後の新 HEAD を指せばよく、`2504ba6` を push する必要はない（コンテンツ整合は別タスク）。

### Deferred to Implementation

- **filter-repo 実行後の新 HEAD ハッシュ**：実行後に確認する。
- **GitHub Support へのキャッシュ削除依頼の所要時間**：Support の対応速度に依存。

---

## High-Level Technical Design

> *これは意図するアプローチを示す方向性のガイドであり、実装仕様ではありません。*

```
【現在の状態】
GitHub master:  52738b3 → ... → 2504ba6 → b12702a → ef18c12 → f990345(HEAD)
                                                                  └─ scratch.py ←問題

【filter-repo 後】
GitHub master:  52738b3 → ... → 2504ba6 → b12702a → ef18c12 → f990345_new(HEAD)
                                                                  └─ presentation.html のみ
                （f990345_old は orphan → GC 待ち）

【親リポジトリ更新後】
presentation-materials:
  submodule ref: f990345_old → f990345_new に更新
  → Netlify 自動デプロイ → scratch.py が 404 に
```

---

## Implementation Units

- U1. **git-filter-repo のインストール**

**Goal:** filter-repo ツールを実行環境に準備する

**Requirements:** R1, R2, R3

**Dependencies:** なし

**Files:**
- 変更なし（ツールのインストールのみ）

**Approach:**
- Homebrew 経由でインストール（`brew install git-filter-repo`）が最も簡単
- Homebrew が使えない場合は pip 経由（`pip3 install git-filter-repo`）
- インストール後 `git filter-repo --version` で動作確認

**Test scenarios:**
- `git filter-repo --version` が正常にバージョン番号を返すこと

**Verification:**
- コマンドが利用可能な状態になっていること

---

- U2. **tablet-comparison-proposals をフレッシュクローンして scratch.py を履歴から削除・force push**

**Goal:** `scratch.py` を含むコミット `f990345` を書き換えて GitHub から完全に排除する

**Requirements:** R1, R2, R3, R4

**Dependencies:** U1

**Files:**
- 作業ディレクトリ（一時クローン）：`/tmp/tablet-comparison-work/` など任意の場所
- 変更対象リポジトリ（リモート）: `https://github.com/rapyutapazoo-design/tablet-comparison-proposals`

**Approach:**
1. **フレッシュクローン**：一時ディレクトリに `git clone` する（`--no-local` でローカルの submodule とは別のクローンを作成）
2. **filter-repo 実行**：`git filter-repo --path scratch.py --invert-paths` を実行。これにより `f990345` から `scratch.py` が除去され、新ハッシュのコミットに書き換わる
3. **新 HEAD ハッシュを記録**：filter-repo 後の `git log --oneline -1` で確認・メモする
4. **force push**：`git push origin master --force` を実行
5. **検証**：GitHub UI で master ブランチのファイルが `presentation.html` のみになっていること、`scratch.py` が表示されないことを確認

**Test scenarios:**
- Happy path: `https://github.com/…/blob/master/scratch.py` にアクセスして 404 / "Not Found" になること
- Happy path: `https://raw.githubusercontent.com/…/master/scratch.py` が HTTP 404 を返すこと
- 保護確認: `presentation.html` の内容（NEC LAVIE Tab T12N の機種情報など）が失われていないこと
- 履歴確認: `git log --all --full-history -- scratch.py` が何も返さないこと（一時クローンで確認）

**Verification:**
- GitHub の master ブランチに `scratch.py` が存在しない
- `presentation.html` の差分が生じていない

---

- U3. **親リポジトリのサブモジュール参照を更新・プッシュ**

**Goal:** 親リポジトリ（presentation-materials）が filter-repo 後の新しい tablet-comparison-proposals の HEAD コミットを参照するよう更新し、Netlify に新デプロイをトリガーする

**Requirements:** R1, R5, R6

**Dependencies:** U2

**Files:**
- Modify: `理事会向け/管理室用タブレット比較`（サブモジュール参照ポインタ）

**Approach:**
1. 親リポジトリのルートで `git submodule update --remote 理事会向け/管理室用タブレット比較` を実行（または `cd` して `git checkout` で U2 で記録した新 HEAD にセット）
2. `git status` で `M 理事会向け/管理室用タブレット比較` が消えていることを確認
3. `git add 理事会向け/管理室用タブレット比較` でサブモジュール参照をステージング
4. `git commit -m "fix: update tablet-comparison submodule to remove scratch.py"` でコミット
5. `git push origin master` で親リポジトリを push
6. Netlify の管理画面またはデプロイログで自動デプロイが開始されたことを確認

**Test scenarios:**
- Happy path: Netlify デプロイ完了後、`https://fascinating-bunny-f65218.netlify.app/理事会向け/管理室用タブレット比較/scratch.py` が HTTP 404 を返すこと
- Happy path: `https://fascinating-bunny-f65218.netlify.app/理事会向け/管理室用タブレット比較/presentation.html` が正常に表示されること（HTTP 200）
- Happy path: ポータル（`/`）、ロードマップ（`/理事会向け/デジタル化ロードマップ/index.html`）が引き続き正常に表示されること

**Verification:**
- 親リポジトリの `git log --oneline -1` に更新コミットが存在する
- Netlify のデプロイが成功している
- Netlify URL 経由で `scratch.py` が 404 になっている

---

- U4. **ローカルのテスト用ファイルを削除**

**Goal:** git 未追跡のテスト用フォルダ（`テスト用ファイル/`）をローカルから削除してディレクトリを整理する

**Requirements:** R7

**Dependencies:** なし（U1〜U3 と並行実施可能）

**Files:**
- Delete: `テスト用ファイル/portal.html`
- Delete: `テスト用ファイル/presentation.html`
- Delete: `テスト用ファイル/roadmap.html`
- Delete: `テスト用ファイル/assets/`（5枚のPNG、合計約31MB）
- Delete: `テスト用ファイル/`（フォルダごと）

**Approach:**
1. `rm -rf テスト用ファイル/` でフォルダごと削除
2. `git status` で `?? テスト用ファイル/` の表示が消えていることを確認
3. git 未追跡ファイルのため、追加の git 操作は不要

**Test scenarios:**
- `ls テスト用ファイル/` がディレクトリなし（エラー）になること
- `git status` の untracked files に `テスト用ファイル/` が表示されないこと

**Verification:**
- フォルダが存在しない
- git の状態に変化なし（未追跡だったため）

---

- U5. **GitHub Support へキャッシュ削除を依頼**

**Goal:** force push 後も GitHub の git object cache に残存する可能性がある `f990345` の古いオブジェクトを GitHub 側で強制削除してもらい、Raw URL 経由のアクセスを確実に封じる

**Requirements:** R3

**Dependencies:** U2

**Files:**
- 変更なし（GitHub Support への連絡のみ）

**Approach:**
1. GitHub Support へコンタクト：[https://support.github.com/contact](https://support.github.com/contact)
2. 依頼内容のポイント：
   - リポジトリ URL: `https://github.com/rapyutapazoo-design/tablet-comparison-proposals`
   - 問題のコミットハッシュ: `f990345`（filter-repo で書き換え済み）
   - 削除対象ファイル: `scratch.py`（個人のローカルパスを含む機密データ）
   - 参照ドキュメント: "Removing sensitive data from a repository"
3. GitHub Support は通常、sensitive data removal の依頼を受け付け、git object cache のパージを実施する
4. 対応完了後、`https://raw.githubusercontent.com/rapyutapazoo-design/tablet-comparison-proposals/f990345/scratch.py` が HTTP 404 になることを確認

**Test scenarios:**
- 依頼後: `curl -s -o /dev/null -w "%{http_code}" "https://raw.githubusercontent.com/rapyutapazoo-design/tablet-comparison-proposals/f990345/scratch.py"` が `404` を返すこと

**Verification:**
- GitHub Support から対応完了の返信を受け取っている
- 旧コミットハッシュ経由の Raw URL が 404 になっている

---

## System-Wide Impact

- **Netlify デプロイ**: U3 の親リポジトリ push によって自動デプロイがトリガーされる。デプロイ中（数分間）は古いバージョンが表示される可能性があるが、コンテンツ的な問題はない
- **サブモジュール参照の整合**: force push 後、ローカルのサブモジュール（`理事会向け/管理室用タブレット比較`）は古い履歴を持つ状態になる。`git submodule update --remote` で同期が必要
- **ローカルのサブモジュールとGitHubの内容差異**:  ローカル HEAD（`2504ba6`）は GitHub の書き換え後 HEAD の祖先。内容の整合は別タスク（Deferred）
- **既存リンク**: ポータルからのリンクは `presentation.html` を指しており、`scratch.py` へのリンクは存在しないため影響なし
- **変更されない不変条件**: `presentation.html` の内容、ポータル・ロードマップのすべてのページは変更なし

---

## Risks & Dependencies

| リスク | 対策 |
|--------|------|
| filter-repo 実行ミスで `presentation.html` の内容が破損する | フレッシュクローンで実行し、force push 前に `git diff` で内容を確認する |
| force push 後に GitHub object cache が長期残存し Raw URL が生き続ける | U5 の GitHub Support 依頼で対処。Support 対応まで残存する期間は許容（branch から unreachable になっているため通常の検索やリンクからは到達不能） |
| GitHub の branch protection で force push がブロックされる | 事前に `Settings > Branches` で master の branch protection を確認し、一時的に解除する |
| Netlify のデプロイが失敗する | Netlify の管理画面でデプロイログを確認し、必要に応じて手動再デプロイをトリガー |
| 作業中にローカルとリモートの履歴がさらに乖離する | U2 完了後すみやかに U3 を実施する。作業中は他のコミット・プッシュを行わない |

---

## Documentation / Operational Notes

- 作業完了後、以下の URL が HTTP 404 を返すことを最終確認する：
  - `https://fascinating-bunny-f65218.netlify.app/理事会向け/管理室用タブレット比較/scratch.py`
  - `https://raw.githubusercontent.com/rapyutapazoo-design/tablet-comparison-proposals/master/scratch.py`
  - `https://raw.githubusercontent.com/rapyutapazoo-design/tablet-comparison-proposals/f990345/scratch.py`（U5 完了後）
- 将来の再発防止として、`.gitignore` に `*.py` または `scratch.*` を追記することを推奨（別タスク）

---

## Sources & References

- 調査資料（ローカル）: 本セッションの調査結果（2026-05-08）
- GitHub 公式ドキュメント: [機密データをリポジトリから削除する](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- git-filter-repo: [https://github.com/newren/git-filter-repo](https://github.com/newren/git-filter-repo)
- 対象リポジトリ: [https://github.com/rapyutapazoo-design/tablet-comparison-proposals](https://github.com/rapyutapazoo-design/tablet-comparison-proposals)
- Netlify サイト: [https://fascinating-bunny-f65218.netlify.app](https://fascinating-bunny-f65218.netlify.app)
