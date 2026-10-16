# GPO で Firefox の自動更新を配る手順(コンピューターの構成)

Active Directory のグループポリシー(GPO)で、ドメインに参加している Windows 11 の Firefox に「自動更新を有効にする」設定を配る手順です。
**コンピューターの構成(コンピューターポリシー)** で配ります。ユーザーごとの設定は使いません。

この手順書は、実際に検証環境を 1 から作って再現し、その画面を撮ったものです(2026-10-16 JST)。

## 結論(3行)

- Mozilla が配っている管理用テンプレート(ADMX)をドメインの**中央ストア**に置くと、GPO エディターに `Mozilla > Firefox` が出る。
- `Application Autoupdate` を**有効**にした GPO を、対象 PC が入っている OU にリンクする。
- 配った先では、レジストリ `HKLM\SOFTWARE\Policies\Mozilla\Firefox` の `AppAutoUpdate` が `1` になり、Firefox の `about:policies` に `AppAutoUpdate = true` と出る。

## 検証環境(この手順を再現した構成)

| 項目 | 内容 |
|---|---|
| ハイパーバイザー | Proxmox VE 9.1.1 の仮想マシン 2 台 |
| ドメインコントローラー | Windows Server 2025 Standard 評価版(デスクトップエクスペリエンス)、ホスト名 `LABDC01`、固定 IP `<DCのIPアドレス>` |
| ドメイン | `lab.local`(NetBIOS 名 `LAB`、機能レベル Windows Server 2025) |
| クライアント | Windows 11 Pro 25H2(ビルド 26200)、ホスト名 `LABWIN11`、ドメイン参加済み |
| 対象 OU | `OU=Workstations,DC=lab,DC=local`(`LABWIN11` はこの中) |
| Firefox | 158.0(64 ビット、日本語、MSI 版を `https://download.mozilla.org/?product=firefox-msi-latest-ssl&os=win64&lang=ja` から入手) |
| ADMX テンプレート | Mozilla `policy-templates` v8.4(`https://github.com/mozilla/policy-templates/releases`) |
| 操作に使ったアカウント | DC: `LAB\Administrator`(ドメイン管理者)/ クライアント確認: `LAB\labuser`(一般ユーザー) |

あなたの環境に置き換えるときは、`lab.local` を自社のドメイン名、`Workstations` を対象 PC の OU 名に読み替えてください。

## 前提

- ドメインコントローラーに、ドメイン管理者(または GPO を作成できる権限)でサインインできる。
- 対象の PC がドメインに参加していて、**対象の OU に入っている**(OU に入っていない PC には、リンクしても配られません)。
- Firefox は 75 以降、または ESR 68.7 以降(この設定が使えるバージョン。GPO エディターの説明欄に出ます)。

## 手順 1. Firefox の管理用テンプレート(ADMX)を中央ストアに置く

GPO エディターで Firefox の項目を出すには、Mozilla のテンプレートをドメインの共有領域(SYSVOL)に置きます。

1. ブラウザーで `https://github.com/mozilla/policy-templates/releases` を開き、最新の `policy_templates_vX.Y.zip` をダウンロードして解凍する。
2. 解凍した `windows` フォルダーに、次のものがある。
   - `firefox.admx`、`mozilla.admx`(ポリシーの定義)
   - 言語ごとのフォルダー(`en-US` など)。この中に `firefox.adml`、`mozilla.adml`(画面に出る文字)がある。

   ![解凍した windows フォルダー](images/01-admx-source.png)

3. DC 上の次のフォルダーを開く(なければ作る)。
   `C:\Windows\SYSVOL\domain\Policies\PolicyDefinitions`
   (ネットワーク上では `\\lab.local\SYSVOL\lab.local\Policies\PolicyDefinitions`)
4. 2 の `firefox.admx` と `mozilla.admx` を、この `PolicyDefinitions` に直接コピーする。
5. 言語フォルダー(`en-US`)の `*.adml` を、`PolicyDefinitions\en-US` にコピーする。

   ![中央ストア(PolicyDefinitions)に置いた状態](images/02-admx-central-store.png)

> ⚠️ **本番で最初に中央ストアを作る場合は、先に `C:\Windows\PolicyDefinitions` の中身(Windows 標準のテンプレート)を `PolicyDefinitions` に丸ごとコピーしておく。**
> 中央ストアができると、GPO エディターは中央ストアの中だけを読みます。今回の検証環境のように Mozilla のファイルしか置かないと、「管理用テンプレート」に Windows 標準の項目が出なくなります(後の手順 3 の画面で `Mozilla` と `すべての設定` しか見えないのはこのためです)。

> ℹ️ v8.4 のテンプレートには日本語(`ja-JP`)がありません。そのため GPO エディターのポリシー名と説明は**英語**で表示されます(今回の画面のとおり)。

## 手順 2. GPO を作り、対象の OU にリンクする

1. DC にドメイン管理者でサインインする。

   ![DC のサインイン画面(LAB\Administrator)](images/03-dc-signin.png)

2. サーバーマネージャーの右上 **[ツール] > [グループ ポリシーの管理]** を開く。

   ![サーバーマネージャーのツールメニュー](images/04-servermanager-tools.png)

3. 左のツリーを **フォレスト > ドメイン > lab.local** と開き、対象の OU(ここでは `Workstations`)を見つける。

   ![グループ ポリシーの管理のツリー](images/05-gpmc-tree.png)

4. OU を右クリックし、**[このドメインに GPO を作成し、このコンテナーにリンクする]** を選ぶ。

   ![OU の右クリックメニュー](images/06-ou-context-menu.png)

5. 名前に `Firefox-AutoUpdate` と入れて **[OK]**。ソース スターター GPO は `(なし)` のまま。

   ![新しい GPO のダイアログ](images/07-new-gpo-dialog.png)

6. OU の下に `Firefox-AutoUpdate` ができ、右側の「リンクされたグループ ポリシー オブジェクト」に `リンクの有効化: はい`、`GPO の状態: 有効` で出る。

   ![GPO が作成されて OU にリンクされた状態](images/08-gpo-created-linked.png)

## 手順 3. コンピューターの構成で自動更新を有効にする

1. 左のツリーで作った GPO を右クリックし、**[編集]** を選ぶ。

   ![GPO の右クリックメニュー](images/09-gpo-context-edit.png)

2. グループ ポリシー管理エディターが開く。**[コンピューターの構成] > [ポリシー]** を開くと、`ソフトウェアの設定`、`Windows の設定`、`管理用テンプレート` の 3 つがある。

   ![コンピューターの構成のポリシー](images/10-editor-computer-policies.png)

3. **[管理用テンプレート]** を開く。見出しが「管理用テンプレート: セン…(中央ストア)」になり、その下に `Mozilla` が出る(手順 1 が効いている証拠)。

   ![管理用テンプレートに Mozilla が出ている](images/11-editor-admx-mozilla.png)

4. **[Mozilla] > [Firefox]** を選ぶ。右側にポリシーが 107 個並ぶ(フォルダーが先、設定は ID 順)。
   更新に関する設定は、一覧の上のほうにある。
   - `Application Autoupdate`(**今回有効にするもの**)
   - `Pin updates to a specific version`、`Custom Update URL`
   - `Background updater`
   - `Disable Update`(有効にすると**更新が止まる**ので触らない)

   ![Firefox のポリシー一覧](images/12-firefox-policy-list.png)

5. `Application Autoupdate` をダブルクリックし、**[有効]** を選んで **[OK]**。
   説明欄の要点: 有効 = ユーザーの承認なしに自動更新される / 無効 = 更新は取得するが、いつ入れるかはユーザーが選ぶ / 未構成 = ユーザーが選べる。

   ![Application Autoupdate を有効にする](images/13-autoupdate-enabled.png)

   > 小さい画面(1280×800 など)だと、このダイアログの [OK] ボタンが画面の下にはみ出します。その場合は **Enter キー**で確定できます。

6. 一覧で `Application Autoupdate` の状態が **有効** になっていれば完了。エディターを閉じる(保存は自動)。

   ![状態が有効になった](images/14-autoupdate-status-enabled.png)

   - `Background updater` は**未構成のまま**にしてあります。説明欄のとおり、未構成は「有効」と同じ扱い(Firefox が起動していないときも、承認なしで更新を入れてよい)です。
7. グループ ポリシーの管理で GPO を選び、**[スコープ]** タブで次を確かめる。
   - リンク先が `lab.local/Workstations`、リンクの有効化が `はい`。
   - セキュリティ フィルターが `Authenticated Users`(OU 内のすべてのコンピューターに適用)。

   ![GPO のスコープ](images/15-gpo-scope.png)

## 手順 4. クライアントで配られたことを確かめる

対象 PC(`LABWIN11`)で確認します。コンピューターポリシーは、PC の起動時と、その後 90 分(前後 30 分のゆらぎ)ごとに適用されます。すぐ試すときは次のとおり。

1. 管理者の PowerShell で `gpupdate /force` を実行する(または PC を再起動する)。
2. 適用された GPO を確認する。
   ```powershell
   gpresult /scope computer /r
   ```
   「適用されたグループ ポリシー オブジェクト」に `Firefox-AutoUpdate` が出る。
3. レジストリを確認する。
   ```powershell
   Get-ItemProperty "HKLM:\SOFTWARE\Policies\Mozilla\Firefox" | Select AppAutoUpdate
   ```
   `AppAutoUpdate` が `1`(DWORD)。
4. ドメインのユーザー(ここでは `LAB\labuser`)でサインインする。

   ![Windows 11 のサインイン(ドメインのユーザー)](images/16-win11-signin-domain-user.png)

5. Firefox のアドレス欄に `about:policies` と入れる。**「有効」**の一覧に `AppAutoUpdate = true` が出る。
   (Firefox はポリシーを起動時に読むので、すでに起動していた場合は再起動する。)

   ![about:policies に AppAutoUpdate = true](images/17-firefox-about-policies.png)

6. Firefox の **[設定] > [Firefox について]** を開く。上部に「**ご使用のブラウザーはあなたの所属組織に管理されています**」と出て、「Firefox の更新」に現在のバージョン(158.0)と更新の状態が出る。

   ![Firefox について(組織に管理されている表示)](images/18-firefox-about-update.png)

## 補足

### 設定値と意味(Mozilla のテンプレートの説明どおり)

| ポリシー(エディター上の名前) | 値 | 意味 |
|---|---|---|
| Application Autoupdate(`AppAutoUpdate`) | 有効 | 承認なしで自動更新する |
| Background updater(`BackgroundAppUpdate`) | 未構成(既定) | Firefox が起動していなくても更新を入れてよい |
| Disable Update(`DisableAppUpdate`) | 有効にしない | 有効にすると更新が止まる |
| Manual Update Only(`ManualAppUpdateOnly`) | 有効にしない | 有効にすると、手動の更新確認だけになる |

### 元に戻す・止める

- 自動更新をやめる(ユーザーに選ばせる): `Application Autoupdate` を **未構成** に戻す。更新自体を止めたいときは `Disable Update` を有効にする。
- GPO ごとやめる: グループ ポリシーの管理で OU の下の GPO を右クリックし、**[リンクの有効化]** のチェックを外す(GPO を残したまま配布だけ止める)。
- 配った先の反映: 次回のポリシー更新(`gpupdate /force`)でレジストリ値が消える。Firefox は再起動で反映。

### うまくいかないとき

| 症状 | 確認すること |
|---|---|
| GPO エディターに `Mozilla` が出ない | `firefox.admx` と `mozilla.admx` が `...\Policies\PolicyDefinitions` の直下にあるか。`*.adml` が `PolicyDefinitions\en-US` にあるか(対応する言語フォルダーがないとエラーになる)。 |
| クライアントに `Firefox-AutoUpdate` が出ない | PC が対象の OU に入っているか(`Get-ADComputer -Filter * \| ft Name,DistinguishedName`)。GPO のリンクが有効か。DNS がドメインコントローラーを向いているか。 |
| レジストリはあるのに `about:policies` に出ない | Firefox を完全に終了して起動し直す。 |
| Windows 標準の管理用テンプレートが消えた | 中央ストアに Mozilla のファイルしか置いていない。`C:\Windows\PolicyDefinitions` の内容を中央ストアにコピーする。 |

## この検証で確かめたことと、確かめていないこと

**確かめたこと**
- ADMX を中央ストアに置いたあと、GPO エディターに `Mozilla > Firefox` が出て、`Application Autoupdate` を有効にできる(画面で確認)。
- DC 側の GPO のレジストリ設定に `Software\Policies\Mozilla\Firefox` の `AppAutoUpdate = 1`(DWORD)が入っている(`Get-GPRegistryValue` で確認)。
- クライアントで `gpupdate /force` 後、`gpresult` に `Firefox-AutoUpdate` が適用済みで出て、レジストリ `AppAutoUpdate = 1` になり、Firefox の `about:policies` に `AppAutoUpdate = true` が出る。

**確かめていないこと**
- **実際に新しいバージョンへ自動更新される動作。** 検証でインストールしたのが最新版(158.0)だったため、更新が走る場面を作れていません。古い版を入れて、更新が自動で入ることを確かめるのが次の検証です。
- **Mozilla Maintenance Service の有無による違い。** 今回の MSI(既定のオプション)では、クライアントに `MozillaMaintenance` サービスは入っていませんでした(`Get-Service MozillaMaintenance` で見つからない)。サービスなしでの自動更新で、UAC の確認が出るかどうかは未確認です。
- 設定画面の「Firefox の更新」で、自動更新の選択肢がユーザーに見えなくなるかどうか(設定前後の比較はしていません)。
- ESR 版や、日本語のテンプレート表示(Mozilla 提供の v8.4 には日本語がない)。

## 付録. 同じことを PowerShell で行う(GUI の代わり)

画面操作の代わりに、DC の管理者 PowerShell で同じ設定ができます(`lab.local` / `Workstations` は読み替え)。
手順 1 の ADMX 配置のあとで実行します。使い捨ての GPO で動作を確かめてあり、GUI で作った GPO と同じレジストリ値(`AppAutoUpdate = 1`)になります。

```powershell
# GPO を作って OU にリンク
$ou = 'OU=Workstations,DC=lab,DC=local'
New-GPO -Name 'Firefox-AutoUpdate' | New-GPLink -Target $ou

# コンピューターの構成: Application Autoupdate = 有効(レジストリ値 AppAutoUpdate = 1)
Set-GPRegistryValue -Name 'Firefox-AutoUpdate' -Key 'HKLM\Software\Policies\Mozilla\Firefox' `
  -ValueName AppAutoUpdate -Type DWord -Value 1

# 確認
Get-GPRegistryValue -Name 'Firefox-AutoUpdate' -Key 'HKLM\Software\Policies\Mozilla\Firefox'
```

検証に使ったスクリプト一式は、このフォルダーの `scripts/` にあります(DC の昇格、OU とテストユーザーの作成、ADMX の取得と配置、Win11 の Firefox 導入とドメイン参加)。
