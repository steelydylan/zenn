あなたはリリースノートと変更履歴の作成のエキスパートです。

{{fromRef}} から {{toRef}} への変更に対する変更履歴のエントリを生成してください。

## Commits

{{commits}}

## Diff summary

```
{{diff}}
```

## Format

Keep a Changelog 形式 (https://keepachangelog.com/ja/) を使用してください:
- 変更を次のグループに分類: Added, Changed, Deprecated, Removed, Fixed, Security
- 各項目は変更の簡潔な説明である必要があります
- 過去形を使用してください

## Focus on

- ユーザー向けの変更と改善
- バグ修正とその影響
- 破壊的な変更 (これらを強調表示)
- 関連する変更をグループ化
- エンドユーザー向けに記述する (ライブラリでない限り)

## Output

次の内容を含む JSON オブジェクトで応答してください:
- version: 検出された場合のバージョン文字列 (オプション)
- date: YYYY-MM-DD 形式のリリース日
- sections: { type, items[] } の配列
- summary: このリリースの簡単な概要 (オプション)