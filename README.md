# OneloginSamlDemo on AWS

https://developers.onelogin.com/saml/python

### Install Python(3.6.2)

Pythonをインストールするためのライブラリをインストール

```sh
sudo yum -y update

sudo yum -y install \
  git gcc zlib-devel libffi-devel sqlite-devel \
  bzip2-devel readline-devel openssl-devel
```

pyenvをインストール

```sh
git clone git://github.com/yyuu/pyenv.git ~/.pyenv

cat << 'EOT' >> ~/.bashrc

# pyenv
PATH=$PATH:${HOME}/.pyenv/bin
eval "$(pyenv init -)"
EOT

source ~/.bashrc
pyenv -v
  # pyenv 1.2.9-2-g6309aaf2
```

Pythonをインストール

```sh
pyenv install 3.6.8

pyenv versions
  # * system
  #   3.6.8 (set by /home/ec2-user/.pyenv/version)

pyenv global 3.6.8
python --version
  # Python 3.6.8
```

pipのアップグレード

```sh
pip install --upgrade pip

pip --version
  # pip 19.0.1 from /home/ec2-user/.pyenv/versions/3.7.2/lib/python3.7/site-packages/pip (python 3.7)
```

### Install Library for DemoApp

```sh
sudo yum -y install libxml2-devel xmlsec1-openssl-devel libtool-ltdl-devel

pip install dm.xmlsec.binding
pip install isodate
pip install defusedxml
pip install python3-saml
```

### Download DemoApp

```sh
git clone https://github.com/ot-nemoto/OneloginSamlDemo.git
cd OneloginSamlDemo/

pip install -r requirements.txt
```

### DemoApp Configuration

onelogin へ Administrator でログイン

- APPS > Add Apps > SAML Test Connector (IdP w/attr)
  - Display Name: (任意)
  - Save

saml/settings.json

```
export DEMO_APP_URL=http://$(curl -s http://169.254.169.254/latest/meta-data/public-hostname):8000
export IDP_ENTITY_ID=/* 下記参照 */
export IDP_SSO_SERVICE_URL=/* 下記参照 */
export IDP_SLO_SERVICE_URL=/* 下記参照 */
export IDP_X509CERT=/* 下記参照 */
```

##### IDP_ENTITY_ID

- (任意) > SSO > issuer URL

##### IDP_SSO_SERVICE_URL

- (任意) > SSO > SAML 2.0 Endpoint (HTTP)

##### IDP_SLO_SERVICE_URL

- (任意) > SSO > SLO Endpoint (HTTP)

##### IDP_X509CERT

- (任意) > SSO > X.509 Certificate > View Details
  - X.509 Certificate をコピーし、[Format a X.509 certificate](https://www.samltool.com/format_x509cert.php) を開き X.509 cert に貼り付け
  - FORMAT X.509 CERTIFICATE をクリック
  - X.509 cert in string format

```sh
cat saml/settings.json.org | envsubst > saml/settings.json
```

### onelogin Configuration

- (任意) > Configuration
  - Audience: `${DEMO_APP_URL}/metadata/`
  - Recipient: `${DEMO_APP_URL}/?acs`
  - ACS (Consumer) URL Validator: `.*`
  - ACS (Consumer) URL: `${DEMO_APP_URL}/?acs`
  - Single Logout URL: `${DEMO_APP_URL}/?sls`
  - Save

### Run Server

```
./manage.py runserver 0.0.0.0:8000
```

バックグラウンドで起動

```
nohup ./manage.py runserver 0.0.0.0:8000 > /dev/null 2>&1 < /dev/null &
```

# Login by SAML

### クライアントIDとクライアントシークレットの生成

https://developers.onelogin.com/api-docs/1/getting-started/working-with-api-credentials

onelogin へ Administrator でログイン

- DEVELOPERS > API Credentials > NEW CREDENTIAL
  - Name: (任意)
  - Credentials Scope: Manage All
  - Save

- `Client ID` および `Client Secret` が振られるので、環境変数に設定

```sh
CLIENT_ID=<Client ID>
CLIENT_SECRET=<Client Secret>
```

### アクセストークンの取得

https://developers.onelogin.com/api-docs/1/oauth20-tokens/generate-tokens

```sh
curl -s 'https://api.us.onelogin.com/auth/oauth2/token' \
  -X POST \
  -H "Authorization: client_id:${CLIENT_ID}, client_secret:${CLIENT_SECRET}" \
  -H "Content-Type: application/json" \
  -d '{ "grant_type":"client_credentials" }' | jq
```

Access Tokenのみ抽出

```sh
ACCESS_TOKEN=$(curl -s 'https://api.us.onelogin.com/auth/oauth2/token' \
  -X POST \
  -H "Authorization: client_id:${CLIENT_ID}, client_secret:${CLIENT_SECRET}" \
  -H "Content-Type: application/json" \
  -d '{ "grant_type":"client_credentials" }' | jq -r '.data[].access_token')
echo ${ACCESS_TOKEN}
  #
```

### セッショントークンの取得

https://developers.onelogin.com/api-docs/1/login-page/create-session-login-token

- `Login User or Email`: Onelogin User
- `Password`: Onelogin User Passeord
- `Onlogin SubDomain`: https://**\<SubDomain>**.onelogin.com/portal/

```sh
LOGIN_USER=<Login User or Email>
LOGIN_PASSWORD=<Password>
LOGIN_SUBDOMAIN=<Onlogin SubDomain>

curl -s 'https://api.us.onelogin.com/api/1/login/auth' \
  -X POST \
  -H "Authorization: bearer: ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{ \"username_or_email\":\"${LOGIN_USER}\", \"password\":\"${LOGIN_PASSWORD}\", \"subdomain\":\"${LOGIN_SUBDOMAIN}\" }" | jq
```

Session Tokenのみ抽出

```sh
export SESSION_TOKEN=$(curl -s 'https://api.us.onelogin.com/api/1/login/auth' \
  -X POST \
  -H "Authorization: bearer: ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{ \"username_or_email\":\"${LOGIN_USER}\", \"password\":\"${LOGIN_PASSWORD}\", \"subdomain\":\"${LOGIN_SUBDOMAIN}\" }" | jq -r '.data[].session_token')
echo ${SESSION_TOKEN}
  #
```

### ブラウザ経由でOnloginのポータルを表示

https://developers.onelogin.com/api-docs/1/login-page/create-session-via-token

```sh
cat << 'EOT' | envsubst > onelogin-demo.html
<!doctype html>
<html>
    <head>
        <meta charset="utf-8">
    </head>
    <body>
        <p>Auth API Test</p>
        <form action=
         "https://${LOGIN_SUBDOMAIN}.onelogin.com/session_via_api_token" method="POST">
            <input type="hidden" name="session_token" value="${SESSION_TOKEN}">
            <input type="submit" placeholder="GO">
            <input id="auth_token" type="hidden">
        </form>
    </body>
</html>
EOT
```

- Open `onelogin-demo.html` via browser

### ブラウザ経由でデモアプリのログイン後の画面を表示

https://developers.onelogin.com/api-docs/1/saml-assertions/generate-saml-assertion

- `App ID`:
  - onelogin へ Administrator でログイン
  - APP > Company Apps > \<App>
  - https://\<SubDomain>.onelogin.com/apps/**\<App ID>**/edit

```sh
APP_ID=<App ID>

curl -s "https://api.us.onelogin.com/api/1/saml_assertion" \
  -X POST \
  -H "Authorization: bearer: ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{ \"username_or_email\":\"${LOGIN_USER}\", \"password\":\"${LOGIN_PASSWORD}\", \"app_id\":\"${APP_ID}\", \"subdomain\":\"${LOGIN_SUBDOMAIN}\" }" | jq
```

SAML Responseのみ抽出

```sh
export SAML_RESPONSE=$(curl -s "https://api.us.onelogin.com/api/1/saml_assertion" \
  -X POST \
  -H "Authorization: bearer: ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{ \"username_or_email\":\"${LOGIN_USER}\", \"password\":\"${LOGIN_PASSWORD}\", \"app_id\":\"${APP_ID}\", \"subdomain\":\"${LOGIN_SUBDOMAIN}\" }" | jq -r '.data')
echo ${SAML_RESPONSE}
  #
```

```sh
cat << 'EOT' | envsubst > onelogin-demo.html
<!doctype html>
<html>
    <head>
        <meta charset="utf-8">
    </head>
    <body>
        <p>Auth API Test</p>
        <form action="${DEMO_APP_URL}?acs" method="POST">
            <input type="hidden" name="RelayState" value="${APP_URL}">
            <input type="hidden" name="SAMLResponse" value="${SAML_RESPONSE}">
            <input type="submit" placeholder="GO">
            <input id="auth_token" type="hidden">
        </form>
    </body>
</html>
EOT
```

- Open `onelogin-demo.html` via browser
