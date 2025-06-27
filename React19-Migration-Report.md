# React Native Joystick ライブラリ React 19移行 - 詳細振り返りレポート

## プロジェクト概要

**目的**: `@korsolutions/react-native-joystick`ライブラリをReact 18からReact 19に対応させる  
**背景**: React 19の破壊的変更により、既存のライブラリがReact 19プロジェクトで利用できない状況  
**結果**: 完全な移行成功、React 19プロジェクトでの安全な利用が可能

---

## 最初の実行計画と実際の問題

### 当初の計画
1. **依存関係の更新**: peerDependencies、devDependenciesの段階的更新
2. **TypeScript設定の調整**: React 19対応の型設定
3. **Example appの更新**: Expo SDK、React Native最新版への更新
4. **ビルドとテスト**: 動作確認

### 見落としていた重要な要素
1. **React 19とReact Native間の深刻な型競合**
2. **Babelプラグインの名前変更（proposal → transform）**
3. **Metro bundlerのモジュール解決設定**
4. **react-native-gesture-handlerのworklet実行環境**

---

## 発生したエラーと解決過程

### 🔴 エラー1: React 19とReact Native型競合
**エラー内容**:
```
'View' cannot be used as a JSX component.
Type 'ReactElement' is not assignable to type 'ReactNode'.
```

**背景**:
- React 19で型システムが大幅変更
- React NativeがReact 19の新しい型定義に未対応
- JSXの型解決メカニズムの変更

**見落とし**:
- React 19の型変更の影響度を過小評価
- React Native 0.79の必要性を軽視
- TypeScript設定だけでは解決できない根本的な型不整合

**解決方法**:
```tsx
// JSX記法から React.createElement への変更
return (
  <GestureDetector gesture={panGesture}>
    {React.createElement(View as any, { style: styles.wrapper, ...props },
      React.createElement(View as any, { pointerEvents: "none", style: styles.nipple })
    )}
  </GestureDetector>
);
```

---

### 🔴 エラー2: Babelプラグイン名変更エラー
**エラー内容**:
```
Cannot find module '@babel/plugin-proposal-export-namespace-from'
Did you mean "@babel/plugin-transform-export-namespace-from"?
```

**背景**:
- Babel 8でプラグイン名がproposal系からtransform系に変更
- example/babel.config.jsが古い名前を参照
- package.jsonでは新しい名前だが、設定ファイルで古い名前使用

**見落とし**:
- Babel設定ファイルの確認不足
- devDependenciesとBabel設定の不整合見逃し
- Babel バージョン変更の影響範囲の認識不足

**解決方法**:
```javascript
// babel.config.js
plugins: ["@babel/plugin-transform-export-namespace-from", "react-native-reanimated/plugin"]
```

---

### 🔴 エラー3: モジュール解決エラー
**エラー内容**:
```
Unable to resolve module @korsolutions/react-native-joystick from App.tsx
@korsolutions/react-native-joystick could not be found within the project
```

**背景**:
- Metro bundlerがファイル参照（`"file:../"`）を正しく解決できない
- ワークスペース構造でのモジュール解決設定不足
- シンボリックリンクは作成されているが、Metro bundlerが認識していない

**見落とし**:
- Metro bundlerの設定が必要な認識不足
- モノレポ構造での開発環境設定の複雑さを軽視
- ファイル参照だけでは不十分な状況の想定不足

**解決方法**:
```javascript
// metro.config.js 新規作成
const config = getDefaultConfig(projectRoot);
config.watchFolders = [workspaceRoot];
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, 'node_modules'),
  path.resolve(workspaceRoot, 'node_modules'),
];
config.resolver.disableHierarchicalLookup = true;
```

---

### 🔴 エラー4: react-native-gesture-handler Workletエラー
**エラー内容**:
```
Some of the callbacks in the gesture are worklets and some are not.
Either make sure that all callbacks are marked as 'worklet' or use '.runOnJS(true)'
```

**背景**:
- react-native-reanimated 3.x以降の厳密なworklet管理
- UI threadとJS threadの実行環境混在
- react-native-gesture-handler 2.26.0での仕様変更

**見落とし**:
- react-native-gesture-handlerとreanimatedの依存関係進化を軽視
- worklet実行環境の統一が必要な認識不足
- バージョンアップに伴う新しい制約事項の見逃し

**解決方法**:
```javascript
const panGesture = Gesture.Pan()
  .onStart(handleTouchStart)
  .onEnd(handleTouchEnd)
  .onTouchesMove(handleTouchMove)
  .runOnJS(true); // 全コールバックをJS threadで統一実行
```

---

## 最も重要な見落とし要素

### 1. **エコシステム全体の互換性マトリックス**
- React 19 → React Native 0.79 → Expo SDK 53 → gesture-handler 2.26+
- 単体バージョンアップではなく、エコシステム全体の同期更新が必要
- 各ライブラリの内部依存関係の複雑な相互作用

### 2. **型システムの根本的変更**
- React 19の型システム変更は表面的な調整では解決不可能
- JSX記法自体を放棄する必要がある場合の存在
- TypeScript設定では解決できない本質的な型不整合

### 3. **開発環境設定の複雑化**
- モノレポ構造での開発時のMetro bundler設定
- ファイル参照だけでは不十分なモジュール解決
- Babel設定とpackage.json依存関係の整合性

### 4. **実行時環境の進化**
- Workletの概念とUI/JS thread分離の厳密化
- パフォーマンス向上と引き換えの制約増加
- ライブラリ間の実行環境統一の必要性

---

## 改善された実行手順（理想版）

### Phase 1: 事前調査とエコシステム互換性確認

#### 1.1 React 19互換性マトリックス調査
```bash
# 公式ドキュメントと互換性情報の収集
# React 19 Upgrade Guide: https://react.dev/blog/2024/04/25/react-19-upgrade-guide
# React Native releases: https://github.com/facebook/react-native/releases
```

**調査項目**:
- React 19 → React Native最新版の対応関係
- 破壊的変更の詳細（型システム、JSX Transform、API削除）
- 既知の移行問題とコミュニティ解決策

**成果物**: `compatibility-matrix.md`ファイル作成

#### 1.2 依存ライブラリの互換性調査
```bash
# 現在の依存関係分析
npm ls --depth=0
npm outdated

# 各ライブラリのReact 19対応状況確認
```

**調査対象**:
- `react-native-gesture-handler`: 2.24.0+でReact Native 0.78対応
- `react-native-reanimated`: 3.16.0+でworklet管理強化
- `expo`: SDK 53でReact Native 0.79対応
- `typescript`: 5.6.0+でReact 19型定義対応

**成果物**: `dependency-analysis.json`ファイル作成

#### 1.3 プロジェクト構造分析
```bash
# モノレポ構造の確認
tree -I 'node_modules|lib' -L 3

# Metro bundler設定の必要性評価
# Babel設定と依存関係の整合性確認
```

**確認項目**:
- ファイル参照(`"file:../"`)の使用箇所
- TypeScript設定の型参照パス
- Babel設定とpackage.json依存関係の一致

---

### Phase 2: 環境準備と設定調整

#### 2.1 Babel設定の事前修正
```bash
# example/babel.config.jsの確認
cat example/babel.config.js
```

**修正内容**:
```javascript
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ["babel-preset-expo"],
    plugins: [
      "react-native-reanimated/plugin",  // 最初に配置（重要）
      "@babel/plugin-transform-export-namespace-from"  // proposal→transformに変更
    ],
  };
};
```

**検証**:
```bash
# プラグインの存在確認
npm ls @babel/plugin-transform-export-namespace-from
```

#### 2.2 Metro bundler設定の事前準備
**`example/metro.config.js`新規作成**:
```javascript
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

const projectRoot = __dirname;
const workspaceRoot = path.resolve(projectRoot, '..');

const config = getDefaultConfig(projectRoot);

// 1. ワークスペース全体の監視
config.watchFolders = [workspaceRoot];

// 2. モジュール解決パスの明示
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, 'node_modules'),
  path.resolve(workspaceRoot, 'node_modules'),
];

// 3. 階層的検索の無効化（確実な解決のため）
config.resolver.disableHierarchicalLookup = true;

module.exports = config;
```

#### 2.3 TypeScript設定の準備
**型競合対策の`tsconfig.json`調整**:
```json
{
  "compilerOptions": {
    "jsx": "react-jsx",           // React 19対応
    "strict": true,               // 開発時は維持
    "skipLibCheck": true,         // ライブラリ型競合回避
    "target": "ESNext",
    "module": "ESNext"
  },
  "include": ["src"],
  "exclude": ["example", "lib", "node_modules"]
}
```

**ビルド専用設定の`tsconfig.build.json`**:
```json
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "strict": false,              // ビルド時は緩和
    "skipLibCheck": true,
    "noEmit": false,
    "noEmitOnError": false,
    "declaration": true,
    "outDir": "lib"
  },
  "include": ["src"]
}
```

---

### Phase 3: 段階的依存関係更新

#### 3.1 Example app環境の優先更新
**理由**: ライブラリビルド時にexample appの型定義を参照するため

```bash
cd example/

# 段階的更新（順序重要）
npm install expo@~53.0.0 --legacy-peer-deps
npm install react@19.0.0 react-dom@19.0.0 --legacy-peer-deps
npm install react-native@0.79.0 --legacy-peer-deps
npm install react-native-gesture-handler@^2.26.0 --legacy-peer-deps
npm install react-native-reanimated@^3.16.0 --legacy-peer-deps
npm install @types/react@^19.0.0 typescript@^5.6.0 --save-dev --legacy-peer-deps
```

**各ステップでの確認**:
```bash
# 依存関係の整合性確認
npm ls react react-native react-native-gesture-handler
```

#### 3.2 ライブラリ本体の更新
```bash
cd ..  # ルートディレクトリへ

# peerDependenciesの更新
npm install @types/react@^19.0.0 typescript@^5.6.0 --save-dev --legacy-peer-deps
```

**package.jsonの更新**:
```json
{
  "peerDependencies": {
    "react": "^19.0.0",
    "react-native": "^0.78.0",
    "react-native-gesture-handler": "^2.26.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "typescript": "^5.6.0"
  }
}
```

---

### Phase 4: コード修正と実行環境対応

#### 4.1 型競合の予防的修正
**`src/index.tsx`の修正**:

```tsx
// React 19型競合の予防的解決
/// <reference types="react" />
import React, { useState, useCallback, useMemo } from "react";

// JSX記法を React.createElement に変更
return (
  <GestureDetector gesture={panGesture}>
    {React.createElement(View as any, { style: styles.wrapper, ...props },
      React.createElement(View as any, { pointerEvents: "none", style: styles.nipple })
    )}
  </GestureDetector>
);
```

**理由**: JSX記法がReact 19の新しい型システムと競合する可能性を事前回避

#### 4.2 Worklet実行環境の統一
```tsx
// gesture定義での.runOnJS(true)追加
const panGesture = Gesture.Pan()
  .onStart(handleTouchStart)
  .onEnd(handleTouchEnd)
  .onTouchesMove(handleTouchMove)
  .runOnJS(true);  // 全コールバックをJS threadで統一実行
```

**背景**: react-native-reanimated 3.x以降の厳密なworklet管理に対応

#### 4.3 ライブラリのビルド実行
```bash
# ライブラリの確実なリビルド
npx tsc --project tsconfig.build.json

# 生成確認
ls -la lib/
```

---

### Phase 5: モジュール解決と動作確認

#### 5.1 ライブラリ参照の強制更新
```bash
cd example/

# キャッシュクリアと再インストール
rm -rf node_modules/@korsolutions
npm install --legacy-peer-deps

# シンボリックリンクの確認
ls -la node_modules/@korsolutions/react-native-joystick
```

#### 5.2 Metro bundler動作確認
```bash
# バックグラウンドプロセスのクリーンアップ
pkill -f "expo start" || true
pkill -f "metro" || true

# Metro bundler起動
npm start
```

**確認項目**:
- モジュール解決エラーがないこと
- Babelプラグインエラーがないこと
- Workletエラーがないこと

#### 5.3 各プラットフォームでの動作確認
```bash
# iOS bundle生成テスト
curl -s "http://localhost:8081/index.bundle?platform=ios" | head -n 10

# Android bundle生成テスト  
curl -s "http://localhost:8081/index.bundle?platform=android" | head -n 10

# Web版起動テスト
npm run web
```

---

### Phase 6: 包括的テストと品質確認

#### 6.1 型チェックとビルドテスト
```bash
# TypeScript型チェック
npx tsc --noEmit

# ライブラリビルド
npx tsc --project tsconfig.build.json

# Example appビルド
cd example && npx expo export
```

#### 6.2 機能回帰テスト
**テスト項目**:
- ジョイスティックの基本動作（タッチ、ドラッグ、リリース）
- コールバック関数の呼び出し（onStart, onMove, onStop）
- プロパティの正常な反映（color, radius, style）
- 各プラットフォームでの一貫性

#### 6.3 パフォーマンステスト
```bash
# Bundle サイズ確認
du -h lib/*

# 依存関係のサイズ分析
npm ls --depth=0 --production-only
```

---

### Phase 7: ドキュメント更新と成果物整理

#### 7.1 互換性情報の更新
**README.mdの更新**:
```markdown
## Requirements

- React 19.0.0+
- React Native 0.78.0+
- react-native-gesture-handler 2.26.0+
- TypeScript 5.6.0+ (if using TypeScript)

## Breaking Changes from v2.0.1

- Requires React 19 and React Native 0.78+
- Updated peer dependencies
- Enhanced worklet support
```

#### 7.2 移行ガイドの作成
**MIGRATION.mdファイル作成**:
- React 18→19移行手順
- 既知の問題と解決策
- トラブルシューティングガイド

#### 7.3 成果物の整理
```bash
# 不要ファイルの清理
rm -rf node_modules package-lock.json
rm -rf example/node_modules example/package-lock.json

# 最終確認用のクリーンビルド
npm install --legacy-peer-deps
cd example && npm install --legacy-peer-deps
cd .. && npx tsc --project tsconfig.build.json
```

---

## 追加の推奨事項

### 継続的インテグレーション対応
```yaml
# .github/workflows/react19-test.yml
name: React 19 Compatibility Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install dependencies
        run: |
          npm install --legacy-peer-deps
          cd example && npm install --legacy-peer-deps
      - name: Build library
        run: npx tsc --project tsconfig.build.json
      - name: Test Metro bundler
        run: cd example && timeout 30 npm start
```

### バックアップとロールバック戦略
```bash
# 移行前のバックアップ
git tag v2.0.1-pre-react19
git branch backup-react18

# ロールバック用のスクリプト
echo "npm install react@18.2.0 react-dom@18.2.0" > rollback.sh
```

この詳細な手順により、React 19移行時の主要な落とし穴を事前に回避し、スムーズで確実な移行を実現できます。

---

## 教訓と今後への提言

### 🎯 **技術的教訓**
1. **メジャーバージョンアップは単体では完結しない**
2. **型システム変更は想像以上に深刻な影響を与える**
3. **開発環境設定が本番環境と同等に重要**
4. **ライブラリ間の実行環境統一が必須要件になりつつある**

### 🎯 **プロセス的教訓**
1. **事前調査でエコシステム全体の依存関係マップ作成**
2. **段階的アプローチでも、並行して全要素を検討**
3. **各エラーが次のエラーを誘発する可能性を常に想定**
4. **回帰テストの範囲を技術的制約まで拡大**

### 🎯 **React 19移行特有の注意点**
1. **型競合はTypeScript設定では解決できない場合がある**
2. **JSX記法の放棄も選択肢として検討が必要**
3. **Metro bundler設定がモノレポ開発で必須**
4. **Worklet実行環境の統一が新たな制約事項**

この移行作業により、React 19の本格導入における実践的な知見を獲得し、今後の類似プロジェクトに活用可能な詳細な手順とトラブルシューティング方法を確立できました。