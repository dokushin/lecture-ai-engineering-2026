# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：
  - 第1回（素のLLM）では、講座固有のルール（宿題の3レベル）を知らないため、一般的なプログラミングの難易度などを推測で答えたり「わからない」と回答した。一方、今回のエージェントは回答前に自動で `search_glossary` を呼び出して用語集を参照し、「基礎・実装・発展の3つのレベルがあり、どれか1つを達成すればよい」と根拠（第1回）付きで正確に回答した。
- `--trace` で見えたツール呼び出し（ツール名と引数）：
  - 「LoRAとQLoRAの違いは？」の質問時：`search_glossary({'query': 'LoRA'})`, `search_glossary({'query': 'QLoRA'})`
  - 確認問題の出題依頼時：`make_quiz({'lecture': 2})`
- 1つの質問での LLM 呼び出し回数（`requests`）：
  - `2` 回（1回目でツール呼び出しを要求し、2回目でツールの戻り値を受け取って最終回答を生成したため）
- 追加したツールと、docstring に書いた説明：
  - 追加したツール：`make_quiz(lecture: int) -> str`
  - docstring の説明：
    ```text
    指定した回（1始まりの整数）の用語集から1つ用語を選び、確認問題とその答えを返す。

    「第2回の復習問題を出して」「確認問題を1問ちょうだい」のように、
    特定の講義回の理解度チェックや確認問題を求められたときに使うこと。
    ```

## 宿題

- 取り組んだレベル：実装
- やったこと：
  - `app/tools.py` に、学習記録を `learning_log/` フォルダへ日時付きの Markdown ファイルとして保存するツール `save_learning_log(title: str, content: str) -> str` を実装した。
  - ファイル保存は取り消しにくい副作用のある操作（書くツール）であるため、`app/agent.py` で `requires_approval=True` を付けて登録し、`Agent` の `output_type` に `[str, DeferredToolRequests]` を指定した。
  - `main()` 関数に承認待ちループ（`while isinstance(result.output, DeferredToolRequests):`）を追加し、実行前にターミナルで `[y/N]` の確認を求め、`y` なら `True`、それ以外なら `ToolDenied("利用者が保存を断りました。")` を `DeferredToolResults` に格納して `message_history=result.all_messages()` とともに再開する Human-in-the-Loop フローを実装した。
- 結果：
  - `uv run python -m app.agent --trace "LoRAについて調べて、今日学んだこととして記録に保存して"` を実行した。
  - まず `search_glossary({'query': 'LoRA'})` が自動で呼ばれ、その結果をもとにモデルがタイトルと要約 Markdown を組み立てて `save_learning_log` を要求した時点で一時停止し、`[承認待ち]` プロンプトが表示された。
  - **`n`（拒否）を入力した場合**：ファイルは作成されず、`[tool return] save_learning_log -> 利用者が保存を断りました。` がモデルに返り、「保存を試みましたが、保存処理がキャンセルされました」と適切に応答した。
  - **`y`（承認）を入力した場合**：`learning_log/` 内に Markdown ファイルが保存され、保存完了の旨が回答された。
- 気づき・疑問：
  - 同じエージェント内でも、読むだけの安全なツール（`search_glossary`）は自動実行させ、副作用のある書くツール（`save_learning_log`）だけを `requires_approval=True` でピンポイントに止めて人間が確認できる仕組みの重要性を実感した。
  - 拒否（`n`）した際にも例外でプログラムを落とすのではなく、`ToolDenied` で理由テキストをモデルに返すことで、モデル自身が「断られた」という状況を認識して自然な回答を続けられる設計になっている点が興味深かった。
  - また、再開後のトレースの `[usage]` では `requests=1` と表示されており、`run_sync` ごとに使用量が集計される（再開前の呼び出しと再開後の1回分に別れる）挙動も確認できた。