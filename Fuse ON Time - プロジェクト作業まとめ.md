# Fuse ON Time - プロジェクト作業まとめ

## プロジェクト概要

**アプリ名:** Fuse ON Time（旧称: Touch On Time）
**種別:** モバイル勤怠管理アプリ（Expo / React Native）
**ストレージ:** ローカルAsyncStorage（サーバー不要）
**認証:** ロールベース（従業員 / 管理者）

元となった資料は「TouchOnTimeマニュアル（スマートフォン用）Ver.1.pdf」で、企業向け勤怠管理システムの従業員用スマートフォンアプリを再現し、さらに管理者ダッシュボードを追加したものです。

---

## 技術スタック

| 技術 | バージョン | 用途 |
|------|-----------|------|
| Expo SDK | 54 | モバイルアプリフレームワーク |
| React Native | 0.81.5 | UIフレームワーク |
| TypeScript | 5.9 | 型安全 |
| Expo Router | 6 | ファイルベースルーティング |
| NativeWind | 4 | Tailwind CSSスタイリング |
| AsyncStorage | 2.2 | ローカルデータ永続化 |
| expo-haptics | 15 | 触覚フィードバック |

---

## ファイル構成

```
touch-on-time/
├── app/
│   ├── _layout.tsx          # ルートレイアウト（AuthProvider + ロールベースルーティング）
│   ├── login.tsx            # ログイン画面
│   ├── (tabs)/              # 従業員用タブグループ
│   │   ├── _layout.tsx      # 4タブ構成（打刻/タイムカード/申請/メニュー）
│   │   ├── index.tsx        # 打刻画面（リアルタイム時計 + 4ボタン）
│   │   ├── timecard.tsx     # タイムカード（勤務実績 + 集計情報）
│   │   ├── applications.tsx # 各種申請メニュー + 申請フォーム
│   │   └── menu.tsx         # メニュー（打刻確認/締め申請/ログアウト）
│   └── (admin)/             # 管理者用タブグループ
│       ├── _layout.tsx      # 4タブ構成（ダッシュボード/ユーザー/勤怠/承認）
│       ├── index.tsx        # 管理者ダッシュボード（サマリーカード）
│       ├── users.tsx        # ユーザー管理（CRUD）
│       ├── attendance.tsx   # 勤怠確認（日別表示）
│       └── approvals.tsx    # 申請承認・却下
├── lib/
│   ├── store.ts             # データモデル + AsyncStorage CRUD
│   ├── auth-context.tsx     # 認証コンテキスト（ロール判定含む）
│   ├── theme-provider.tsx   # テーマプロバイダー
│   └── utils.ts             # ユーティリティ（cn関数）
├── components/
│   ├── screen-container.tsx # SafeArea対応スクリーンラッパー
│   └── ui/
│       └── icon-symbol.tsx  # アイコンマッピング
├── theme.config.js          # カラーパレット定義
├── app.config.ts            # アプリ設定（名前/バンドルID等）
├── design.md                # 設計書
└── todo.md                  # 機能管理リスト
```

---

## 実装済み機能一覧

### 従業員機能

| 機能 | 画面 | 説明 |
|------|------|------|
| ログイン | `/login` | ID/パスワード認証 |
| 打刻 | `/(tabs)/index` | 出勤/退勤/休憩開始/休憩終了の4ボタン打刻 |
| タイムカード | `/(tabs)/timecard` | 月別勤務実績一覧 + 集計情報 |
| スケジュール申請 | `/(tabs)/applications` | シフト登録・変更申請 |
| 打刻申請 | `/(tabs)/applications` | 打刻修正申請 |
| 交通費申請 | `/(tabs)/applications` | 補助項目（交通費）申請 |
| 打刻確認 | `/(tabs)/menu` | 当月の打刻履歴確認 |
| 締め申請 | `/(tabs)/menu` | 月末の勤怠確定処理 |

### 管理者機能

| 機能 | 画面 | 説明 |
|------|------|------|
| ダッシュボード | `/(admin)/index` | 従業員数/出勤者数/打刻数/未承認申請数 |
| ユーザー管理 | `/(admin)/users` | 従業員の追加/編集/削除 |
| 勤怠確認 | `/(admin)/attendance` | 日別の全従業員打刻状況 |
| 申請承認 | `/(admin)/approvals` | 各種申請の承認/却下 |

---

## データモデル

### エンティティ定義

```typescript
// ユーザー
interface User {
  id: string;           // ログインID（ユニーク）
  password: string;     // パスワード
  name: string;         // 表示名
  department: string;   // 部署
  role: "employee" | "admin";  // ロール
}

// 打刻レコード
type PunchType = "clock_in" | "clock_out" | "break_start" | "break_end";
interface PunchRecord {
  id: string;
  userId: string;
  date: string;         // YYYY-MM-DD
  type: PunchType;
  timestamp: string;    // ISO 8601
}

// スケジュール申請
interface ScheduleApplication {
  id: string;
  userId: string;
  type: "schedule";
  date: string;
  shiftType: string;    // 通常/早番/遅番/夜勤/全日休暇/公休
  startTime: string;    // HH:MM
  endTime: string;
  breakStartTime: string;
  breakEndTime: string;
  message: string;
  status: "pending" | "approved" | "rejected";
  createdAt: string;
}

// 打刻申請
interface PunchApplication {
  id: string;
  userId: string;
  type: "punch";
  date: string;
  punchType: PunchType;
  time: string;         // HH:MM
  reason: string;
  status: "pending" | "approved" | "rejected";
  createdAt: string;
}

// 交通費申請
interface ExpenseApplication {
  id: string;
  userId: string;
  type: "expense";
  date: string;
  transportType: string; // 電車/バス/タクシー/その他
  amount: number;
  destination: string;
  route: string;         // 例: "板浜駅→池袋駅"
  message: string;
  status: "pending" | "approved" | "rejected";
  createdAt: string;
}

// 月次締め
interface MonthlyClosing {
  id: string;
  yearMonth: string;    // YYYY-MM
  closedAt: string;
  status: "closed" | "open";
}
```

### AsyncStorageキー

| キー | 内容 |
|------|------|
| `fuse_on_time_users` | ユーザーリスト |
| `fuse_on_time_current_user` | ログイン中ユーザー |
| `fuse_on_time_punches` | 全打刻レコード |
| `fuse_on_time_applications` | 全申請レコード |
| `fuse_on_time_closings` | 月次締めレコード |
| `fuse_on_time_logged_in` | ログイン状態フラグ |

---

## 認証フロー

```
アプリ起動
  ↓
RootLayout (_layout.tsx)
  ↓ AuthProvider でラップ
  ↓ useRootNavigationState で Router 準備完了を待つ
  ↓
isAuthenticated?
  ├── No → /login に遷移
  └── Yes → isAdmin?
        ├── Yes → /(admin) に遷移
        └── No  → /(tabs) に遷移
```

### デフォルトアカウント

| 種別 | ID | パスワード | 名前 |
|------|-----|-----------|------|
| 管理者 | admin | admin123 | 管理者 |
| 従業員 | demo001 | password | 古澤 太郎 |
| 従業員 | demo002 | password | 田中 花子 |

---

## カラーパレット

```javascript
const themeColors = {
  primary:    { light: '#D32F2F', dark: '#E57373' },  // ブランドカラー（赤）
  background: { light: '#FFFFFF', dark: '#151718' },
  surface:    { light: '#F5F5F5', dark: '#1E2022' },
  foreground: { light: '#1A1A1A', dark: '#ECEDEE' },
  muted:      { light: '#687076', dark: '#9BA1A6' },
  border:     { light: '#E0E0E0', dark: '#333333' },
  success:    { light: '#4CAF50', dark: '#66BB6A' },
  warning:    { light: '#FF9800', dark: '#FFA726' },
  error:      { light: '#F44336', dark: '#EF5350' },
};

// 打刻ボタン専用カラー
// 出勤: #2196F3 (青)
// 退勤: #E91E63 (ピンク)
// 休憩開始: #4CAF50 (緑)
// 休憩終了: #FF9800 (オレンジ)
```

---

## 主要な実装パターン

### 1. リアルタイム時計（打刻画面）

```typescript
const [currentTime, setCurrentTime] = useState(new Date());
useEffect(() => {
  const timer = setInterval(() => setCurrentTime(new Date()), 1000);
  return () => clearInterval(timer);
}, []);
```

### 2. ロールベースルーティング（_layout.tsx）

```typescript
function RootNavigator() {
  const { isAuthenticated, isAdmin, isLoading } = useAuthContext();
  const segments = useSegments();
  const navigationState = useRootNavigationState();

  useEffect(() => {
    if (isLoading) return;
    if (!navigationState?.key) return; // Router準備完了を待つ

    if (!isAuthenticated) {
      router.replace("/login");
    } else if (isAdmin) {
      router.replace("/(admin)");
    } else {
      router.replace("/(tabs)");
    }
  }, [isAuthenticated, isAdmin, isLoading, navigationState?.key]);
}
```

### 3. 打刻処理（重複防止付き）

```typescript
const handlePunch = async (type: PunchType) => {
  const punches = await getPunchesByDate(today, user.id);
  const lastSameType = punches.filter(p => p.type === type).pop();
  if (lastSameType) {
    const diff = Date.now() - new Date(lastSameType.timestamp).getTime();
    if (diff < 60000) return; // 1分以内の重複防止
  }
  await addPunch(type);
  Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
};
```

### 4. 管理者の申請承認

```typescript
const handleApprove = async (appId: string) => {
  await updateApplicationStatus(appId, "approved");
  refreshApplications(); // リスト再取得
};
const handleReject = async (appId: string) => {
  await updateApplicationStatus(appId, "rejected");
  refreshApplications();
};
```

---

## 開発プロセス（再現手順）

1. **マニュアル分析** — PDFから画面構成・業務フロー・データ項目を抽出
2. **設計書作成** — `design.md` にスクリーン一覧、ユーザーフロー、カラーパレットを定義
3. **プロジェクト初期化** — Expo テンプレートで `webdev_init_project`
4. **テーマ設定** — `theme.config.js` にブランドカラーを設定
5. **データモデル実装** — `lib/store.ts` に全エンティティとCRUD関数を実装
6. **認証実装** — `lib/auth-context.tsx` でログイン/ログアウト/ロール判定
7. **ルーティング設定** — `app/_layout.tsx` でロールベースの自動遷移
8. **従業員画面実装** — 打刻、タイムカード、申請、メニューの4タブ
9. **管理者画面実装** — ダッシュボード、ユーザー管理、勤怠確認、承認の4タブ
10. **ブランディング** — アプリアイコン生成、アプリ名設定
11. **テスト** — Vitestでストア関数のユニットテスト
12. **スキル化** — `attendance-app-builder` スキルとして再利用可能な形にパッケージ

---

## スキル（再利用可能なプロセス）

作成したスキル: `/home/ubuntu/skills/attendance-app-builder/`

| ファイル | 内容 |
|---------|------|
| `SKILL.md` | ワークフロー、データモデル、画面構成、カラーパレット、実装パターン、チェックリスト |
| `references/data-model.md` | エンティティ詳細仕様、ストレージヘルパー、認証・管理者関数 |
| `references/screen-specs.md` | 全12画面のレイアウト・フィールド・ロジック仕様 |

---

## 今後の拡張候補

| 機能 | 説明 | 優先度 |
|------|------|--------|
| GPS打刻制限 | 指定勤務地でのみ打刻可能 | 高 |
| プッシュ通知 | 打刻忘れリマインダー | 高 |
| CSV/Excelエクスポート | 月次勤怠レポートのダウンロード | 中 |
| シフト管理 | 管理者によるシフト作成・割り当て | 中 |
| サーバー同期 | クラウドDBによるデータ同期 | 低 |
| 生体認証 | Face ID / 指紋によるログイン | 低 |

---

## 注意事項

- 現在のパスワード管理はプレーンテキスト（デモ用途）。本番運用にはハッシュ化が必要。
- AsyncStorageはデバイスローカルのため、アプリ削除でデータ消失。本番はサーバー同期を推奨。
- 管理者アカウントの初期パスワード（admin123）は運用開始前に変更すること。
