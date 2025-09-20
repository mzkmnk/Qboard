# Amazon Q CLI × VS Code 拡張（PoCメモ）

このドキュメントは、Amazon Q CLI（`q`）を VS Code 拡張からラップし、最小構成でチャット体験を実装するための調査結果と実装方針をまとめたものです。現状は JSON/JSONL 出力が未提供のため、当面は `--no-interactive` を前提に進めます。

## 結論（要点）

- 実現可否: 可能。`q chat --no-interactive` を都度起動し、標準出力テキストを取り込み表示する。
- 文脈継続: `--resume` と同一 `cwd` の組み合わせで継続可能（履歴を別途保存しなくても簡易継続）。
- 構造化出力: JSON/JSONL は現状なし。出力はテキスト（ANSIカラー・軽いTUI装飾含む）。
- ツール実行: 非対話では確認ができないため失敗しやすい。`--trust-tools=<限定>` で段階的に許可する運用が現実的。

## 実行サンプル（検証済み）

```bash
# 1回の往復（折返し無効）
q chat --no-interactive --wrap never "簡単な挨拶をして"

# 文脈継続（同じ cwd で resume）
q chat --no-interactive --wrap never --resume "続きの質問"

# ツール使用が必要な場合（例: 読み取り系のみ許可）
q chat --no-interactive --wrap never --resume --trust-tools=fs_read \
  "プロジェクト直下のファイル一覧を教えて"
```

補足:

- `--wrap never` で折返しを抑制（UI整形が容易）。
- 非対話でも、バナーやヒントが先頭に出力される場合がある（後述のフィルタで対処）。

## ツール許可の扱い（エラー時のフロー）

非対話モードでは、ツール実行の許可確認ができずに失敗・ハングすることがあります。以下のフローを推奨します。

1. まず許可なし（または安全な読み取り系のみ）で実行。
2. 出力のテキストからツール使用の兆候を検出し、必要な許可を UI でユーザに確認。
3. 許可を得たら、同じ `cwd` で `--resume` と拡張した `--trust-tools` を付けて再実行。

検出の目印（例）:

```text
Using tool: fs_read
Using tool: fs_write
```

実装上は ANSI を除去してから、行テキストに対して以下のような正規表現で検出します。

```regex
^Using tool:\s*(\S+)
```

注意:

- `--trust-all-tools` は強力だが高リスク。既定は `--trust-tools` の段階的拡大を推奨。

## 出力整形（ANSI・装飾の対処）

- `strip-ansi` 等で ANSI コードを除去する。
- 先頭に出るバナー（ASCIIアートや「What's New」等）やヒントボックスは不要ならフィルタする。
  - 例: 先頭数十行のうち、既知フレーズ（"What's New in Amazon Q CLI", "Did you know?"）を含むブロックを除外。
- 実際の回答は多くが Markdown 風なので、Webview 側で Markdown としてレンダリングするか、プレーンテキストで表示する。

## VS Code 拡張（最小PoC方針）

- 実行方式: `child_process.spawn`（または `execa`）でワンショット起動。
- コマンド例: `q chat --no-interactive --wrap never [--resume] [--trust-tools=…] "<ユーザ入力>"`
- ストリーミング: `stdout` を逐次受信→ANSI 除去→Webview に反映。
- タイムアウト: 30–60秒程度を目安に設定（ハング検知）。
- ツール許可: 既定は未許可 or 読み取り系のみ。失敗時に UI で許可確認→`--resume` で再実行。
- セッション: 候補は2つ。
  - 単一セッション: ワークスペースの `cwd` を統一して常に `--resume`。
  - 複数セッション: 拡張ストレージ配下にセッション別の作業ディレクトリを割当（同一セッションID→同一cwd）。

擬似コード（概略）:

```ts
const args = [
  'chat',
  '--no-interactive',
  '--wrap',
  'never',
  '--resume',
  userPrompt,
  ...(trustedTools.length ? [`--trust-tools=${trustedTools.join(',')}`] : []),
];
const child = spawn('q', args, { cwd });
child.stdout.on('data', (buf) => {
  const text = stripAnsi(buf.toString('utf8'));
  buffer += text;
  webview.postMessage({ type: 'append', text });
});
child.on('close', (code) => {
  if (code !== 0 || timedOut) {
    const neededTools = parseNeededTools(buffer); // /^Using tool:\s*(\S+)/ のヒットを収集
    promptUserToTrust(neededTools);
  }
});
```

## セキュリティ・運用上の注意

- `--trust-all-tools` は極力避ける（予期せぬコマンド実行・ファイル変更のリスク）。
- まずは `fs_read` のみなど最小権限で開始し、必要時に限定的に拡大。
- 実行前にユーザへ意図（例: ファイル書き込みを伴う）を明示する UI を用意する。
- タイムアウト・キャンセル（プロセス kill）を必ず用意。

## 既知の限界と今後の発展

- 構造化出力（JSON/JSONL）がないため、厳密なイベント駆動 UI（ツール開始/終了など）の実装は困難。
- 当面はテキストのヒューリスティックで補う。将来的に JSON/JSONL が提供されたら置換できるよう抽象化する。
- フルインタラクティブを UI に埋め込みたい場合は `node-pty + xterm.js` で `q chat` を TTY 越しに扱う（実装コスト高）。

## 参考（便利なオプション）

- `--no-interactive`: TUI を起動せず 1往復で終了。
- `--wrap never`: 折返し無効（UIでレイアウトしやすい）。
- `--resume`: 同一ディレクトリに紐づいた会話を継続。
- `--trust-tools=<names>`: 許可するツールをカンマ区切りで指定。
- `--agent`, `--model`: 利用するプロファイルやモデルの指定（必要に応じて）。

## .gitignore（推奨）

`q chat` 実行により、作業ディレクトリ直下に `qlog/` が作成されることがあります。リポジトリ直下で実行する場合は、以下を `.gitignore` へ追加することを検討してください（拡張の専用作業ディレクトリを使うなら不要）。

```gitignore
qlog/
```

---

質問や要望があれば、この方針に沿って最小PoC実装（Webview 入力欄→`q chat` 実行→出力表示）を進めます。
