---
title: ExpoアプリをEASではなくApp Distributionに自動配布し費用を抑える方法
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["expo", "reactnative", "firebase", "githubactions"]
published: true
---

React Nativeでアプリを開発するときにほとんどのケースでExpoを利用すると思います。ExpoはReact Nativeの開発をサポートするためのフレームワークで、開発体験がとてもいいです。
しかし、Expoで作成したアプリをテスト用などに配布する場合、Expoの公式サービスであるEASを利用することができますが、無料プランだとアプリのビルド数に制限があり、1ヶ月に**30ビルドまで**となっています。

https://expo.dev/pricing

30ビルドは、CI/CDを導入しているとすぐに使い切ってしまうことがあるため、無料でアプリを配布する方法を探していました。そこで、FirebaseのApp Distributionを利用することで、無料でアプリを配布することができることを知りました。

https://firebase.google.com/docs/app-distribution?hl=ja

この記事では、ExpoのアプリをApp Distribution経由で配布する方法を紹介します。App DistributionはFirebaseの機能の一つで、アプリの配布を行うことができます。無料プランでもアプリの配布が可能で、ビルド数に制限がないため、Expoのアプリを無料で配布することができます。

## Expoの設定

まずは、Expo側の設定を行います。
App Distributionで配布しますがビルドのためにEASも利用します。

https://expo.dev/

Expoのコンソールにアクセスし、プロジェクトを作成します。

### tokenの取得

次に、管理画面よりtokenを取得しておきます。

https://expo.dev/settings/access-tokens

## ローカルでの作業

次に、ローカルでの作業を行います。
ローカルではeas-cliを使ってプロジェクトをExpoの管理画面で設定したそれとあわせておきましょう

まずはExpoアプリを作成するために、以下のコマンドを実行します。

```bash
$ npx create-expo-app@latest
```

これでExpoアプリの雛形が作成されます。

次に、Expoのアプリをeas-cliでビルドするために、eas-cliをインストールします。

```bash
$ npm install -g eas-cli
$ eas login
$ eas build:configure
```

`eas build:configure`コマンドを実行すると、`app.config.js`ファイルが自動で作成されます。

### Android用のkeystoreおよび、iOS用の証明書の設定

アプリをビルドするには、Android用のkeystoreおよび、iOS用の証明書が必要です。それぞれの設定を行います。
Expoでは、`eas credentials`コマンドを使ってAndroid用のkeystoreおよび、iOS用の証明書を設定することができます。
便利ですね！！

```bash
$ eas credentials
```

この状態で、`eas build --local`コマンドを実行すると、ローカルでアプリのビルドができるようになります。

```bash
$ eas build --local
```

### 環境変数周りの設定

また、Expoではローカルの開発に`.env.local`ファイルを使って環境変数を設定することができます。

```
FIREBASE_API_KEY=xxxx
FIREBASE_APP_ID=xxxx
```

このファイルはgitに含めないように.gitignoreに追加しておきましょう。

設定した環境変数は先ほど作成した`app.config.js`に記載しておきます。
`extra`という項目に自由に環境変数を追加することができます。

```js
export const expoConfig: ExpoConfig = {
  ios: {
    bundleIdentifier: "com.example.app",
    config: {
      googleServicesFile: "./GoogleService-Info.plist"
    }
  },
  android: {
    package: "com.example.app",
    config: {
      googleServicesFile: "./google-services.json"
    }
  },
  extra: {
    FIREBASE_API_KEY: process.env.FIREBASE_API_KEY,
    FIREBASE_DOMAIN,: process.env.FIREBASE_DOMAIN,
    FIREBASE_PROJECT_ID: process.env.FIREBASE_PROJECT_ID,
  }
}
```

設定した環境変数はアプリ内で参照することができます。

```js
import Constants from "expo-constants";

// Firebaseの設定
const firebaseConfig = {
  apiKey: Constants.expoConfig?.extra?.FIREBASE_API_KEY ?? "",
  authDomain: Constants.expoConfig?.extra?.FIREBASE_DOMAIN ?? "",
  projectId: Constants.expoConfig?.extra?.FIREBASE_PROJECT_ID ?? "",
};
```

## Firebaseプロジェクトの作成

つぎに、Firebaseプロジェクトを作成します。Firebaseのコンソールにアクセスし、新しいプロジェクトを作成します。

https://console.firebase.google.com/

次に、プロジェクトの設定に移動します。

![](https://storage.googleapis.com/zenn-user-upload/10e090f39138-20241208.png)

マイアプリより、iOS用とAndroid用のアプリを追加します。

![](https://storage.googleapis.com/zenn-user-upload/6f70037b7872-20241208.png)

それぞれのアプリのパッケージ名を入力します。

![](https://storage.googleapis.com/zenn-user-upload/c83ed825f509-20241208.png)

ここは、Expoのapp.config.jsに記載されている`ios.bundleIdentifier`と`android.package`を入力します。

```ts
export const expoConfig: ExpoConfig = {
  ...
  ios: {
      bundleIdentifier: "com.example.app"
  },
  android: {
    package: "com.example.app"
  } 
}
```

それぞれのアプリの設定ファイルをダウンロードしておきます。

## App Distributionの設定

次に、FirebaseのApp Distributionの設定を行います。Firebaseのコンソールにアクセスし、App Distributionを選択します。

![](https://storage.googleapis.com/zenn-user-upload/c4122ab14aca-20241208.png)

ここで、「開始」をクリックします。

![](https://storage.googleapis.com/zenn-user-upload/98bcb99c606e-20241208.png)

## GitHub Actionsの設定

最後に、GitHub Actionsを使ってアプリをApp Distributionに配布する設定を行います。

冒頭にも記載した通り、今回はExpoのアプリをExpoの管理画面上ではなく、FirebaseのApp Distributionを使って無料で配布するので、Expoのプロジェクトで環境変数を設定しておく方法は使えません。

ですので、先ほどローカルで設定した環境変数を**GitHub→Settings→Secrets→Actions**に設定しておきます。
取得したExpoのtokenも同様にSecretsに設定しておきます。

![](https://storage.googleapis.com/zenn-user-upload/d2897c97bb80-20241208.png)

## actionsの設定

プロジェクトでGitHub Actionsの設定ファイルを作成します。

僕は以下のようにAndroid用とiOS用のjobをわけています

::: details 各設定ファイル

```yaml:./.github/workflows/deploy.yml
name: Distribute Android and iOS Apps with App Distribution

on:
  workflow_dispatch:
  push:
    branches:
      - development

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-android:
    name: Build and Distribute Android App
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 2
      - name: setup common
        uses: ./.github/actions/setup
        env:
          EXPO_ACCESS_TOKEN: ${{ secrets.EXPO_ACCESS_TOKEN }}
      - name: create env file
        run: |
          touch .env
          echo "ANDROID_CLIENT_ID=${{ secrets.ANDROID_CLIENT_ID }}" >> .env
          echo "FIREBASE_API_KEY=${{ secrets.FIREBASE_API_KEY }}" >> .env
          echo "FIREBASE_APP_ID=${{ secrets.FIREBASE_APP_ID }}" >> .env
          echo "FIREBASE_DOMAIN=${{ secrets.FIREBASE_DOMAIN }}" >> .env
          echo "FIREBASE_PROJECT_ID=${{ secrets.FIREBASE_PROJECT_ID }}" >> .env
          echo "IOS_CLIENT_ID=${{ secrets.IOS_CLIENT_ID }}" >> .env
      - name: Build on EAS (Android)
        run: eas build --platform android --non-interactive --no-wait --local
        env:
          APP_VARIANT: development
      - name: Find APK file
        run: |
          APK_PATH=$(find ./ -name "*.apk" | head -n 1)
          echo "EAS_BUILD_OUTPUT_PATH=$APK_PATH" >> $GITHUB_ENV
      - name: Deploy to Firebase App Distribution (iOS)
        uses: ./.github/actions/distribute
        env:
          EAS_BUILD_OUTPUT_PATH: ${{ env.EAS_BUILD_OUTPUT_PATH }}
          FIREBASE_APP_ID_STAGING: ${{ secrets.FIREBASE_APP_ID_STAGING }}
          FIREBASE_SERVICE_CREDENTIALS_JSON: ${{ secrets.FIREBASE_SERVICE_CREDENTIALS_JSON }}
  build-ios:
    name: Build and Distribute iOS App
    runs-on: macos-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 2
      - name: setup common
        uses: ./.github/actions/setup
        env:
          EXPO_ACCESS_TOKEN: ${{ secrets.EXPO_ACCESS_TOKEN }}
      - name: create env file
        run: |
          touch .env
          echo "ANDROID_CLIENT_ID=${{ secrets.ANDROID_CLIENT_ID }}" >> .env
          echo "FIREBASE_API_KEY=${{ secrets.FIREBASE_API_KEY }}" >> .env
          echo "FIREBASE_APP_ID=${{ secrets.FIREBASE_APP_ID }}" >> .env
          echo "FIREBASE_DOMAIN=${{ secrets.FIREBASE_DOMAIN }}" >> .env
          echo "FIREBASE_PROJECT_ID=${{ secrets.FIREBASE_PROJECT_ID }}" >> .env
          echo "IOS_CLIENT_ID=${{ secrets.IOS_CLIENT_ID }}" >> .env
      - name: Build on EAS (iOS)
        run: eas build --platform ios --non-interactive --no-wait --local
        env:
          APP_VARIANT: development
      - name: Find IPA file
        run: |
          IPA_PATH=$(find ./ -name "*.ipa" | head -n 1)
          echo "EAS_BUILD_OUTPUT_PATH=$IPA_PATH" >> $GITHUB_ENV
      - name: Deploy to Firebase App Distribution (iOS)
        uses: ./.github/actions/distribute
        env:
          EAS_BUILD_OUTPUT_PATH: ${{ env.EAS_BUILD_OUTPUT_PATH }}
          FIREBASE_APP_ID_STAGING: ${{ secrets.FIREBASE_APP_ID_IOS_STAGING }}
          FIREBASE_SERVICE_CREDENTIALS_JSON: ${{ secrets.FIREBASE_SERVICE_CREDENTIALS_JSON }}
```

```yaml:./.github/actions/setup/action.yaml
name: "setup common"
description: "Setup common settings"
runs:
  using: composite
  steps:
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 20
        registry-url: "https://registry.npmjs.org"
    - name: Set up JDK
      uses: actions/setup-java@v3
      with:
        java-version: 17
        distribution: "zulu"
    - name: Setup Expo and EAS
      uses: expo/expo-github-action@v8
      with:
        eas-version: latest
        token: ${{ env.EXPO_ACCESS_TOKEN }}
    - name: Install dependencies
      run: npm ci
      shell: bash
```

```yaml:./.github/actions/distribute/action.yaml
name: "distribute"
description: "Distribute app to testers"
runs:
  using: composite
  steps:
    - name: Install Firebase CLI
      run: curl -sL https://firebase.tools | bash
      shell: bash
    - name:
      id: create-json
      uses: jsdaniell/create-json@v1.2.2
      with:
        name: "credentials.json"
        json: ${{ env.FIREBASE_SERVICE_CREDENTIALS_JSON }}
        dir: "."
    - name: Deploy to Firebase App Distribution
      env:
        GOOGLE_APPLICATION_CREDENTIALS: credentials.json
      run: |
        firebase appdistribution:distribute \
          ${{ env.EAS_BUILD_OUTPUT_PATH }} \
          --app ${{ env.FIREBASE_APP_ID_STAGING }} \
          --groups testers
      shell: bash
```
:::

これで、GitHub Actionsを使ってExpoのアプリをApp Distributionに配布する設定が完了しました。

## テスターの設定

次に、テスターの設定を行います。FirebaseのApp Distributionの管理画面からテスターを追加します。

![](https://storage.googleapis.com/zenn-user-upload/583e2e3075d0-20241208.png)

テスターを追加したあと、GitHub Actionsを実行します。

アプリがApp Distributionに配布されていることを確認します。

![](https://storage.googleapis.com/zenn-user-upload/fca8ec8f2134-20241208.png)

無事にアプリが配布されると、テスターに招待メールが送られます。

Androidの場合は、メールに記載されたリンクをクリックすることでアプリをインストールすることができます。
`App Tester`アプリがインストールされていることを確認してください。

![](https://storage.googleapis.com/zenn-user-upload/f6d5ac072c42-20241208.png =250x)

iOSの場合は、メールに記載されたリンクをクリックし「Install Firebase profile」をクリックします。
Profileをインストールすると設定アプリに「Firebase App Distribution」が追加されているので、そこからアプリをインストールすることができます。

アプリをインストールしたタイミングではまだ、「Firebase App Distribution」からアプリがダウンロードできません。


### Apple Developerでの設定

そこで、iOSの場合は、テスターのUDIDをApple Developerに登録する作業が必要です。
テスターがアプリをインストールすると、`App Distribution`より開発者にテスターのUDIDが届くので、そのUDIDをApple Developerに登録します。

https://developer.apple.com/account/resources/devices/list

次に作ったdeviceをもとにProvisioning Profileを作成します。
「AdHoc」のProvisioning Profileを作成し、ダウンロードします。

https://developer.apple.com/account/resources/profiles/list

### ExpoでのProvisioning Profileの設定

ダウンロードしたProvisioning ProfileはExpoのCredentialsより登録します。

![](https://storage.googleapis.com/zenn-user-upload/c908af8559d8-20241208.png)


### 再度GitHub Actionsを実行

再度GitHub Actionsを実行し、iOSのアプリをApp Distributionに配布します。
成功すると「ダウンロード」ボタンがApp Distributionアプリに表示されるので、そこからアプリをインストールすることができます。

![](https://storage.googleapis.com/zenn-user-upload/b70395c155ae-20241208.png =250x)


以上で、ExpoのアプリをApp Distributionに配布する方法を紹介しました。
無料プランでもアプリの配布が可能で、ビルド数に制限がないため、Expoのアプリを無料で配布することができます。

ただ、Expoでそのままアプリを配布する場合と比べるとはるかに手間がかかりますね。
予算に余裕がある場合は、EASをそのまま使ってアプリを配布することをおすすめします。

