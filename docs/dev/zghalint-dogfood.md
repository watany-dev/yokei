# zghalint v0.0.1-rc.1 dogfood — chokkin

chokkin の GitHub Actions / Dependabot 設定に [zghalint](https://github.com/watany-dev/zghalint)
`v0.0.1-rc.1` をかけ、検出結果を真陽性 / 誤検知 / 意図的抑制に分類した記録。

- 日付: 2026-09-09
- 対象リポジトリ: `watany-dev/chokkin` (`main` @ `616adbc`)
- zghalint: `v0.0.1-rc.1` (`zghalint v0.0.1-rc.1`)
- バイナリ: release ワークフロー run `34303774946` の
  `zghalint-linux-x86_64` artifact（commit `7600004458b07441b5a6c7d0b408dc676664c9da`）
- 走査対象: `.github/workflows/*.yml`、`.github/dependabot.yml`（4 ファイル）

## 判定

RC は **検出器としては動く**（exit 0、error 0、クラッシュなし）。ただし
SHA ピン止めを推奨する SEC001 と、SHA 参照を常に脆弱扱いする SC003 が衝突し、
このリポジトリでは warning 8 件のうち 4 件が既知のパッチ済みアクションへの誤検知だった。
GitHub Release に成果物が載っていないため、Action 経由の CI 導入はまだできない。

| 項目 | 結果 |
| --- | --- |
| バイナリ | `v0.0.1-rc.1`、静的リンク ELF x86_64 |
| GitHub Release 資産 | なし（`Create Release` が既存 tag で失敗） |
| ネットワーク規則 | `GITHUB_TOKEN` なし。stderr: `SC005, SC008 skipped (github api unreachable)` |
| 診断（JSON） | error 0 / warning 8 / info 24 / 合計 32（`--quick` は 25、差分は SC005 の 7 件） |
| 診断（terminal） | warning 8 / info 17 / 合計 25。SC005 は JSON にだけ入り、terminal には出ない |
| クラッシュ | なし |
| exit code | 0（error が無いため。warning では落ちない） |

## 検出サマリ（修正前）

| rule | severity | 件数 | 判定 | 対応 |
| --- | --- | ---: | --- | --- |
| BP002 | info | 13 | スタイル（許容） | 変更しない。`run:` 1 行に毎回 `name:` を付ける価値は低い |
| SC005 | info | 7 | **zghalint FP** | annotated tag の peel 失敗を `no_tag` に倒している |
| SC003 | warning | 4 | **zghalint FP** | SHA ピンはバージョン判定不能として常に warn |
| DEP001 | info | 3 | TP | `cooldown.default-days: 7` を追加 |
| PERF003 | warning | 2 | 意図的 | 行列の全 OS / 全 wheel 結果を残すため `fail-fast: false` |
| BP001 | warning | 1 | TP | `zizmor` job に `timeout-minutes: 10` を追加 |
| PERF001 | warning | 1 | 意図的 | `pip install --no-cache-dir` で Semgrep を 1 パッケージ入れるだけ |
| PERM001 | info | 1 | 意図的 | `gh release create` に `contents: write` が必要 |

`--quick` では SC005 が JSON からも消える以外は同一。埋め込み advisory の SC003 はオフラインでも発火する。

stderr は `SC005, SC008 skipped` と出すが、JSON には SC005 が残る。同じ実行の terminal 出力は SC005 を出さない。CI が `--format json` を読むと、人間が見る terminal より 7 件多い。

## 分類の根拠

### SC003（4 warning）— SHA ピン × 過去 advisory の衝突

| uses | コメント | GHSA | patched | 実際のバージョン |
| --- | --- | --- | --- | --- |
| `github/codeql-action/upload-sarif@e4fba868…` | v4.37.3 | GHSA-vqf5-2xx6-9wfm (`>= 3.26.11, <= 3.28.2` と `>= 2.26.11, < 3.0.0`) | 3.28.3 | v4.37.3（範囲外） |
| `actions/download-artifact@3e5f45b2…` | v8.0.1 | GHSA-cxww-7g56-2vh6 (`>= 4.0.0, < 4.1.3`) | 4.1.3 | v8.0.1（軽量タグ、範囲外） |
| 同上（`publish-pypi` job） | v8.0.1 | 同上 | 4.1.3 | 同上 |
| `pypa/gh-action-pypi-publish@ba38be9e…` | v1.14.1 | GHSA-vxmw-7h4f-hqxh (`< 1.13.0`) | 1.13.0 | v1.14.1（範囲外） |

`src/rules/advisory.zig` はタグ参照だけ `vulnerable_range` を見る。SHA 参照は
コメントの `# v4.37.3` も読まず、そのアクションに advisory が 1 件でもあれば
warn する（テスト名 `SC003: SHA ref with vulnerable action still warns`）。
SEC001 が SHA ピンを求める以上、パッチ済みの現行ピンがすべて SC003 になる。
hint の「update to 4.1.3 or later」は既に v8.0.1 を使っている呼び出しには役に立たない。

### SC005（7 info）— annotated tag を untagged 扱い

7 件はすべて `Swatinem/rust-cache@c19371144… # v2.9.1`。
`v2.9.1` は annotated tag で、peel 先の commit がまさにこの SHA である。

`src/rules/rest_fallback.zig` の `matchShasInRefs` は annotated tag の
dereference が失敗すると `continue` し、タグ数が 100 未満なら残りの SHA を
`no_tag` にする。今回は GitHub API が zghalint の HTTP クライアントから到達できず
（stderr の skip 注記）、peel が全部落ちた結果、本物の tagged SHA が SC005 になった。

対照的に `actions/checkout@v7.0.1` などは軽量タグ（`object.type: commit`）なので
listing だけで `has_tag` になり、指摘されない。`github/codeql-action` のように
タグが 100 件を超えるリポジトリは早期 return で `unknown` のまま沈黙する
（stale SHA の取りこぼし側）。

### 許容した指摘

- **BP002**: `run: cargo fmt --check` のような単一コマンドに `name:` を要求する。可読性は上がるが、このリポジトリの既存スタイルと合わない。
- **PERF003**: `ci.yml` の OS 行列と `release.yml` の wheel 行列は、1 件失敗で残りを殺すとプラットフォーム固有の失敗を見落とす。
- **PERF001**: Semgrep job は lockfile なしで `pip install --no-cache-dir semgrep==1.166.0` している。`cache: pip` は dependency file が無く、setup-python が落ちうる。
- **PERM001**: `release` job の `contents: write` は `gh release create` に必要。read では足りない。

## このリポジトリで直したもの

- `.github/workflows/ci.yml` — `zizmor` job に `timeout-minutes: 10`（BP001）
- `.github/dependabot.yml` — 3 ecosystem に `cooldown.default-days: 7`（DEP001）

## zghalint RC へのフィードバック

優先度高い順:

1. **SC003 は SHA ピンを「常に脆弱」にしない。** `# vX.Y.Z` コメント、または SC005 が解決した tag を semver 判定に使う。取れないときは warning ではなく info / 沈黙（`--quick` の「調べられなかった」側）に倒す。
2. **SC005 は annotated tag の peel 失敗を `no_tag` にしない。** 既存コメントどおり `unknown` なら無指摘。今回の rust-cache 7 件がそれ。
3. **GitHub Release 資産。** tag `v0.0.1-rc.1` の Release は空で、release workflow は
   `a release with the same tag name already exists` で失敗した。
   `uses: watany-dev/zghalint@v0.0.1-rc.1` はアーカイブを Releases から取るため、
   このままだと Action 導入できない。既存 Release へ `gh release upload` するか、
   ワークフローを `release edit` / skip-if-exists にする。
4. **SC003 hint の patched version** は、呼び出しが既にそれより新しい major にいると逆効果（v8 に対して「4.1.3 以降へ」）。
5. **ネットワーク到達不能の注記は JSON と矛盾する。** stderr は SC005 を skipped と出すが、JSON には rust-cache の `no_tag` が 7 件入る。terminal には出ない。`--format json` を CI に載せる前に揃える。

## CI 導入

`docs/dev/ci-porting-notes.md` に延期理由を書いた。資産が公開されたら
`actionlint` / `zizmor` の隣に `watany-dev/zghalint@<sha> # v0.0.1-rc.1` を足せる。
そのときは BP002 / PERF003 / PERF001 / PERM001 / SC003（修正前）を
`.zghalint.yml` で明示的に残すか無効化するかを決める。

## 実行コマンド

```bash
zghalint --version   # zghalint v0.0.1-rc.1
zghalint --format json
zghalint --quick --format json
```

修正後の再実行（実測）:

| 出力 | errors | warnings | infos | total |
| --- | ---: | ---: | ---: | ---: |
| JSON | 0 | 7 | 21 | 28 |
| terminal | 0 | 7 | 14 | 21 |
| `--quick` JSON | 0 | 7 | 14 | 21 |

残った warning は SC003 4 + PERF003 2 + PERF001 1。DEP001 と BP001 は消えた。
