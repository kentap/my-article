---
title: "Windows版OrcaでWSL2内のリポジトリを開発する"
emoji: "🐋"
type: "tech"
topics: ["orca", "wsl2", "claudecode", "gitworktree", "windows"]
published: false
---

Windows + WSL2でOrcaを動かすまでの手順と詰まった点を、自分用の備忘として残しておきます。

周りの開発者はほとんどMacを使っていて、[Orca](https://github.com/stablyai/orca) についても、私が見つけた手順はMac前提のものばかりでした。Orcaは、作業ごとにgit worktree(1つのリポジトリから作業フォルダを複数作るgitの機能。以下、ワークツリー)を作り、それぞれでClaude CodeやCodexを動かすデスクトップIDEです。私の環境はWindows + WSL2で、リポジトリもエージェントもすべてWSL内にあります。

結論から書くと、Windows版OrcaにはWSL対応が組み込まれていて、**画面から使うだけなら、ワークツリーの保存先をWSL内に変えるだけ**で開発できました。ターミナルもエージェントもWSL内で動きます。

WSLのシェルやWSL内のClaude Codeから `orca` コマンドでOrcaを操作したい場合は、作業が2つ増えます。

- Linuxのパスを変換するラッパースクリプトを置く
- Claude Code向けのスキルを `orca skills get` で手動で置く

人が自分でコマンドを打つ場面は多くないはずです。コマンドが要るのは主に、Claude Codeにワークツリーの作成や他のエージェントへの指示まで任せたいときです。この記事では前半で画面だけの使い方を、後半でコマンドで操作するための準備を書きます。

WSL側に自分で常駐プロセスやパッケージを追加する必要はなく、`.wslconfig` も変えていません。なお、Orcaは中継用の小さなNode.jsプロセス(`~/.orca-wsl/hook-relay/` 配下)をWSL内に自動で置いて動かします。WSL内にNode.jsが入っていない環境では、この部分の挙動を確かめていません。

## 検証した環境

| 項目 | バージョン |
|---|---|
| Orca(Windows版) | 1.4.206 |
| WSL | 3.0.1.0 |
| WSLのディストリビューション | AlmaLinux 10.0 |
| エージェント | Claude Code、Codex CLI(どちらもWSL内にインストール) |

WSL内には `git` と `gh`(GitHub CLI)も入っています。Mac版Orcaは試していません。

記事中のパスはサンプルに置き換えています。Windowsのユーザー名は `<WinUser>`、WSLのユーザー名は `<user>`、ディストリビューション名は `<Distro>`、リポジトリ名は `my-app` と表記します。

## 画面だけで使うなら、保存先を変えるだけでいい

### ワークツリーの保存先をWSL内の専用フォルダに変える

ワークツリーの保存先は、初期値が `C:\Users\<WinUser>\orca\workspaces` で、Windows側のフォルダになっています。WSL内のリポジトリを扱うなら、Orcaの設定画面の General にある **Workspace Directory** を、WSL内の専用フォルダに変えます。

```text
\\wsl.localhost\<Distro>\home\<user>\orca-worktrees
```

フォルダは事前に作らなくて大丈夫です。最初のワークツリーを作るときに、Orcaが作ります。ワークツリーは `~/orca-worktrees/<リポジトリ名>/<ワークツリー名>` のように、リポジトリごとのフォルダに分かれて作られます。

:::message alert
入力するときは、パスの末尾に空白が入らないように注意してください。Linuxは末尾が空白のフォルダ名を作れますが、Windowsではそのフォルダを扱えません。エクスプローラーやOrcaから開けないフォルダができるおそれがあります。
:::

私は一度、末尾に空白が付いたまま保存してしまいました。保存された値は、WSLから次のコマンドで設定ファイルを見て確かめられます。

```bash
grep -o '"workspaceDir":"[^"]*"' \
  /mnt/c/Users/<WinUser>/AppData/Roaming/orca/profiles/local-default/orca-data.json | cat -A
```

`cat -A` を付けると、行末が `$` で表示されます。`"$` の直前に空白がなければ正しく保存されています。このファイルは確認だけに使い、変更は設定画面から行ってください。Orcaの起動中に直接書き換えると、アプリ側の保存で上書きされるおそれがあります。

### リポジトリを登録して、画面から開発する

Orcaにリポジトリを登録すると、WSL内のリポジトリがそのまま画面に並びます。登録されるのはメタデータだけで、ファイルは複製されません。パスは `\\wsl.localhost\<Distro>\home\<user>\repo\my-app` というWindows形式で扱われます。私は後半のコマンドで登録したので、画面から追加する操作は試していません。

あとは画面でワークツリーを作り、ターミナルやエージェントを開いて開発します。ここまでで環境は整いましたが、実際の開発ではまだほとんど使っていません。ここから本格的に触っていくつもりです。

ワークツリーのターミナルで `pwd` と `uname -r` を実行すると、WSL内で、ワークツリーのフォルダを作業ディレクトリにして動いていることが分かります。WSLに入れた `claude` や `codex` もそのまま使えます。

```text
$ pwd; uname -r
/home/<user>/orca-worktrees/my-app/feature-login
6.18.40.1-microsoft-standard-WSL2
```

コミットとPRは、このターミナルで普段どおりに行います。

```bash
git add -A && git commit -m "..." && git push -u origin HEAD
gh pr create
```

このターミナルの中身はWSLの普通のbashなので、後半で紹介するラッパーは要りません。ターミナルからOrca自体を操作したい場合も、Orcaはターミナルに `ORCA_CLI_COMMAND=orca-ide` という環境変数を設定していて、公式の `orca-ide` を使う想定になっています。`orca-ide` は今いるフォルダから対象のワークツリーを推定できるので、自分のワークツリーに対する操作ならそのまま動きます。

```text
$ echo $ORCA_CLI_COMMAND
orca-ide
$ orca-ide worktree current
```

ワークツリーを削除するときに、注意点が1つあります。CLIのヘルプによると、ワークツリーのフォルダは消えますが、ブランチが消えるのは、マージ済みだとOrcaが判定できた場合だけです。GitHubでsquash mergeした場合などは残ることがあるので、`git branch` で確かめてください。

## ワークツリーごとに.venvなどを作り直す分のディスクに注意する

ワークツリーにチェックアウトされるのは、gitで管理しているファイルだけです。私が試したリポジトリは全体で約890MBありますが、gitで管理しているファイルは148KBしかなく、ワークツリーを作った直後はほとんどディスクを使いません。

容量の大半は、gitの管理外にある `.venv`(109MB)と、terraformのprovider(784MB)でした。これらはワークツリーにコピーされないので、ワークツリーで実行や検証をするなら、`uv sync` や `terraform init` で作り直すことになります。エージェントのプロセスもワークツリーごとに増えます。私は経験則として、同時に動かすワークツリーを2〜3個までにし、使い終わったものはすぐ削除しています。

## WSLからコマンドで操作したい場合

ここからは、WSLのシェルやWSL内のClaude Codeから `orca` コマンドを使うための準備です。画面だけで使うなら読み飛ばしてかまいません。Orcaのターミナルでリポジトリを触るだけなら、ここの作業は不要です。ラッパーが必要になるのは、`--repo path:$HOME/...` のようにLinuxのパスでOrcaの操作対象を指定したいときだけです。

### つまずく原因は、orca.exeがパスやコマンドをWindows側で解決すること

Orcaのコマンド本体 `orca.exe` はWindowsのプログラムです。そのため、パスの解釈も、`npx` などのコマンドやエージェントの検出も、Windows側で行われます。WSLから使うと、ここでずれが生まれます。

- `orca.exe` に `/home/<user>/...` のようなLinuxのパスを渡しても通じない
- `orca skills install` が、WSL内のエージェントや `npx` を見つけられない

前者はラッパーで、後者はスキルを手で置くことで回避しました。

### 公式のorca-ideは、Linuxのパスを変換しない

Orcaには、WSLから使う公式のコマンドも用意されています。設定画面の「一般 > Orca CLI > WSL シェルコマンド」で登録すると、WSL内に `~/.local/bin/orca-ide` が置かれます。ただ、`orca-ide` は引数に含まれるLinuxのパスを変換しません。リポジトリのフォルダの中で実行したり、`name:` で名前を指定したりする分には動きますが、`path:` にLinuxのパスを渡すと見つかりません。

```text
$ orca-ide worktree list --repo path:$HOME/repo/my-app
repo_not_found
```

PowerShellを経由して `orca.exe` を呼ぶ仕組みなので、起動にも時間がかかります。私の環境では、`orca-ide status` が約1.5秒、次に紹介するラッパーでは約0.5秒でした。Linuxのパスのまま操作したかったので、パスを変換するラッパーを別に用意しました。

### パスを変換するラッパーを置く

Windows版Orcaには、CLIの実行ファイルが同梱されています。

```text
C:\Users\<WinUser>\AppData\Local\Programs\orca\resources\bin\orca.exe
```

WSLにはWindowsの実行ファイルを直接呼べる仕組み(interop)があるので、この `orca.exe` はWSLからそのまま動きます。

```bash
/mnt/c/Users/<WinUser>/AppData/Local/Programs/orca/resources/bin/orca.exe status
```

出力に `runtimeReachable: true` が出れば、起動中のOrcaとつながっています。

ただし、Orcaが読めるのは `\\wsl.localhost\<Distro>\home\<user>\repo\my-app` という形式のパスです。そこで、引数に含まれるLinuxのパスを `wslpath -w` で変換してから `orca.exe` に渡すラッパーを作りました。

```bash:~/.local/bin/orca
#!/usr/bin/env bash
# WSLからWindows版OrcaのCLIを呼ぶラッパー
# --path <Linuxのパス> と path:<Linuxのパス> を \\wsl.localhost\... 形式に変換する
ORCA_EXE="/mnt/c/Users/<WinUser>/AppData/Local/Programs/orca/resources/bin/orca.exe"

to_win() { case "$1" in /mnt/*|/*) wslpath -w "$1" 2>/dev/null || printf '%s' "$1" ;; *) printf '%s' "$1" ;; esac; }

args=(); prev=""
for a in "$@"; do
  if [[ "$prev" == "--path" ]]; then a="$(to_win "$(realpath -m "$a")")"
  elif [[ "$a" == --path=* ]]; then a="--path=$(to_win "$(realpath -m "${a#--path=}")")"
  elif [[ "$a" == path:/* ]]; then a="path:$(to_win "${a#path:}")"
  fi
  args+=("$a"); prev="$a"
done
exec "$ORCA_EXE" "${args[@]}"
```

```bash
mkdir -p ~/.local/bin   # ~/.local/bin が PATH に入っていることも確認する
chmod +x ~/.local/bin/orca
orca status
```

変換するのは、`--path` オプションと、`path:` で始まる引数の2種類です。後者はOrcaで操作対象を指定する書き方(セレクタ)の一つで、`--repo path:...` や `--worktree path:...` のように使います。私が使った範囲では、Linuxのパスを渡すのはこの2種類だけでした。なお、`path:` のほうは相対パスを変換しないので、`$HOME` などから始まる絶対パスで書いてください。

### コマンドでリポジトリを登録し、ワークツリーを操作する

ラッパーがあれば、Linuxのパスのままリポジトリを登録できます。

```bash
orca repo add --path ~/repo/my-app
orca repo list
```

ワークツリーも、`path:$HOME/...` のようなLinuxのパスのまま操作できます。コマンドで作ったワークツリーは画面にも表示され、画面で作ったものもコマンドの一覧に出ます。

```bash
# 作業ごとにワークツリーを作る(同名のブランチもできる)
orca worktree create --repo path:$HOME/repo/my-app --name feature-login --activate

# 一覧と状況を見る
orca worktree list
orca worktree ps
```

`--activate` を付けると、作ったワークツリーがOrcaの画面に表示されます。付けなければ画面は切り替わりません。作成と同時にエージェントを起動して、指示を渡すこともできます。

```bash
orca worktree create --repo path:$HOME/repo/my-app --name feature-login \
  --agent claude --prompt "ログイン機能を実装して"
```

`--base-branch` を付けずに作ると、ワークツリーは手元で作業中のブランチではなく、リポジトリの既定のベース(私の環境では `main`)から分岐します。作業中のブランチの続きを進めたいときは `--base-branch <ブランチ名>` を付けます。

削除は次のコマンドです。ブランチが残る場合がある点は、画面から削除するときと同じです。

```bash
orca worktree rm --worktree name:feature-login
```

`name:` には `--name` で付けた名前を指定します(`orca worktree list` の `displayName` で確認できます)。

### スキルは `orca skills install` が使えないので、`skills get` で直接置く

Orcaには、エージェントに `orca` コマンドの使い方を教えるスキルが同梱されています。私はそのうちの2つを入れました。

- **orca-cli**: ワークツリーの作成と削除、ターミナルの読み取りと入力、別のエージェントへの作業の引き継ぎなど
- **orchestration**: 複数のエージェントにタスクを振り分け、進み具合を見ながらまとめる

スキルを入れるコマンドとして `orca skills install` が用意されていますが、WSLからは使えませんでした。オプションを変えて2回試し、どちらもエラーで止まりました。

```text
$ orca skills install --skill orca-cli
No coding agent detected on this host, so there is no install target. ...

$ orca skills install --skill orca-cli --agent claude-code
Running: npx --yes skills add https://github.com/stablyai/orca --skill orca-cli --global --agent claude-code -y
Could not run npx: spawn npx ENOENT. Install Node.js and ensure npx is on PATH.
```

1回目は、Windows側の `orca.exe` からはWSL内のエージェントが見えず、インストール先が見つからないというエラーです。2回目は `--agent` でインストール先を指定しましたが、Windows側にNode.jsが入っていないため `npx` を起動できませんでした。WSL内のNode.jsは使われません。`--global` が付くので、仮にWindows側にNode.jsを入れても、インストール先はWindowsのホームフォルダになるはずです(未検証)。

そこで、`orca skills get` でスキルの本文を取り出し、WSL内のClaude Codeのスキルフォルダに直接保存しました。`get` が返すのは、使っているOrcaのバージョンに合わせて同梱されている版です。

```bash
for s in orca-cli orchestration; do
  mkdir -p ~/.claude/skills/$s
  orca skills get $s > ~/.claude/skills/$s/SKILL.md
done
```

これで、WSL内のClaude Codeに「別のワークツリーでClaudeを起動して、この機能を作らせて」と頼めるようになります。Orcaを更新したら、同じコマンドでスキルを入れ替えてください。

## 詰まる箇所は、Orcaが何をWindows側で解決するかで予測できる

画面から使う分には、Orcaに組み込まれたWSL対応がほとんどを引き受けてくれます。手間がかかったのは、保存先の末尾空白と、コマンドで使うときのLinuxのパスや `orca skills install` でした。原因はどれも、Orcaがパスやコマンドの解決をWindows側で行うことです。今後Orcaに新しいコマンドが増えても、引数にLinuxのパスを取るか、WSL内のツールを呼ぶかを見れば、ラッパーで済むか、手作業が要るかを見分けられます。
