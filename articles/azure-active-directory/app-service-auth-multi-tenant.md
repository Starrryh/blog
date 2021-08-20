---
title: App Service 認証を利用して複数のテナントの Azure AD ユーザーをサインインさせる方法
date: 2021-06-30
tags:
  - Azure AD
  - App Service
---

# App Service 認証を利用して複数のテナントの Azure AD ユーザーをサインインさせる方法

こんにちは、Azure & Identity サポート チームの高田です。今回は、App Service 認証を利用して複数のテナントの Azure AD ユーザーを App Service にサインインさせる方法について案内いたします。

まず、App Service は Web サイトや REST API など HTTP ベースのサービスを公開するための Azure のサービスです。例えば Visual Studio で開発した ASP.NET Core アプリケーションを Azure を用いて公開したい場合、Visual Studio の Publish 機能を用いることで非常に簡単に実現できます。またお客様によっては、そのようにして公開した Web サイトに対し認証機能を組み込みたい場合もあると思います。この場合に活用いただけるのが App Service 認証です (Easy Auth という名称で呼ばれることもあります)。

以下では、App Service 認証を利用して複数の Azure AD テナントのユーザーを App Service 上の Web サイトにサインインさせる方法をご紹介します。Web サイトの公開範囲ごとにシナリオを分けて解説していきます。

- シナリオ 1: App Service の Web サイトを自テナントのユーザーだけに公開したい
- シナリオ 2: 主に自テナントのユーザー向けに提供している App Service の Web サイトを別テナントの一部ユーザーにもアクセスさせたい
- シナリオ 3: App Service の Web サイトを複数の Azure AD テナントのユーザーにアクセスさせたい

以下にシナリオごとに順に解説していきます。以下のシナリオは、全て ASP.NET Core Web アプリケーションなどを Visual Studio などから App Service に Publish 済みであるということを前提とします。App Service へ ASP.NET Core アプリケーションを展開する方法については、[Azure App Service に ASP.NET Core アプリを展開する](https://docs.microsoft.com/ja-jp/aspnet/core/host-and-deploy/azure-apps/?view=aspnetcore-5.0&tabs=visual-studio) などのページをご覧ください。

## シナリオ 1: App Service の Web サイトを自テナントのユーザーだけに公開したい

まず最も基本的なパターンとして、App Service のサイトを自テナントのユーザーだけに公開したい場合を考えます。この場合、必要なのは、App Service 認証の設定のみです。App Service 認証の設定を行うだけで、アプリへのアクセス時に Azure AD での認証が求められるようになります。また認証が求められた際には、そのアプリが登録されている Azure AD テナントのユーザーのみがサインイン可能です。

App Service を構成するため以下の手順を進めます。以下の手順は [チュートリアル: Azure App Service で実行されている Web アプリに認証を追加する](https://docs.microsoft.com/ja-jp/azure/app-service/scenario-secure-app-authentication-app-service#configure-authentication-and-authorization) でも解説がありますので、よろしければ併せてご参照ください。

1. Auzre ポータルに App Service の管理者でサインインし、App Service を開きます。
2. App Service 認証を構成したい Web アプリを選びます。
3. [認証] を選択して、[ID プロバイダーを追加] を選択します。
4. [ID プロバイダー] として [Microsoft] を選択します。
5. [アプリの登録の種類] が [アプリの登録を新規作成する] であることを確認します。
6. [サポートされているアカウントの種類] が [現在のテナント - 単一テナント] であることを確認します。
7. [認証] が [認証が必要] に設定されていることを確認します。
8. [追加] を選択します。

![](./app-service-auth-multi-tenant/screen1.png)

以上で操作は可能です。Web サイトの URL にアクセスいただくと、Azure AD の認証画面が表示されるようになります。上記操作を行った Azure AD テナントのユーザーを指定すれば、アプリにサインインできます。

なお、ユーザーが App Service 認証を経由して Web アプリにアクセスすると、Web アプリには以下のような値を含む HTTP ヘッダーが渡されます。Web アプリはこのヘッダーの値を確認することで、どのようなユーザーがサインインしてきたかを識別可能です。詳細については [Azure App Service 上での認証と承認の高度な使用方法 - ユーザー要求へのアクセス](https://docs.microsoft.com/ja-jp/azure/app-service/app-service-authentication-how-to#access-user-claims) もご参照ください。

| HTTP ヘッダーの名前 | ヘッダーの値 (例) |
| ------ | ------|
| X-MS-CLIENT-PRINCIPAL-NAME | test@contoso.com |
| X-MS-CLIENT-PRINCIPAL-ID | bfef01fb-9b32-457a-b488-690de651bf7f |

## シナリオ 2: 主に自テナントのユーザー向けに提供している App Service の Web サイトを別テナントの一部ユーザーにもアクセスさせたい

このシナリオの場合、Azure AD が提供する B2B ゲスト招待機能を利用することが適切です。App Service 認証はシナリオ 1 と同様に単一テナントとして構成し、自テナントに外部テナントのユーザーをゲスト招待します。外部テナントのユーザーを都度招待するということが手間でなければ (事前にこのユーザーは App Service の Web サイトアクセスさせたいというのがわかっていれば)、Azure AD の観点でも適切な方法になります。

上述のとおり、このシナリオの場合は App Service 認証の観点では特に構成変更の必要はありません。シナリオ 1 に加えて、以下の Azure AD 側の対応 (ゲストの招待操作) を実施いただければと思います。

1. Auzre ポータルにアクセスします。
2. App Service を構成しているテナントに Azure AD のグローバル管理者やゲスト招待元のディレクトリ ロールを持つユーザーでサインインします。
3. Azure Active Directory ブレードに進み、[ユーザー] を開きます。
4. [+ 新しいゲスト ユーザー] を押します。
5. 招待したい外部校テナントのユーザーの E メール アドレス (test@fabrikam.com) を入力して、招待ボタンを押します。
6. 指定されたユーザーで招待メールが届いているかを確認します。
7. 招待メールを受け取ったユーザーは、メールに従い招待の処理を完了します。

以上の操作を行うことで、招待処理を完了したユーザーは、App Service を構成している Azure AD テナントにゲストとして Azure AD アカウントが作成されます。これにより、そのゲスト ユーザーは App Service を構成している Azure AD テナント上のユーザーとして認識されます。このため App Service 認証の設定が単一テナントのままでも、その Web サイトにサインインが可能になります。招待していないユーザーで Web サイトにアクセスしようとするとそのユーザーはアクセスがブロックされます。

## シナリオ 3: App Service の Web サイトを複数の Azure AD テナントのユーザーにアクセスさせたい

上記シナリオ 2 では、App Service の Web サイトに外部のユーザーをアクセスさせるにあたり個別にゲスト ユーザーの招待が必要でした。個別に招待を行うのではなく、複数の Azure AD テナントのユーザーに Web サイトへのアクセスを許可したい場合は、App Service 認証を単一テナントではなく、マルチテナントとして構成します。これにより、どの Azure AD テナントからでもその Web サイトにアクセスさせることができるようになります。

まず、既に単一テナントで App Service 認証を構成している場合は、以下の手順に沿って既存の構成を一旦削除ください。

1. Auzre ポータルに App Service の管理者でサインインし、App Service を開きます。
2. App Service 認証を構成したい Web アプリを選びます。
3. [認証] を選択して、[認証の設定] の隣にある [編集] を選択します。
4. [認証されていないアクセスを許可する] を選択して、[保存] を押下します。
5. [ID プロバイダー] から Microsoft を探して、削除のアイコンをクリックします。
6. 確認画面で [削除] を選択します。

続いて、マルチテナントとして構成するため ID プロバイダーを追加しなおします。

1. [ID プロバイダーを追加] を選択します。
2. [ID プロバイダー] として [Microsoft] を選択します。
3. [アプリの登録の種類] が [アプリの登録を新規作成する] であることを確認します。
4. [サポートされているアカウントの種類] で [任意の Azure AD ディレクトリ - マルチテナント] を選択します。
5. [認証] が [認証が必要] に設定されていることを確認します。
6. [追加] を選択します。

なお、[追加] を押下した際に、同じ名前のアプリケーションが既に存在するメッセージが表示されるかと思います。これは先に単一テナントで App Service 認証を構成したときのアプリケーション オブジェクトが Azure AD 側に残っているためです。メッセージを回避するには名前を別のものに変更するか、以下の手順で Azure AD から該当のアプリケーション オブジェクトを削除して再度同じ名前で追加を実施ください。

1. Azure Active Directory ブレードを開き、[アプリの登録] を開きます。
2. 検索ボックスにアプリケーションの名前を入力します。
3. 結果が表示されたらそれをクリックします。
4. [削除] を選択します。
5. 確認が求められたら [削除] を再度押下します。

ID プロバイダーの追加が完了したら、さらに以下の操作を行います。

1. [ID プロバイダー] から Microsoft を探して、編集のアイコンをクリックします。
2. [発行者の URL] を <https://login.microsoftonline.com/common/v2.0> に変更します。
3. [保存] を選択します。

![](./app-service-auth-multi-tenant/screen2.png)

以上で操作は完了です。

Azure AD ではアプリケーション オブジェクト側の構成とログオン時の URL の指定内容によって、どのユーザーがアプリケーションにサインイン可能かを制御しています。元々上記 [発行者の URL] にはテナント ID を含む値が指定されていました。この場合、アプリケーションがマルチテナントで構成されていても、そのアプリにサインインできるのはその指定されたテナント上のユーザーのみとなります。発行者の URL を <https://login.microsoftonline.com/common/v2.0> に書き換えると、URL としてもマルチテナントを受け入れるようになるため、複数のテナントのユーザーからサインインが可能となります。

なお、上記状態では構成したとおり様々な Azure AD テナントから Web アプリにアクセスが可能となります。もしアプリを特定の数テナントだけにアクセスさせたいという場合は、アプリケーション側で X-MS-CLIENT-PRINCIPAL-NAME ヘッダーの値を確認し、サインインしたユーザーのドメイン名が意図したものでなければアクセスを拒否するなどの対応をご検討ください。

## シナリオ 4: App Service の Web サイトを、App Service を構成したテナントとは別のテナントのユーザーにアクセスさせたい

App Service に紐づくサブスクリプションがあるテナントと、ユーザーを管理しているテナントが異なる場合などに、ご利用いただける方法です。ここでご紹介する手順は、[Azure AD ログインを使用するように App Service または Azure Functions アプリを構成する - 個別に作成された既存の登録を使用する](https://docs.microsoft.com/ja-jp/azure/app-service/configure-authentication-provider-aad#use-an-existing-registration-created-separately) の公開情報でも解説がございます。

**テナント構成の前提**
- テナント A: App Service の Web サイトをホストしているテナント
- テナント B: App Service にアクセスするユーザーが存在しているテナント

**テナント B 上での作業**
1. テナント B の Azure ポータルにサインインし、Azure Active Directory ブレードを開き、[アプリの登録] を開きます。
2.  [+ 新規登録] を選択し、各項目を下記のように設定し、[登録] を押下します。
    - [名前] に任意の名前を設定します。
    - [サポートされているアカウント] の種類は [この組織ディレクトリのみに含まれるアカウント (シングル テナント)] を選択します。
    - [リダイレクト URI] では [Web] を選択し、"<App Service の URL>/.auth/login/aad/callback" と入力します  
    (例: "https://contoso.azurewebsites.net/.auth/login/aad/callback")
3. アプリの登録完了したら、 [アプリケーション (クライアント) ID] と [ディレクトリ (テナント) ID] を控えておきます。
4. 左端メニューから [認証] タブを選択します。 [暗黙的な許可およびハイブリッド フロー] で "ID トークン" を有効にします。
5. (省略可能) 左端メニューから [ブランド] で [ホーム ページ URL] に App Service アプリの URL を入力して [保存] を選択します。
6. 左端メニューから [API の公開] を選択します。
7. [アプリケーション ID の URI] に App Service アプリの URL を入力し、[保存] を選択します。
8. 左端メニューから [証明書とシークレット] を選択します。
9. [新しいクライアント シークレット] を選択し、クライアント シークレットを追加します。  

なお、クライアント シークレットの追加後は、画面に表示されるクライアント シークレットの値を必ずお控えください。一度この画面を離れたら二度と表示されません。クライアント シークレットの有効期限は最長 2 年間です。有効期限が切れた場合は更新の対応が必要です。

**テナント A 上での作業**
1. テナント A の Azure ポータルにサインインし、Web アプリをホストしている App Service の画面を開きます。
2. 左側のメニューで [認証] を選択します。 [ID プロバイダーの追加] をクリックします。
3. [ID プロバイダー] ドロップダウンで [Microsoft] を選択し、下記のように設定します。
    - [アプリの登録の種類] では [既存アプリの登録の詳細を提供します] を選択します。
    - [アプリケーション (クライアント) ID] には、前述の手順で控えた [アプリケーション (クライアント) ID] を設定します。
    - [クライアント シークレット] には前述の手順で控えたクライアント シークレットの値を設定します。
    - [発行者の URL] には "https://login.microsoftonline.com/<テナント B のテナント ID>/v2.0" を設定します。  
    テナント B のテナント ID は、前述の手順で控えた [ディレクトリ (テナント) ID] です。
4. これまで Web アプリに他の ID プロバイダーがされていない場合は、[App Service 認証設定] のセクションが表示されます。
    - [認証] は "認証が必要" を選択します。
    - [認証されていない要求] では、[HTTP 302 リダイレクトが見つかりました: Web サイトに推奨] を選びます。  
    この設定は、アプリケーションが認証されていない要求にどのように応答するかを決定します。基本的には上記設定で問題ありません。
5. [追加] をクリックします。 

設定手順は以上です。

App Service のアプリの URL にアクセスすることで、Azure AD の認証画面が表示されます。ここからサインインできるのは、テナント B 上のユーザー (テナントユーザー、テナント B に招待されたゲストユーザー) のみです。他のテナントのユーザーの資格情報でサインインを試行すると、エラーが表示されて失敗します。