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

[onelogin](https://opentone.onelogin.com/admin) へ Administrator でログイン

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
nohup python apps/manage.py runserver 0.0.0.0:8000 > /dev/null 2>&1 < /dev/null &
```
