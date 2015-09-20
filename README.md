# chef_server_client

3つのvmを用意
chef-workstation：192.168.33.9
chef-client：192.168.33.91
chef-server：192.168.33.38
＊仮想環境同士の通信は可能
（参考：http://dev.classmethod.jp/server-side/chef-server-install/)

【chef-server】
◆Chef Serverをインストール

https://downloads.chef.io/chef-server/redhat/
登録をしてダウンロード

それをアップロードして以下のコマンドを実行
sudo rpm -Uvh chef-server-core-12.2.0-1.el6.x86_64.rpm

１つのChef Serverで10,000+以上のノードを管理が可能

以降のインストール手順
https://docs.chef.io/server/install_server.html

参考：https://gist.github.com/kazu69/0efcc34d02f5443bf0a8

◆サービスの実行
chef-server-ctl reconfigure
(4,5分かかる)

◆セットアップ
sudo chef-server-ctl reconfigure
sudo chef-server-ctl install opscode-manage
sudo opscode-manage-ctl reconfigure
sudo chef-server-ctl reconfigure

◆管理者ユーザの作成
chef-server-ctl user-create user_name first_name last_name email password --filename FILE_NAME
例）
chef-server-ctl user-create admin yusuke ueno yusuke1581@gmail.com J0ker817 --filename chef-user.pem
*鍵自体は大事な場所に保管する必要がある
鍵はサーバへの接続用

◆管理機構の作成
chef-server-ctl org-create short_name "full_organization_name" --association_user user_name --filename ORGANIZATION-validator.pem
例）
sudo chef-server-ctl org-create test "Chef Test" --association admin --filename chef-validator.pem
===確認===
chef-server-ctl org-show
chef-server-ctl org-user-add chef admin

【workstation】
◆rubyのインストール
sudo yum -y groupinstall “Development Tools"
sudo yum -y install git openssl-devel zlib-devel readline-devel
git clone http://github.com/sstephenson/rbenv.git ~/.rbenv
echo ‘export PATH=“$HOME/.rbenv/bin:$PATH”’ >> ~/.bash_profile
echo ‘eval “$(rbenv init -)”’ >> ~/.bash_profile
exec $SHELL -l
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
rbenv install -v 2.1.0
rbenv global 2.1.0

◆Workstationの.chef/knife.rbの設定
(Workstation：Chefサーバの操作を行う端末)
/opt/opscode/bin/knife configure
WARNING: No knife configuration file found
Where should I put the config file? [/home/vagrant/.chef/knife.rb]
Please enter the chef server URL: [https://localhost:443]
Please enter an existing username or clientname for the API: [vagrant]
Please enter the validation clientname: [chef-validator]
Please enter the location of the validation key: [/etc/chef-server/chef-validator.pem]
Please enter the path to a chef repository (or leave blank):
*****

You must place your client key in:
/home/vagrant/.chef/vagrant.pem
Before running commands with Knife!

*****

You must place your validation key in:
/etc/chef-server/chef-validator.pem
Before generating instance data with Knife!

*****
Configuration file written to /home/vagrant/.chef/knife.rb

◆Workstationに鍵を転送
scp -o stricthostkeychecking=no vagrant@192.168.33.12:/home/vagrant/admin.pem .chef/admin.pem

scp -o stricthostkeychecking=no vagrant@192.168.33.9:/home/vagrant/chef-validator.pem .chef/chef-validator.pem

workstation上で以下を実行
scp -o stricthostkeychecking=no vagrant@192.168.33.38:/vagrant/chef-user.pem chef- user.pem
scp -o stricthostkeychecking=no vagrant@192.168.33.38:/vagrant/chef-validator.pem chef-validator.pem

◆chefのgemをインストール
gem install chef —no-ri —no-rdoc

◆証明書の取得
/opt/opscode/bin/knife ssl fetch -s https://chef-server/organizations/chef/

WARNING: Certificates from chef-server will be fetched and placed in your trusted_cert
directory (/home/vagrant/.chef/trusted_certs).

Knife has no means to verify these are the correct certificates. You should
verify the authenticity of these certificates after downloading.

Adding certificate for localhost in /home/vagrant/.chef/trusted_certs/localhost.crt

sslで通信できるチェック
/opt/opscode/bin/knife ssh check

WARN: Failed to read the private key /home/vagrant/.chef/vagrant.pem: #<Errno::ENOENT: No such file or directory @ rb_sysopen - /home/vagrant/.chef/vagrant.pem>
ERROR: Your private key could not be loaded from /home/vagrant/.chef/vagrant.pem
Check your configuration file and ensure that your private key is readable

http://www.creationline.com/lab/6632

◆WorkstationからChef Serverにnodeを登録
$ knife bootstrap chef-client -x vagrant -P vagrant —sudo

Errorメッセージ
Doing old-style registration with the validation key at /home/vagrant/chef/chef-repo/.chef/chef-validator.pem...
Delete your validation key in order to use your user credentials instead

→参考記事：http://www.creationline.com/lab/9295
.chef/ディレクトリからvalidator秘密鍵ファイルを削除すれば良いとのこと
(homeディレクトリにvalidator秘密鍵ファイルを移動する)
https://github.com/chef/chef/issues/2536

chef-validator.pemを.chefに置くと、
knife bootstrap 192.168.33.91 -N node1 -x vagrant -P vagrant --sudo
が通るが、
Chef encountered an error attempting to create the client "node1"

Permissionsを作成したいが...
https://docs.chef.io/server_orgs.html#org-create

https://docs.chef.io/auth.html
chef-validatro
clientは/etc/chef/client.pem(秘密鍵)を利用してリクエストを発行する
初回にclientが実行される際には、秘密鍵が存在しないため、clientは/etc/chef/validation.pemに置かれたchef-validatorの秘密鍵を利用しようと試みる
validation.pemはknife configureで作成される

（clientとnodeの違い：http://blog.madoro.org/mn/81)

http://www.creationline.com/lab/6632

knife bootstrap 192.168.33.91 -x vagrant --sudo



◆Groupを作成してみる

◆設定状況
１）workstation


２）chef-server
管理機構
test "Chef Test"
admin
chef-validator.pem

ユーザ
admin (yusuke ueno)
yusuke1581@gmail.com
J0ker817
chef-user.pem

chef-validator.pem
chef-validatorは新しいクライアントを登録するための特別なクライアント
