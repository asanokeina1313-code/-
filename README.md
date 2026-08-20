# 旅の栞 (Tabi no Shiori) — Travel Album

Google Apps Script版「Travel Album」を、GitHubで管理できる独立したWebアプリとして
作り直したものです。GASやGoogle Sheets/Driveへの依存をなくし、
**Firebase（認証・データベース・画像保存）＋ Google Maps API** で動作します。

## 元アプリからの変更点

| 機能 | 元アプリ (GAS) | 新アプリ |
|---|---|---|
| ホスティング | Google Apps Script Webアプリ | GitHub Pages / Firebase Hosting（静的サイト） |
| ユーザー認証 | 独自実装（名前+平文パスワードをスプレッドシートに保存） | Firebase Authentication（メール/パスワード、安全にハッシュ化） |
| データ保存 | Google Sheets | Firestore（NoSQLデータベース） |
| 写真保存 | Google Drive | Firebase Storage |
| 地図・住所検索 | Google Apps Script Maps Service + Leaflet | Google Maps JavaScript API（Geocoding + Places） |
| 周辺のおすすめ | Gemini API（AIによる提案文生成） | **Google Places Nearby Search**（実在店舗の評価・住所を取得。AI呼び出し不要） |
| 周遊ルート作成 | Gemini APIで順序を提案 | 最近傍法（nearest neighbor）で自動的に効率の良い順序に並べ替え |

> AIによる自然文でのおすすめ（「〇〇周辺のグルメを教えて」等）が欲しい場合は、
> `js/places.js` の `nearbySearch` の呼び出し部分を Gemini API 呼び出しに差し替えることで
> 追加できます。設計上、置き換えやすいように分離してあります。

## セットアップ手順

### 1. Firebaseプロジェクトを作成

1. https://console.firebase.google.com/ で新規プロジェクトを作成
2. 「Authentication」→「Sign-in method」で **メール/パスワード** を有効化
3. 「Firestore Database」を作成（本番モードでOK。ルールは後述のものを使用）
4. 「Storage」を有効化
5. 「プロジェクトの設定」→「マイアプリ」→ ウェブアプリを追加し、`firebaseConfig` を取得

取得した値を `js/firebase-config.js` に貼り付けます。

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "...",
};
```

### 2. Google Maps APIキーを取得

1. https://console.cloud.google.com/ でプロジェクトを作成（Firebaseと同じプロジェクトでも可）
2. 「APIとサービス」で以下を有効化:
   - **Maps JavaScript API**
   - **Places API**
   - **Geocoding API**
3. APIキーを発行し、`index.html` 内の2箇所の `YOUR_GOOGLE_MAPS_API_KEY` を置き換え
4. 課金アカウントの設定が必要です（無料枠内でも個人利用なら十分収まることが多いです）
5. 本番運用時はAPIキーにHTTPリファラー制限をかけてください

### 3. セキュリティルールを反映

Firebaseコンソールの Firestore / Storage の「ルール」タブに、それぞれ
`firestore.rules` / `storage.rules` の内容を貼り付けて公開してください。
（Firebase CLIを使う場合は `firebase deploy --only firestore:rules,storage:rules`）

### 4. GitHubにプッシュ

```bash
cd travel-album
git init
git add .
git commit -m "Initial commit: 旅の栞"
git branch -M main
git remote add origin https://github.com/YOUR_NAME/YOUR_REPO.git
git push -u origin main
```

### 5. デプロイ（どちらか）

**GitHub Pages の場合**
- リポジトリの Settings → Pages → Source を `main` ブランチ / ルートに設定するだけで公開されます

**Firebase Hosting の場合**
```bash
npm install -g firebase-tools
firebase login
firebase init hosting   # 既存の firebase.json を使う場合はスキップ可
firebase deploy
```

## ローカルでの確認

Firebase/Google Maps はどちらも `http://localhost` からのアクセスを許可できるので、
簡易サーバーで動作確認できます。

```bash
npx serve .
# または
python3 -m http.server 8000
```

## フォルダ構成

```
travel-album/
├── index.html          画面全体のHTML
├── css/style.css        デザイン（パスポート／スタンプをモチーフにしたテーマ）
├── js/
│   ├── firebase-config.js   Firebase初期化（要: 自分の設定値に書き換え）
│   ├── auth.js               ログイン・新規登録
│   ├── places.js             Geocoding・周辺検索（Google Maps）
│   ├── map.js                 地図表示・マーカー
│   └── app.js                  データCRUD・画面描画のメインロジック
├── firestore.rules      Firestoreのアクセス権限
├── storage.rules         Storageのアクセス権限
└── firebase.json          Firebase Hosting設定（任意）
```

## データモデル（Firestore）

```
users/{uid}
  displayName, createdAt

users/{uid}/places/{placeId}
  place, lat, lng, note, status(planned|visited),
  date, tripTitle, prefecture, imageUrls[], createdAt
```

## 未実装・今後の拡張候補

- AIによる自然文のおすすめ（Gemini API連携）
- 複数ユーザーでのアルバム共有
- PWA化（オフライン対応・ホーム画面への追加）
- Google Maps Directions APIを使った実際の経路案内表示
