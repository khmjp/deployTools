<!-- TOC -->

- [やりたいこと](#やりたいこと)
- [試した環境](#試した環境)
- [（ツール作成サーバ）事前準備](#ツール作成サーバ事前準備)
    - [Python3のインストール](#python3のインストール)
    - [Gitのインストール](#gitのインストール)
- [（ツール作成サーバ）deployToolsの構築](#ツール作成サーバdeploytoolsの構築)
- [（ツール実行サーバ）deployToolsの配置](#ツール実行サーバdeploytoolsの配置)

<!-- /TOC -->

# やりたいこと
- オフライン環境でのサーバ構築で、以下のソフトウェアを使用したい。(以下、deployToolsと呼ぶ)
  - Ansible (構築)
  - Testinfra (試験)
  - JupyterLab (手順)
  - Git (バージョン管理)
- deployTools用に専用サーバを設けることが難しいので、業務サーバと兼用する形で配置するため、サーバへの影響を最小限としたい。
  - rpmインストールは最小限とする
  - 影響範囲を特定ユーザ（ansibleユーザなど）のみとしたい
- ディレクトリをアーカイブ（tgz）して、まるっと持っていけば動くようなものとしたい。
  - （ツール作成サーバ）でツールを作成し、アーカイブ
  - （ツール実行サーバ）でツールを展開し、実行
- 各ツールで必要なファイルや設定については、本稿では対象外とする。
  - （公開するには手間がかかるため）

# 試した環境
- ツール作成サーバ、ツール実行サーバともに以下
  - OSはRHEL7.5
  - 上記ツールは未導入
  - Python3は未導入

# （ツール作成サーバ）事前準備
## Python3のインストール

- jupyterlabを使用するにはPython3が必要となる。

INSTALLPATHおよびPYVER変数を設定
```
# インストール先のディレクトリ
INSTALLPATH="/home/ansible/local"
# インストールするPythonのバージョン
PYVER="3.7.4"
```

ビルドに必要なツール・ライブラリのインストール
```
yum groupinstall "development tools"
yum install bzip2-devel gdbm-devel libffi-devel \
  libuuid-devel ncurses-devel openssl-devel readline-devel \
  sqlite-devel tk-devel wget xz-devel zlib-devel
```

ソースコードのダウンロード
```
mkdir ${INSTALLPATH}
cd ${INSTALLPATH}
curl -O https://www.python.org/ftp/python/${PYVER}/Python-${PYVER}.tgz
tar xfz Python-${PYVER}.tgz
```

ビルド
```
cd Python-${PYVER}
./configure --prefix=${INSTALLPATH} --with-ensurepip
make
make install
```

バージョン確認
```
${INSTALLPATH}/bin/python3 --version
${INSTALLPATH}/bin/pip3 --version
```

ソースコードの削除
```
cd ../
rm -f Python-${PYVER}.tgz
rm -rf Python-${PYVER}
```

## Gitのインストール
INSTALLPATHおよびPYVER変数を設定
```
# インストール先のディレクトリ
INSTALLPATH="/home/ansible/local"
# インストールするGitのバージョン
GITVER="2.23.0"
```

ビルドに必要なツール・ライブラリのインストール
```
yum -y install gcc curl-devel expat-devel gettext-devel openssl-devel perl-devel zlib-devel
```

ソースコードのダウンロード
```
mkdir ${INSTALLPATH}
cd ${INSTALLPATH}
curl -O https://mirrors.edge.kernel.org/pub/software/scm/git/git-${GITVER}.tar.gz
tar xfz git-${GITVER}.tar.gz
```

ビルド
```
cd git-${GITVER}
make prefix=${INSTALLPATH} all
make prefix=${INSTALLPATH} install
```

バージョン確認
```
${INSTALLPATH}/bin/git --version
```

ソースコードの削除
```
cd ../
rm -f git-${GITVER}.tar.gz
rm -rf git-${GITVER}
```

# （ツール作成サーバ）deployToolsの構築

インストール先のディレクトリを作成（ツール実行サーバと同じディレクトリパスにすること）
```
# インストール先のディレクトリ
INSTALLPATH="/home/ansible/local"
mkdir ${INSTALLPATH}
```

ローカルディレクトリにインストールするよう設定
```
cd ${INSTALLPATH}
export PIPENV_VENV_IN_PROJECT=true
```

Pipfileの作成
- ansibleはバージョン2.5からpython3をサポート
- Notebookでボタンやバーの表示をしたいので、以下を導入
  - ipywidgets
- EXCELファイルで作成された設定書や試験項目書の読み書きをするので、以下を導入
  - pandas (dfを使いたい)
  - xlrd
  - openpyxl
- testinfraの出力をcsv化する、また並列実行を可能とするため、以下を導入
  - pytest-csv
  - pytest-xdist

```
cat << EOT > Pipfile
[[source]]
name = "pypi"
url = "https://pypi.org/simple"
verify_ssl = true

[dev-packages]

[packages]
ansible = ">=2.5"
jupyterlab = "*"
ipywidgets = "*"
pandas = "*"
xlrd = "*"
openpyxl = "*"
testinfra = "*"
pytest-csv = "*"
pytest-xdist = "*"

[requires]
python_version = "3.7"
EOT
```

pipenvによるインストール
```
pipenv update --python=${INSTALLPATH}/bin/python3
```

tarでアーカイブ、圧縮
- 圧縮後で200MB程度

```
cd ../
# tar cfzP local.tgz /home/ansible/local --exclude=Pipfile.lock
tar cfzP ${INSTALLPATH##*/}.tgz ${INSTALLPATH} --exclude=Pipfile.lock
```

# （ツール実行サーバ）deployToolsの配置
ツール配置
```
# /home/ansible/local.tgzへ配置
SFTP等で${INSTALLPATH}の1つ上の階層のディレクトリへ配置
```

tarで展開
```
INSTALLPATH=/home/ansible/local
# cd /home/ansible
cd ${INSTALLPATH%/*}
# tar cfz pipenv.tgz pipenv
tar xfzP ${INSTALLPATH##*/}.tgz
```

ユーザプロファイルの編集
```
# ansibleユーザで実行する場合
su - ansible
# INSTALLPATH変数に応じて、修正すること
sed -i 's/^PATH=/PATH=\/home\/ansible\/local\/bin:/home/ansible/local/.venv/bin:/g' .bash_profile
source .bash_profile
```

バージョン確認
```
python --version
git --version
ansible --version
jupyter --version
pytest --version
```
