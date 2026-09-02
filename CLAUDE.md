# football-broadcast-2026-27

2026-27シーズンの欧州5大リーグ＆トッテナムの放送チャンネル・試合日程をまとめた静的サイト（ビルドツールなし、素のHTML/CSSのみ）。GitHub Pagesで配信。

- `index.html` — 欧州5大リーグ＆日本サッカーの放送チャンネル一覧
- `tottenham.html` — トッテナム全試合日程・放送チャンネル
- `tottenham.ics` — トッテナムの試合日程カレンダー（購読用）
- `analysis/` — トッテナム試合の詳細分析ページ（本ドキュメントの対象）

デザインは各HTMLファイルに`<style>`を直書きする方式（共通CSSファイルなし）。ダークモード対応はCSS変数＋`prefers-color-scheme`＋`data-theme`属性の3段構成（既存ファイルからコピーする）。

## 試合分析システム（analysis/）

**目的**：トッテナムの毎試合終了後に、スタッツ・戦術分析をメインにした詳細な振り返りページを作る。

**運用方針**：完全自動化はせず、ユーザーが試合ごとに「この試合を分析して」と依頼したときに、以下の手順で生成する（Routine化は将来の拡張候補）。

### データソース：API-Football

- 無料枠（1日100リクエスト）で開始。トッテナム単体なら1試合あたり数リクエストで足りる規模。
- ベースURL: `https://v3.football.api-sports.io`
- 認証ヘッダー: `x-apisports-key: <APIキー>`
- **APIキーは絶対にリポジトリにコミットしない。** ユーザーがチャットで渡すか、セッションの環境変数として渡してもらい、その場でAPI呼び出しに使うだけにする（ファイルに書き出さない）。

### 手順

1. **チームIDの解決**（初回のみ、または不明な場合）
   `GET /teams?search=Tottenham` でトッテナムの内部team IDを取得する。IDを推測でハードコードしない。一度確認できたら、このファイルに追記して以後使い回してよい。
   - トッテナムのteam ID: `（未確認・初回実行時にここへ記入する）`

2. **対象試合の特定**
   `GET /fixtures?team={team_id}&season=2026&date={YYYY-MM-DD}` またはユーザーが指定した対戦カードで該当fixture IDを特定する。ステータスが試合終了（`FT`）であることを確認する。

3. **データ取得**（fixture IDが分かったら）
   - `GET /fixtures?id={fixture_id}` — スコア、日時、スタジアム、審判、大会情報
   - `GET /fixtures/statistics?fixture={fixture_id}` — 両チームのスタッツ（支配率、シュート数・枠内数、パス数・成功率、コーナー、ファウル、オフサイド、カード枚数、GKセーブ数など）
   - `GET /fixtures/lineups?fixture={fixture_id}` — フォーメーション・スタメン・控え・監督
   - `GET /fixtures/events?fixture={fixture_id}` — 得点・カード・交代の時系列イベント
   - `GET /fixtures/players?fixture={fixture_id}` —（任意）選手個人スタッツ・採点。総括で触れたい場合に使う。

4. **戦術分析の執筆**
   取得したスタッツ・フォーメーション・交代タイミングを踏まえて、以下を文章化する（単なる数値の言い換えではなく、試合展開の解釈を書く）：
   - 両チームの基本布陣とプレスタイル（ハイプレスか撤退守備か、ボール保持志向か等）
   - 試合の流れが変わったポイント（先制点、退場、采配など）
   - 交代采配の意図と効果
   - 次節に向けた注目点

5. **ページ生成**
   - ファイル名: `analysis/YYYY-MM-DD-{home-slug}-{away-slug}.html`（スラッグは英小文字・ハイフン区切り。例: `analysis/2026-08-30-tottenham-newcastle.html`）
   - 下記テンプレート構造をコピーして中身を埋める（CSS変数・カードスタイルは`tottenham.html`と統一する）
   - `analysis/index.html`の`.match-list`内、先頭（最新試合が上）に`match-card`を追加する。1件でも追加したら`.empty-state`ブロックは削除する
   - `tottenham.html`の該当行の「カード」列に「 · `<a href="analysis/xxx.html">分析</a>`」を追記する（該当試合が消化済みになったタイミング）

### ページテンプレート構造

`analysis/index.html`と同じCSS変数（`--bg`, `--card-bg`, `--text`, `--accent`, `--win`/`--draw`/`--loss`等）を土台に、以下のカードを上から順に並べる。

```html
<!-- ヘッダー: 大会名・日時・スタジアム・スコア -->
<header class="hero">
  <div class="hero-inner">
    <a class="back" href="index.html">← 分析一覧に戻る</a>
    <h1>トッテナム 2-1 ニューカッスル</h1>
    <p>プレミアリーグ 第2節｜2026年8月30日（日本時間）｜トッテナム・ホットスパー・スタジアム</p>
  </div>
</header>

<main>
  <!-- カード1: 試合結果サマリー（得点者・カード） -->
  <div class="card"><h3>試合結果</h3> ... </div>

  <!-- カード2: スタッツ比較（横棒で両チーム比較） -->
  <div class="card">
    <h3>スタッツ比較</h3>
    <div class="stat-row">
      <span class="stat-val">58%</span>
      <div class="stat-bar"><div class="stat-bar-fill home" style="width:58%"></div></div>
      <span class="stat-label">ボール支配率</span>
      <div class="stat-bar"><div class="stat-bar-fill away" style="width:42%"></div></div>
      <span class="stat-val">42%</span>
    </div>
    <!-- シュート数、枠内シュート、パス成功率、コーナー、ファウル、オフサイド等を同形式で繰り返す -->
  </div>

  <!-- カード3: フォーメーション・スタメン -->
  <div class="card"><h3>フォーメーション・スタメン</h3> ... </div>

  <!-- カード4: 戦術分析（文章） -->
  <div class="card"><h3>戦術分析</h3> ... </div>

  <!-- カード5: タイムライン（得点・カード・交代） -->
  <div class="card"><h3>タイムライン</h3>
    <ul class="timeline">
      <li><span class="min">23'</span> ⚽ ソン・フンミン（アシスト: マディソン）</li>
    </ul>
  </div>

  <!-- カード6: 総括・次節注目点 -->
  <div class="card"><h3>総括</h3> ... </div>
</main>

<footer>
  <p>本ページは公開情報（API-Football）をもとに作成した非公式の分析です。戦術評価には主観的な解釈を含みます。</p>
  <p>リポジトリ: <a href="https://github.com/chimoke-hub/football-broadcast-2026-27" target="_blank" rel="noopener">chimoke-hub/football-broadcast-2026-27</a></p>
</footer>
```

新しいCSSが必要なコンポーネント（`.stat-row` / `.stat-bar` / `.stat-bar-fill` / `.timeline` / `.formation-grid`）は、既存ページの`.card`まわりのスタイルに揃えて都度追加する。すでに1ページ作成済みならそのファイルの`<style>`をコピーして使い回す。
