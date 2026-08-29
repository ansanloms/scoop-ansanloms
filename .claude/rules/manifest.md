# manifest の追加

この bucket (`bucket/*.json`) に Scoop manifest を新規追加する手順。既存 manifest の更新は Excavator (`.github/workflows/schedule.yml`) が自動で行うため、この文書は新規追加のみを扱う。

## 調査

manifest を書く前に、上流について次を実測して確定する。学習データや推測で埋めない。

- リリース一覧: `gh release list -R <owner>/<repo> --limit 10`。全件が Pre-release かどうかを必ず見る (checkver の方式が変わる)。
- 最新リリースの詳細: `gh release view <tag> -R <owner>/<repo> --json tagName,assets,body,isPrerelease`。アセットのファイル名とタグの対応 (`$version` 展開の形) と、body に書かれた依存要件・注意事項。アセット名がリリースごとに変わりうる (番号接頭辞の付け外し等) なら、過去リリースのアセット名も見る (下記「checkver / autoupdate」)。
- アセットの中身: zip をダウンロードして `unzip -l` で見て、トップレベルの階層と配置すべきファイルを確定する。
- ライセンスと説明: `gh api repos/<owner>/<repo> --jq '.license.spdx_id'` と `gh api repos/<owner>/<repo> --jq '.description'`。どちらも出典であって転記元ではない。`spdx_id` は上流が LICENSE を SPDX 認識可能な形で置いていないと `NOASSERTION` を返す (adobe-fonts/source-han-sans は README と同梱 LICENSE.txt で OFL-1.1 を明記しているが API は `NOASSERTION`) ので、同梱の LICENSE / README と突合して識別子を決める。`description` は開発メモのことがある (aviutl2_gcmzdrops2 は「ごちゃまぜドロップス2、今後はGCMZDrops2表記にしていきたい」) ので、そのまま使わず言い換える。
- インストール手順: 上流 README のインストール節。GUI 経由の手順しか無い場合は、配置先を本体側の仕様 (下記「配置先の型」) から確定する。
- ハッシュ: ダウンロードした zip の `sha256sum`。

## 配置先の型

既存 manifest は型 1〜4 のいずれかに分かれる。新規追加ではどの型かを先に決める。

1. 単体アプリ: `url` / `hash` に加え、GUI アプリは `shortcuts` (`aviutl2.json`)、CLI ツールは `bin` (`mpd.json`) を書く。
2. TVTest / AviUtl1 のプラグイン: `depends` に `sudo` と本体を置き、`installer.script` で `$scoopdir\apps\<host>\current\...` へ `New-Item -ItemType SymbolicLink` で symlink を張り、`uninstaller.script` で `(Get-Item <link>).Delete()` で消す (`aviutl-exedit.json`)。古い manifest には必須プロパティや書式が CI の基準に合わないものがある (`tvtest-video-decoder.json` は `description` と `license` を欠く。`tvcaptionmod2.json` は `description` を欠き、インデントが 2 スペースで `formatjson.ps1` の 4 スペースと合わない)。そのまま雛形にすると PR 検証 CI が落ちる。
3. AviUtl ExEdit2 のプラグイン・スクリプト (`gcmzdrops2.json`、`psdtoolkit2.json`): 配置先は本体の `$dir` ではなく `$env:ProgramData\aviutl2\{Plugin,Script,Alias,Language}`。AviUtl2 は `ProgramData\aviutl2` 配下を読み (`Plugin` と `Script` は一つ下のフォルダも対象)、exe と同じフォルダに `data` フォルダがある場合だけそちらを読む (aviutl2 配布物の `aviutl2.txt` 「ファイル配置」節)。bucket の `aviutl2.json` は `data` を作らないため `ProgramData` 側が対象になる。aviutl2 を一度も起動していないと `ProgramData\aviutl2` が無いので、リンクを張る前に `New-Item -ItemType Directory -Force` で対象ディレクトリを用意する。
   - リンクは symlink ではなく junction (`New-Item -ItemType Junction -Path "$env:ProgramData\aviutl2\Plugin\<Name>" -Value "$dir\Plugin\<Name>"`)。junction は昇格不要で、symlink は `SeCreateSymbolicLinkPrivilege` が要る。uninstaller は installer と同じ権限で動く必要があるため、`depends` に `sudo` を置かず、script 内で `sudo` も呼ばない (型 2 の symlink + `sudo` は張り先が `$scoopdir` 配下だから成立していた)。
   - 張る前に張り先の既存物を見る。reparse point (アンインストール失敗で残った junction) なら削除して張り直し、実ディレクトリなら `Write-Warning` を出してリンクを飛ばす。
   - 他パッケージのファイルと同じフォルダに並ぶもの (`Alias\*`、`Language\*`) は junction にせずコピーする (`Get-ChildItem "$dir\Alias" -File | Copy-Item -Destination "$env:ProgramData\aviutl2\Alias"`。`-Force` を付けず、名前衝突をエラーとして表面化させる)。uninstaller は `$dir` 側のファイル名で `Remove-Item -LiteralPath ... -Force -ErrorAction SilentlyContinue` する。`notes` に「コピーであり、ローカル編集は更新で消える」と書く。
   - uninstaller は対象が reparse point のときだけ `[System.IO.Directory]::Delete($link)` で消し、実ディレクトリは `Write-Warning` で残す。`catch { }` を空にしない (scoop は hook のエラーを無視するので、黙って失敗すると dangling な junction が残り次の install を壊す)。`catch` では `Write-Warning` で理由を出す。
   - 利用者が編集するファイル (`PSDToolKit.user.ini` 等) は `persist` に挙げ、`pre_install` で `$persist_dir` 側にだけ作る。`$dir` 側にも作ると `persist_data` が毎回 `*.original` を残す。ディレクトリの persist (`GCMZShared`) は `persist` に挙げるだけでよい。
   - `notes` の文字列で `$env:ProgramData` は展開されない (scoop の `substitute` の対象外)。`%ProgramData%\aviutl2\...` と書く。
   - `.au2pkg.zip` は AviUtl2 の GUI が解釈するパッケージ形式だが、実体はデータフォルダの相対配置 (`Plugin/`, `Script/`, `Alias/`, `Language/`) をそのまま zip にしたもの。Scoop の展開判定は末尾 `.zip` の一致なので通常の zip として展開される。公式のパッケージ機構 (メニューの「パッケージ情報」) は経由しないため、`notes` にその旨と scoop でアンインストールすることを書く。
4. フォント: `installer.script` で `$env:windir\Fonts` へコピーし `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts` にレジストリ登録する。`uninstaller.script` で両方を戻す (`cica.json`、`udev-gothic.json` 等)。既存のフォント manifest を写す。

共通事項:

- `depends` は「無ければ入れる」だけで、下限バージョンは指定できない。上流が要求するバージョンは `notes` に書く。
- `scoop uninstall` は `uninstaller.script` を実行してから `$dir` を削除するので、uninstaller から `$dir` 配下のファイル名を参照してよい。
- 利用者が生成するファイルが `$dir` 配下に置かれる場合は `persist` に挙げる (更新で消えるため)。

## checkver / autoupdate

- 上流に通常リリース (非 pre-release) がある場合: `"checkver": { "github": "<repo URL>" }`。タグに `v` 以外の接頭辞・接尾辞があれば `re` で拾う (`bondriver-mirakc.json`)。
- 全リリースが pre-release の場合: `checkver.github` は `<repo>/releases/latest` を固定で引き、pre-release を返さないため使えない。GitHub API の releases 配列を直接引く。

  ```json
  "checkver": {
      "url": "https://api.github.com/repos/<owner>/<repo>/releases",
      "jsonpath": "$[0].tag_name",
      "regex": "^v(.+)$"
  }
  ```

  `$[0]` は `created_at` 降順の先頭であってバージョン順ではない。上流が新しいリリースの後に旧系列のパッチを公開すると古いタグを拾い、Excavator (毎時 `-Update`) が manifest をバージョンダウンで書き換える。上流のリリース運用が単一系列であることを確認してから使う。

- アセット名がリリースごとに変わる場合 (番号接頭辞の付け外し等): `autoupdate.url` にファイル名を固定で書くと、変わった時点で Excavator が 404 の URL と壊れた hash で manifest を書き換える。`checkver.url` を `releases/latest` の API にし、`jsonpath` で対象アセットの `browser_download_url` に絞り、regex の名前付きグループでタグとファイル名を取る (`source-han-sans-jp.json`)。

  ```json
  "checkver": {
      "url": "https://api.github.com/repos/<owner>/<repo>/releases/latest",
      "jsonpath": "$.assets[?(@.name =~ /<Name>\\.zip$/)].browser_download_url",
      "regex": "download/([\\d.]+R)/(?<filename>[^\"]+)$"
  },
  "autoupdate": {
      "url": "https://github.com/<owner>/<repo>/releases/download/$version/$matchFilename"
  }
  ```

  checkver は「1 つの本文に 1 回 regex をかける」だけで、グループ 1 が `$version`、それ以外の名前付きグループが `$match<Name>` (先頭大文字) として `autoupdate.url` に置換される (`scoop/bin/checkver.ps1` の `$matchesHashtable`、`scoop/lib/autoupdate.ps1` の `Get-VersionSubstitution`)。`checkver.github` は HTML ページを固定で引くので `jsonpath` と併用できない。
- `autoupdate.url` はアセット URL のバージョン部分を `$version` に置き換える。タグに `v` が付くなら `v$version` と書く。

## 必須プロパティと書式

- `description` と `license` は必ず入れる。PR 検証 CI (`.github/workflows/pull_request.yml`) が無いものを落とす。
- 書式は `scoop/bin/formatjson.ps1` の出力に合わせる (CI が lint する)。
- 4 バイト文字 (絵文字等) を manifest に書かない。非 ASCII のファイル名を扱う場合は manifest に直書きせず `Get-ChildItem` で列挙する。

## 検証

WSL の pwsh (`/usr/bin/pwsh`) で、リポジトリ直下から実行する。隔離 worktree では `scoop/` submodule が空なので、先に `git submodule update --init` を実行する。新規 manifest は untracked のため、検証前に `git add bucket/<name>.json` で index に載せてから `git diff` を見る (untracked のままだと diff が常に空になり、ツールによる書き換えを検出できない)。既存 manifest は改行コードが CRLF で、新規 manifest も CRLF で保存する。Linux の pwsh で `formatjson.ps1` や `checkver.ps1 -u` を実行すると末尾の改行が LF に書き換わり `git diff` に 1 hunk 出るので、その hunk は無視し、コミット前に CRLF へ戻す (`sed -i '$ s/$/\r/' bucket/<name>.json`。戻した後 `tail -c 3 bucket/<name>.json | xxd -p` が `7d0d0a` になることを確認する)。CI は Windows で `formatjson.ps1` を実行して比較するため、LF が 1 行でも混じると lint で落ちる (`.gitattributes` は未整備なので手で保つ)。

Windows 実機での `scoop install` / `scoop uninstall` は、WSL の interop 経由で Windows 側の Scoop を呼んで検証する (`cmd.exe /c "scoop --version"` で到達できることを先に確認する)。検証できなかった場合のみ、報告に「未検証」と明記する。

- Windows 側のコマンドは cwd を `/mnt/c/` 配下に置いて実行する。cwd が `\\wsl.localhost\...` だと cmd.exe が UNC 非対応の警告を出す。`cmd.exe /c "..."` の代わりに `powershell.exe -NoProfile -NonInteractive -Command '...'` でもよい (出力は `tr -d '\r'` を通す)。
- 単体の manifest は Windows 側のパス (例: `C:\Users\<user>\AppData\Local\Temp\<name>.json`) にコピーし、`cmd.exe /c "scoop install <その Windows パス>"` で入れる。ローカル json から入れた app は bucket に追従しないので、検証後は `scoop uninstall <name>` で戻し、マージ後に `scoop install ansanloms/<name>` で入れ直す。
- 未マージの manifest 同士に `depends` がある場合、ローカル json のパス指定では `depends` が解決できない。Windows 側に一時 bucket を作る: `bucket/` を `C:\Users\<user>\AppData\Local\Temp\ansanloms-wip\bucket` にコピーし、その中で `git init` と commit をしてから `scoop bucket add ansanloms-wip C:/Users/<user>/AppData/Local/Temp/ansanloms-wip` (スラッシュ区切り必須。`\\wsl.localhost\...` の UNC パスは `scoop/lib/buckets.ps1` の URL 判定 (`<provider>/<user>/<repo>` 形の正規表現) で弾かれ、Windows 側 git の clone も dubious ownership で失敗する。global の git config を変えて回避しない)。`scoop install ansanloms-wip/<name>` で `depends` が一時 bucket から解決される。片付けは `scoop uninstall <name> -p` (persist も消す) → `scoop bucket rm ansanloms-wip` → `Remove-Item -Recurse -Force <一時 bucket>`。
- プラグイン型 (型 2 / 3) の検証順序: install → リンクと persist の確認 (`Get-Item <link> -Force | Select-Object FullName,LinkType,Target`、persist は `$env:USERPROFILE\scoop\persist\<name>`) → `scoop uninstall <name>` (persist は残る) → 再 install (persist が再利用され、`*.original` が出ないこと) → 片付け。GUI アプリ本体は起動しない (利用者に任せる)。
- `sudo` が要る manifest は `cmd.exe /c "sudo scoop install ..."` で Windows 組み込みの `sudo.exe` が UAC を出す。人が UAC に応答するまで待つので `timeout` を長めに取る。非管理者で実行すると installer のガードで止まるが、Scoop 側に "Install failed" のエントリが残るため、`scoop uninstall <name>` で掃除してから昇格して入れ直す。
- レジストリの確認は `reg.exe query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Fonts"` を WSL から直接呼ぶ。`cmd.exe /c 'reg query "..."'` で包むと `Windows NT` の空白で引用が壊れ、構文エラーになる。
- `scoop list <a> <b>` は最初の 1 つしか表示しない。複数を見るなら引数無しの `scoop list` を使う。
- 日本語のファイル名は powershell.exe / cmd.exe の出力を WSL で受けると文字化けする (表示だけの問題)。ファイルの有無はサイズや `Get-Item` の結果で確認し、名前の見た目で判断しない。

```sh
pwsh -File ./scoop/bin/checkver.ps1 <name> -dir ./bucket -u -f   # 実行後の git diff が末尾改行の hunk だけなら version / url / hash が autoupdate と整合
pwsh -File ./scoop/bin/checkhashes.ps1 <name> -dir ./bucket
pwsh -File ./scoop/bin/formatjson.ps1 <name> -dir ./bucket       # 実行後の git diff が末尾改行の hunk だけなら書式が整合
pwsh -c "Test-Json -Json (Get-Content -Raw bucket/<name>.json) -SchemaFile scoop/schema.json"
```

`checkver.ps1` は App 引数にファイルパスを渡すと bucket 外のファイルを対象にできる (`-Update` 無しなら書き込まない)。checkver の設計を試すときは manifest のコピーを一時ディレクトリに置いて dry-run する。

`scoop/` は submodule で特定のコミットに固定されている。CI は最新の Scoop を使うので、ローカルの結果と食い違ったら CI の判定を優先する。

## PR

- PR 検証 CI は PR の `opened` イベントと、PR への `/verify` コメントでしか走らない。修正を push した後は `gh pr comment <番号> --body /verify` で再実行する。`/verify` で走るのは main にある `pull_request.yml` なので (`issue_comment` は default branch の workflow を使う)、`pull_request.yml` 自体を追加・変更する PR ではその変更は `/verify` に反映されない。`/verify` の実行は `issue_comment` イベントなので `gh pr checks` には出ない。`gh run list --workflow "Pull Requests"` で結果を見る。
- 結果は PR コメントと `manifest-fix-needed` / `review-needed` ラベルで返る。ラベルは action が初回に自動作成して付ける (既定色 `ededed`)。手で `gh label create` する必要は無い。
- fork からの PR では `opened` で検証もコメントも行われず、workflow は成功扱いで終わる (read-only token のため)。`/verify` は維持者 (OWNER / MEMBER / COLLABORATOR) のコメントでしか走らないよう `pull_request.yml` で制限しているが、`/verify` はコメント時点のブランチ先端を clone して manifest の `checkver.script` を PowerShell として実行するため、差分を読んだ後に force-push されると読んだものと別のコードが走る。fork PR に `/verify` は使わず、差分を読んだ上で自分のブランチに取り込み、同一リポジトリの PR として開き直して `opened` で検証する。
- コミット件名は `feat: <name> を追加`。
