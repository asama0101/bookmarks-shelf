# サイドバーのグループタブをデフォルト表示にする

- 日付: 2026-08-07
- 対象ファイル: `bookmarks.html`, `bookmarks-filesync.html`

## 背景

サイドバーは「タグ」「グループ」の2タブ切り替え式（`state.sidebarTab`）。現状は常に「タグ」タブが初期選択されており、グループでの絞り込みを主に使うユーザーにとっては毎回タブを切り替える手間がある。

## 要件

- ページを開いた直後は、常に「グループ」タブがアクティブ・表示された状態にする。
- 前回選択したタブを記憶する永続化は行わない（起動のたびに必ずグループタブから始まる）。
- 「タグ」タブへの切り替え・タブ間の行き来、既存のキーボードショートカット（`t`/`g`）の挙動は変更しない。

## 設計

現状、初期表示は2箇所に分かれて決まっている:

1. JS の状態初期値: `state.sidebarTab: 'tag'`
2. HTML マークアップのハードコードされた初期属性:
   - `#sidebar-tab-tag` に `aria-selected="true"`、`#sidebar-tab-group` に `aria-selected="false"`
   - `#tag-nav-panel` は `hidden` 属性なし（表示）、`#group-nav-panel` は `hidden` 属性あり（非表示）

起動時に `switchSidebarTab()` を呼んでDOMを状態と同期させる処理は存在しないため、(1) だけを変更してもタブボタンの見た目・パネルの表示は変わらない。したがって両方を変更する。

### 変更内容（`bookmarks.html` / `bookmarks-filesync.html` 共通）

1. `state.sidebarTab` の初期値を `'tag'` → `'group'` に変更
2. 初期 HTML マークアップを入れ替え:
   - `#sidebar-tab-tag` → `aria-selected="false"`
   - `#sidebar-tab-group` → `aria-selected="true"`
   - `#tag-nav-panel` → `hidden` 属性を追加
   - `#group-nav-panel` → `hidden` 属性を削除

### 変更しないもの

- `switchSidebarTab()` 関数のロジック
- キーボードショートカット（`t`/`g`）の挙動
- `state.tag` / `state.group` フィルタ状態そのもの（タブ切り替えとフィルタは独立という既存の設計を維持）
- 永続化の仕組み（localStorage / ファイル同期データへのタブ状態保存は追加しない）

## 検証方法

自動テストは存在しないプロジェクトのため、ブラウザでの手動確認のみ:

- `bookmarks.html` を（localStorage をクリアした状態、または新しいブラウザプロファイルで）開き、初回表示でグループタブがアクティブ・グループナビが表示されていることを確認
- タグタブへの切り替え → グループタブへ戻る操作が従来通り動作することを確認
- `t`/`g` キーボードショートカットでタブ切り替え・フォーカス移動が従来通り動作することを確認
- `bookmarks-filesync.html` でも同様に確認（Chrome/Edge）

## スコープ外

- タグ管理機能、ショートカットキーの改善、追加フォームの簡略化は別スペックで扱う。
