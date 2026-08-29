# manifest の追加

この bucket (`bucket/*.json`) に Scoop manifest を新規追加する手順。既存 manifest の更新は Excavator (`.github/workflows/schedule.yml`) が自動で行うため、この文書は新規追加のみを扱う。

## 調査

manifest を書く前に、上流について次を実測して確定する。学習データや推測で埋めない。

- リリース一覧: `gh release list -R <owner>/<repo> --limit 10`。全件が Pre-release かどうかを必ず見る (checkver の方式が変わる)。
- 最新リリースの詳細: `gh release view <tag> -R <owner>/<repo> --json tagName,assets,body,isPrerelease`。アセットのファイル名とタグの対応 (`$version` 展開の形) と、body に書かれた依存要件・注意事項。
- アセットの中身: zip をダウンロードして `unzip -l` で見て、トップレベルの階層と配置すべきファイルを確定する。
- ライセンスと説明: `gh api repos/<owner>/<repo> --jq '.license.spdx_id'` と `gh api repos/<owner>/<repo> --jq '.description'`。
- インストール手順: 上流 README のインストール節。GUI 経由の手順しか無い場合は、配置先を本体側の仕様 (下記「配置先の型」) から確定する。
- ハッシュ: ダウンロードした zip の `sha256sum`。

## 配置先の型

既存 manifest は型 1・2・4 のいずれかに分かれる。型 3 は既存に無く、AviUtl ExEdit2 向けに新規追加するときの設計方針として定める。新規追加ではどの型かを先に決める。

1. 単体アプリ: `url` / `hash` に加え、GUI アプリは `shortcuts` (`aviutl2.json`)、CLI ツールは `bin` (`mpd.json`) を書く。
2. TVTest / AviUtl1 のプラグイン: `depends` に `sudo` と本体を置き、`installer.script` で `$scoopdir\apps\<host>\current\...` へ `New-Item -ItemType SymbolicLink` で symlink を張り、`uninstaller.script` で `(Get-Item <link>).Delete()` で消す (`aviutl-exedit.json`)。古い manifest には必須プロパティや書式が CI の基準に合わないものがある (`tvtest-video-decoder.json` は `description` と `license` を欠く。`tvcaptionmod2.json` は `description` を欠き、インデントが 2 スペースで `formatjson.ps1` の 4 スペースと合わない)。そのまま雛形にすると PR 検証 CI が落ちる。
3. AviUtl ExEdit2 のプラグイン・スクリプト (既存に無い設計方針): 型 2 と同じ symlink 方式 (`depends` に `sudo` と `aviutl2` を置き、symlink 作成は `sudo New-Item -ItemType SymbolicLink` で行う) だが、張り先は本体の `$dir` ではなく `$env:ProgramData\aviutl2\{Plugin,Script,Alias,Language}`。AviUtl2 は `ProgramData\aviutl2` 配下を読み (`Plugin` と `Script` は一つ下のフォルダも対象)、exe と同じフォルダに `data` フォルダがある場合だけそちらを読む (aviutl2 配布物の `aviutl2.txt` 「ファイル配置」節)。bucket の `aviutl2.json` は `data` を作らないため `ProgramData` 側が対象になる。aviutl2 を一度も起動していないと `ProgramData\aviutl2` が無いので、symlink を張る前に `New-Item -ItemType Directory -Force` で対象ディレクトリを用意する。
   - `.au2pkg.zip` は AviUtl2 の GUI が解釈するパッケージ形式だが、実体はデータフォルダの相対配置 (`Plugin/`, `Script/`, `Alias/`, `Language/`) をそのまま zip にしたもの。Scoop の展開判定は末尾 `.zip` の一致なので通常の zip として展開される。公式のパッケージ機構 (メニューの「パッケージ情報」) は経由しないため、`notes` にその旨と scoop でアンインストールすることを書く。
4. フォント: `installer.script` で `$env:windir\Fonts` へコピーし `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts` にレジストリ登録する。`uninstaller.script` で両方を戻す (`cica.json`、`udev-gothic.json` 等)。既存のフォント manifest を写す。

共通事項:

- `depends` は「無ければ入れる」だけで、下限バージョンは指定できない。上流が要求するバージョンは `notes` に書く。
- `scoop uninstall` は `uninstaller.script` を実行してから `$dir` を削除するので、uninstaller から `$dir` 配下のファイル名を参照してよい。
- 利用者が生成するファイルが `$dir` 配下に置かれる場合は `persist` に挙げる (更新で消えるため)。

## checkver / autoupdate

- 上流に通常リリース (非 pre-release) がある場合: `"checkver": { "github": "<repo URL>" }`。タグに `v` 以外の接頭辞があれば `re` で拾う (`bondriver-mirakc.json`)。
- 全リリースが pre-release の場合: `checkver.github` は `<repo>/releases/latest` を固定で引き、pre-release を返さないため使えない。GitHub API の releases 配列を直接引く。

  ```json
  "checkver": {
      "url": "https://api.github.com/repos/<owner>/<repo>/releases",
      "jsonpath": "$[0].tag_name",
      "regex": "^v(.+)$"
  }
  ```

  `$[0]` は `created_at` 降順の先頭であってバージョン順ではない。上流が新しいリリースの後に旧系列のパッチを公開すると古いタグを拾い、Excavator (毎時 `-Update`) が manifest をバージョンダウンで書き換える。上流のリリース運用が単一系列であることを確認してから使う。

- `autoupdate.url` はアセット URL のバージョン部分を `$version` に置き換える。タグに `v` が付くなら `v$version` と書く。

## 必須プロパティと書式

- `description` と `license` は必ず入れる。PR 検証 CI (`.github/workflows/pull_request.yml`) が無いものを落とす。
- 書式は `scoop/bin/formatjson.ps1` の出力に合わせる (CI が lint する)。
- 4 バイト文字 (絵文字等) を manifest に書かない。非 ASCII のファイル名を扱う場合は manifest に直書きせず `Get-ChildItem` で列挙する。

## 検証

WSL の pwsh (`/usr/bin/pwsh`) で、リポジトリ直下から実行する。隔離 worktree では `scoop/` submodule が空なので、先に `git submodule update --init` を実行する。新規 manifest は untracked のため、検証前に `git add bucket/<name>.json` で index に載せてから `git diff` を見る (untracked のままだと diff が常に空になり、ツールによる書き換えを検出できない)。既存 manifest は改行コードが CRLF で、新規 manifest も CRLF で保存する。Linux の pwsh で `formatjson.ps1` や `checkver.ps1 -u` を実行すると末尾の改行が LF に書き換わり `git diff` に 1 hunk 出るので、その hunk は無視し、コミット前に CRLF へ戻す (`unix2dos` 等)。CI は Windows で `formatjson.ps1` を実行して比較するため、LF のままだと lint で落ちる (`.gitattributes` は未整備なので手で保つ)。

Windows 実機での `scoop install` / `scoop uninstall` は、WSL の interop 経由で Windows 側の Scoop を呼んで検証する (`cmd.exe /c "scoop --version"` で到達できることを先に確認する)。検証できなかった場合のみ、報告に「未検証」と明記する。

- Windows 側のコマンドは cwd を `/mnt/c/` 配下に置いて実行する。cwd が `\\wsl.localhost\...` だと cmd.exe が UNC 非対応の警告を出す。
- manifest を Windows 側のパス (例: `C:\Users\<user>\AppData\Local\Temp\<name>.json`) にコピーし、`cmd.exe /c "scoop install <その Windows パス>"` で入れる。ローカル json から入れた app は bucket に追従しないので、検証後は `scoop uninstall <name>` で戻し、マージ後に `scoop install ansanloms/<name>` で入れ直す。
- `sudo` が要る manifest は `cmd.exe /c "sudo scoop install ..."` で Windows 組み込みの `sudo.exe` が UAC を出す。人が UAC に応答するまで待つので `timeout` を長めに取る。非管理者で実行すると installer のガードで止まるが、Scoop 側に "Install failed" のエントリが残るため、`scoop uninstall <name>` で掃除してから昇格して入れ直す。
- レジストリの確認は `reg.exe query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts"` を WSL から直接呼ぶ。`cmd.exe /c 'reg query "..."'` で包むと `Windows NT` の空白で引用が壊れ、構文エラーになる。

```sh
pwsh -File ./scoop/bin/checkver.ps1 <name> -dir ./bucket -u -f   # 実行後の git diff が末尾改行の hunk だけなら version / url / hash が autoupdate と整合
pwsh -File ./scoop/bin/checkhashes.ps1 <name> -dir ./bucket
pwsh -File ./scoop/bin/formatjson.ps1 <name> -dir ./bucket       # 実行後の git diff が末尾改行の hunk だけなら書式が整合
pwsh -c "Test-Json -Json (Get-Content -Raw bucket/<name>.json) -SchemaFile scoop/schema.json"
```

`scoop/` は submodule で特定のコミットに固定されている。CI は最新の Scoop を使うので、ローカルの結果と食い違ったら CI の判定を優先する。

## PR

- PR 検証 CI は PR の `opened` イベントと、PR への `/verify` コメントでしか走らない。修正を push した後は `gh pr comment <番号> --body /verify` で再実行する。`/verify` で走るのは main にある `pull_request.yml` なので (`issue_comment` は default branch の workflow を使う)、`pull_request.yml` 自体を追加・変更する PR ではその変更は `/verify` に反映されない。
- 結果は PR コメントと `manifest-fix-needed` / `review-needed` ラベルで返る。
- fork からの PR では `opened` で検証もコメントも行われず、workflow は成功扱いで終わる (read-only token のため)。`/verify` は維持者 (OWNER / MEMBER / COLLABORATOR) のコメントでしか走らないよう `pull_request.yml` で制限しているが、`/verify` はコメント時点のブランチ先端を clone して manifest の `checkver.script` を PowerShell として実行するため、差分を読んだ後に force-push されると読んだものと別のコードが走る。fork PR に `/verify` は使わず、差分を読んだ上で自分のブランチに取り込み、同一リポジトリの PR として開き直して `opened` で検証する。
- ラベル `manifest-fix-needed` / `review-needed` はリポジトリに事前作成されていない。無ければ `gh label create <ラベル名>` で作る。
- コミット件名は `feat: <name> を追加`。
