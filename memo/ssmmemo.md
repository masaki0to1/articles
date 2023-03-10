# SSM についてのまとめ

## パラメータのバージョンについて
---

### CFn テンプレートに記述する際の注意点
前提：
- テンプレート内で参照したい SSM Parameter Store のパラメータのバージョンを指定しなければ、最新のパラメータを動的参照できる（2021年4月のアップデートより）
- これにより、パラメータのバージョンが上がる度に、テンプレートの該当箇所を修正する作業から解放された

本題：
- テンプレートの Resourses セクションの Properties 内で 参照する場合は、上記前提通り、バージョン未指定で最新パラメータを参照できるが、**Parameters セクションで参照する場合は、バージョンを指定しないとエラーになる**。
- なお、動的参照しているパラメータを更新した場合は、再度参照元スタックを更新デプロイしないと反映されない
- テンプレート Parameters セクションで SSM Parameter Store のValue を指定する際は、公式では パラメータの Type に AWS::SSM::Parameter::Value<\String> を指定できるとあるが、エラーが出るので、String 型を指定する
