# 単体テスト作成ルール

## 基本方針

- テストは「何をテストしているか」が明確に伝わるように記述する
- 実装の詳細に依存せず、期待する動作に焦点を当てたテストを書く
- テストケースは境界値や特殊ケースを含め、機能を網羅的にカバーする

## テストコメントの目的と役割

テストコードのコメントは主に2つの目的を持ち、それぞれ異なる役割を果たします：

### 1. 仕様書としてのコメント（実装コードの理解を助ける）

テストは実装コードの仕様書としての役割も持ちます。このタイプのコメントは：

- **ビジネスルールの表現**: 技術的な実装よりも、ビジネス要件やドメインルールを明確に記述する
- **意図の説明**: 実装コードには書ききれない意図や背景をテストに記述し、コードの「なぜ」を伝える
- **境界条件の明示**: 「この入力ではこの出力を期待する」という境界条件や特殊ケースを説明する

例：
```ruby
it 'applies 10% discount for orders over 10,000円' do
  # 割引は税抜き価格に対して適用され、
  # 送料は割引対象外である
  # ...
end
```

### 2. 認知負荷低減のためのコメント（テストコード自体の理解を助ける）

テストコードは複雑になりがちで、その構造や目的を理解するためのコメントも重要です：

- **テスト構造の整理**: Arrange-Act-Assert（AAA）パターンの各セクションを明示し、テストの流れを把握しやすくする
- **モックの意図説明**: 複雑なモックやスタブの設定の目的を説明し、「なぜこのようなモックが必要なのか」を伝える
- **検証の焦点**: 多数の検証がある場合、それらが何を検証しているのかをグループ化して説明する

例：
```ruby
it 'returns user data with associated orders' do
  # このテストでは以下を確認する:
  # 1. ユーザーデータが正しく取得される
  # 2. 関連する注文データも含まれる
  # 3. 権限のない注文は除外される
  
  # 権限チェックのモック（このテストでは権限チェックの詳細ロジックは検証対象外）
  allow(Permission).to receive(:check).and_return(true)
  
  # ...
end
```

### コメントの書き方の原則

テストコメントを書く際は、以下の原則に従うことで、より価値の高いコメントになります：

1. **具体性**: 「正しい」「適切な」などの曖昧な表現ではなく、具体的に何を期待しているかを説明する
2. **完結性**: 文末は「〜する」「〜である」などの表現で完結させ、体言止めを避ける
3. **グループ化**: 関連する検証をまとめて一つのコメントで説明し、情報の密度を最適化する
4. **バランス**: DRYよりも明確さを優先し、テストの意図が独立して理解できるようにする

これらの原則に従うことで、テストコードは単なる検証ツールではなく、コードベースの理解を深め、保守性を高める重要なドキュメントとなります。

## テスト名と構造

- 具体的な期待値を含めたテスト名を使用する（例：`returns ["order", 2021-01-01, 2021-01-31 23:59:59] for January 2021`）
- `context` ブロックでテスト条件を明確に区分する

## 期待値の設定

- 実装をコピーした期待値（`Date.new(2021, 1, 31).end_of_day`）は避け、具体的な値または個別の属性で検証する
```ruby
expect(end_date.year).to eq(2021)
expect(end_date.month).to eq(1)
expect(end_date.day).to eq(31)
```
- 環境依存の値（タイムゾーン、マイクロ秒）は直接比較しない

## 境界値とエラーケース

- 境界値（数値の最小/最大、日付の月末/月初、閏年）をテストする
- エラーケースをテストする
```ruby
expect { controller.send(:get_filter_date_range, 'order_invalid_month') }.to raise_error(ArgumentError)
```

## テストの網羅性

- 網羅性チェックを実装し、新しい条件追加時にテストが失敗するようにする
```ruby
it 'tests all booking statuses' do
  explicitly_tested_statuses = %i[pending? approve? decline?]
  all_statuses = Booking.statuses.keys.map { |s| "#{s}?".to_sym }
  expect(all_statuses - explicitly_tested_statuses).to be_empty
end
```
- テストケースは独立させ、状態を共有しない

## プライベートメソッドとモック

- プライベートメソッドは `send` メソッドでテストする
- 外部依存はモックまたはスタブで分離し、最小限に抑える

## テストコードの可読性向上

### テスト構造の明確化

- **ルール**: `describe`直下の`let`や`before`ブロックには、何を設定しているのかを説明するコメントを追加する
- **理由**: テストの前提条件を明確にし、モックやスタブが何を表現しているのかを一目で理解できるようにするため
- **例**:
  ```ruby
  # テストで使用するモックオブジェクトの定義
  let(:business_group) { double('BusinessGroup') }
  let(:orders_relation) { double('ActiveRecord::Relation') }
  
  # 全テストケースで共通の前提条件を設定
  before do
    # ビジネスグループから注文の関連を取得
    allow(business_group).to receive(:orders).and_return(orders_relation)
    # チェック済みの注文のみを取得
    allow(orders_relation).to receive(:checked).and_return(checked_orders)
  end
  ```

- **ルール**: `it`ブロック内の設定（Arrange）と検証（Assert）部分には、何をしているのかがパッと見でわかるコメントを追加する
- **理由**: 長いテストコードを論理的なセクションに分割し、各モックやスタブの目的と検証の意図を明確にするため
- **例**:
  ```ruby
  it 'returns orders data with empty bookings' do
    # このテストでは、注文モード（'order'）で以下を確認する:
    # 1. 指定された月（2021年1月）の注文データのみが返される
    # 2. 予約データは空になる
    # 3. 注文データに対してIbBank::Order.zipが呼ばれる
    
    # 2021年1月の注文データに対するフィルタクエリをモック
    allow(controller).to receive(:build_filter_query)
      .with('order', 'order_2021_1')
      .and_return({created_at: Date.new(2021, 1, 1)..Date.new(2021, 1, 31).end_of_day})
    
    # メソッドを実行
    result = controller.send(:get_operated_data, business_group, 'order', 'order_2021_1')
    
    # 注文データのみが設定され、予約データは空になる
    expect(result[:business_group_orders]).to eq(paginated_orders)
    expect(result[:orders_count]).to eq(5)
    expect(result[:business_group_bookings]).to eq([])
    expect(result[:bookings_count]).to eq(0)
  end
  ```

### コメントの書き方

- **ルール**: 「正しい」「正しく」などの曖昧な表現は避け、具体的に何を期待しているかを説明する
- **理由**: テストは「正しさ」を検証するものなので、「正しい」は自明で情報量が少ないため
- **例**:
  - 悪い例：「戻り値が正しい構造と値を持つことを確認」
  - 良い例：「戻り値がハッシュで、期待するキーを含むことを確認する」

- **ルール**: 体言止めは避け、文末は「〜する」「〜である」などの表現にする
- **理由**: 「〜する」「〜である」などの表現は、期待する振る舞いや状態をより明確に伝えるため
- **例**:
  - 悪い例：「注文データの確認」
  - 良い例：「注文データのみが設定され、予約データは空になる」

### コメントの粒度とバランス

- **ルール**: すべての`expect`にコメントを付けるのではなく、関連する検証をグループ化してコメントを付ける
- **理由**: 情報の密度を最適化し、テストの論理構造を明確にするため
- **例**:
  ```ruby
  # 戻り値がハッシュで、期待するキーを含むことを確認する
  expect(result).to be_a(Hash)
  expect(result.keys).to contain_exactly(:business_group_orders, :business_group_bookings, :orders_count, :bookings_count)
  ```

- **ルール**: テストの可読性とDRYのバランスを考慮し、明確さを優先する
- **理由**: テストは仕様の文書化でもあり、各テストが独立して理解できることが重要なため
- **例**: 参照実装として `spec/controllers/users/business_groups_controller_spec.rb` を参照
