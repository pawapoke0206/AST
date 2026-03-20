# AST (AJS Support Tool)

JP1/AJS運用保守を支援するPython製GUIツール。
銀行システムのAIX/Linux環境でのジョブ定義取得・変換・解析を自動化する。

## リポジトリ情報

- GitHub: https://github.com/pawapoke0206/ast（パブリックリポジトリ）
- 旧リポジトリ名: ajs_helper

## パブリックリポジトリに関する注意事項

- このリポジトリはパブリックであり、誰でも閲覧可能
- コミット前に必ず以下を確認すること：
  - 銀行名・顧客情報・接続先ホスト名など業務固有の情報が含まれていないか
  - config.json等の設定ファイルが.gitignoreに含まれているか
  - 新規追加ファイルに機密情報が含まれていないか
- コミットメッセージにも業務固有の情報を含めないこと

## プロジェクト構成

```
ast/
├── ajs_main.py              # GUI本体 (tkinter/ttk) - エントリーポイント
├── ajs_constants.py          # 共通定数 (パス, 文字コード, ログ設定)
├── ajs_print_logic.py        # Tab1: 定義取得・変換ロジック
├── ajs_define_logic.py       # Tab2: 定義回復ロジック
├── ajs_inout_logic.py        # Tab3: 入出力解析ロジック (最も複雑)
├── ajs_rel_logic.py          # Tab4: 先行関係解析ロジック (NetworkX)
├── ajs_depend_logic.py       # Tab5: 依存関係解析ロジック (Tab3+4の組み合わせ)
├── ajs_exception_editor.py   # I/O例外ルールエディタ (別ウィンドウ)
├── build.spec                # PyInstaller設定 (--onefile)
├── build.bat                 # ビルドスクリプト (ASCII-only必須)
├── BUILD_README.md           # ビルド手順・Defender誤検知対策
├── config.json               # 銀行別の初期変数設定 (.gitignore対象)
├── io_exceptions.json        # 入出力解析の手動例外ルール (.gitignore対象)
├── AJS_trans.prm             # 環境変換テーブル 本番⇔ミラー⇔開発 (.gitignore対象)
└── history.json              # GUI入力履歴 (自動生成, .gitignore対象)
```

## 開発環境

- WSL (Ubuntu) + VS Code + Claude Code
- ソースパス: ~/projects/ast → シンボリックリンクでWindows側を参照
- ビルド: Windows側で `build.bat` を実行 (PyInstaller --onefile)
- Python 3.x + tkinter/ttk

## 主要ライブラリ

- **paramiko**: SSH/SFTP接続 (JP1サーバとの通信)
- **networkx**: グラフ理論 (先行関係・依存関係の解析)
- **openpyxl**: Excel出力 (入出力解析結果)

## アーキテクチャ

### データフロー

```
GUI入力 → gui_vars辞書 → 各ロジックファイルの*_start_job() → SSH経由でajsprint実行 → 結果ファイル取得・解析 → 出力
```

### GUI構造 (ajs_main.py)

- グローバルスクリプト方式 (クラスベースではない)
- `gui_vars_map` 辞書でGUI変数をロジックに渡す
- `gui_funcs_common` 辞書で共通関数 (`update_status`, `get_ssh_client`等) を渡す
- `run_in_thread()` でロジック実行をスレッド化しGUIブロックを防止
- 全体スクロール用Canvas + タブ内ウィジェットのスクロール干渉防止

### UIデザインシステム

Task-elと共通の `_COLORS` パレットを使用:
- bg: `#F5F6FA`, accent: `#3B82F6`, section_bg: `#F0F4FF`
- ttkスタイル: `Section.TLabelframe`, `Common.TLabelframe`, `Custom.TNotebook`
- 実行ボタン: `create_accent_button()` で青フラットボタンに統一

### 基準パス (ajs_constants.py)

- `get_base_path()`: exe時は `sys.executable` の親、スクリプト時は `__file__` の親
- config.json, io_exceptions.json, AJS_trans.prm, log/ はすべて基準パスからの相対参照

## 5つのタブの機能

| Tab | ファイル | 機能 | キーとなる処理 |
|-----|---------|------|--------------|
| 1 | ajs_print_logic.py | 定義取得・変換 | ajsprintでrecover/verify取得、AJS_trans.prmで環境変換 |
| 2 | ajs_define_logic.py | 定義回復 | ファイルをSFTPアップロード→ajsdefine実行、改行LF固定 |
| 3 | ajs_inout_logic.py | 入出力解析 | シェルスクリプトの静的解析→変数展開→I/Oファイル特定 |
| 4 | ajs_rel_logic.py | 先行関係解析 | NetworkXでグラフ構築→必要ユニット計算→ar行再結線 |
| 5 | ajs_depend_logic.py | 依存関係解析 | Tab3のI/O情報+Tab4のグラフでBFS逆引きトレース |

## ビルド

```bash
# Windows側で実行
build.bat
# 出力: dist\AST.exe
```

### Defender誤検知対策 (3層)

1. UPX圧縮無効 (`upx=False` in build.spec)
2. bootloaderをソースからリビルド (`pip install --no-binary pyinstaller`)
3. `--onefile` モード (以前のonedirから変更)

### build.batの制約

- **ASCII文字のみ使用** (日本語禁止) - cmd.exeのエンコーディング問題を回避するため

## コーディング規約・注意点

- **文字コード**: `ENC` 辞書で管理 (SJIS=cp932, UTF-8=utf-8)
- **改行コード**: `NL` 辞書で管理 (CRLF/LF)
- **ログ**: 各タブごとに `LOG_DIR / "tabN_xxx.log"` + JSON詳細ログ
- **SSH接続**: `get_ssh_client()` でparamiko.SSHClient生成、with文で使用
- **銀行リスト**: `BANKS` 定数で定義
- **履歴**: history.json に直近10件を保存、Comboboxで表示
