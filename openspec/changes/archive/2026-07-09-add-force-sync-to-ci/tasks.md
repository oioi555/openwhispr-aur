## 1. workflow_dispatch 入力追加

- [x] 1.1 `.github/workflows/update-aur.yml` の `on.workflow_dispatch` に `force_sync` boolean input (default `false`, description 付き) を追加する — *Spec: ci-workflow › Scheduled and manual triggering › Manual dispatch with force sync*
- [x] 1.2 update ジョブの `if` を `needs.check.outputs.needs_update == 'true' || inputs.force_sync == true` に拡張する — *Spec: ci-workflow › Scheduled and manual triggering › Manual dispatch with force sync*
- [x] 1.3 package-specific steps (`Update openwhispr-bin`, `Update openwhispr-vulkan`) の `if` も `force_sync == true` で実行されるよう拡張する — *Spec: ci-workflow › Package update job › force sync scenarios*

## 2. Step summary の force_sync マーカー

- [x] 2.1 update ジョブの最終 step summary 生成ステップで、`force_sync == true` の時に `🔄 Force sync triggered` を先頭に表示する — *Spec: ci-workflow › Step summary reporting › Summary content*

## 3. 検証

- [x] 3.1 `actionlint` (または YAML 構文チェック) でワークフロー定義が valid であることを確認する
- [x] 3.2 `inputs.force_sync` の boolean 評価が GitHub Actions 式で正しく動くことをドキュメント/型定義から確認する
