# メールサーバー構築手順整理

## 環境
```bash
CentOS7.5
```

#### 各種インストール
```bash
# yum install -y cyrus-sasl dovecot postfix cyrus-sasl-plain cyrus-sasl-md5
```

#### postfix master.cf
```bash
#vim /etc/postfix/master.cf
コメントアウトを外す
#submission inet n       -       n       -       -       smtpd
submission inet n       -       n       -       -       smtpd
```

#### postfix main.cf
```bash
# vim /etc/postfix/main.cf

ドメインを設定
mydomain = FQDN(メールの@以降)

myorigin設定
myorigin = $mydomain

inet_interfaces = all

inet_protocols = all


mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain, FQDN(メールの@以降)

home_mailbox = Maildir/


#Postfix SMTPサーバの SASL 認証を有効にします。デフォルトでは、Postfix SMTPサーバは認証を使いません。
smtpd_sasl_auth_enable = yes

# 使用可能な認証メカニズムの設定
# noanonymous : 匿名での接続を拒否。
# noplaintext : PLAIN認証を拒否(Outlook ExpressはPLAIN認証のみ対応）
#smtpd_sasl_security_options = noanonymous, noplaintext
#smtpd_sasl_security_options = noanonymous

#アウトルック，Outlook Expressのログイン認証を利用するための設定
# AUTHコマンドのサポートを認識できないクライアントへの対応(sample-compatibility.cf)
# Outlook Express 4 および Exchange 5等はAUTH コマンドをサポートしていることを認識できないので、
# 使用時は下記のような設定を追加する。
broken_sasl_auth_clients = yes

#SMTP-AUTH 認証で利用するドメイン
#smtpd_sasl_local_domain = $myhostname
smtpd_sasl_local_domain = $mydomain

#Postfix 2.10以降に新設されたオプション。Postfixが受け取ったメールを他のサーバへリレーさせるポリシを規定する
smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination

#Postfix SMTPサーバが RCPT TO コマンドの場面で適用するアクセス制限。
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination


#追加設定（受信メールサイズ制限
message_size_limit = 10485760

#SMTPサーバがクライアントからSMTP接続の要求を受けた際に適用する、オプションのアクセス制限。
#2021-08-07 permit_mynetworks追加
smtpd_client_restrictions = permit_mynetworks,reject_unknown_client_hostname

#Postfix SMTPクライアントでのデフォルトのSMTP TLSセキュリティレベル;
#may日和見TLS。サーバがサポートしていればTLSが使われます
smtp_tls_security_level = may
```


#### dovecot dovecot.conf
```bash
#vim /etc/dovecot/dovecot.conf
protocols = imap pop3 lmtp
```

#### dovecot 10-auth.conf
```bash
vim /etc/dovecot/conf.d/10-auth.conf
disable_plaintext_auth = no
```

#### firewall-cmd
```bash
firewall-cmd --add-port=587/tcp --zone=public --permanent
firewall-cmd --reload
firewall-cmd --add-port=25/tcp --zone=public --permanent
firewall-cmd --reload
firewall-cmd --permanent --add-service=pop3 --zone=public
firewall-cmd --permanent --add-service=imap --zone=public
firewall-cmd --reload
firewall-cmd --list-services
```


#### SFP設定
対象ドメインのSPFに以下のレコードを設定する
```bash
メールドメイン.          120     IN      TXT     "v=spf1 +ip4:IPアドレス -all"
```

#### DMARK設定
_dmarc.対象ドメインのtxtに以下のレコードを設定する
```bash
_dmarc.メールドメイン.   120     IN      TXT     "v=DMARC1\; p=none\; rua=mailto:dmarc-ra@*******\; ruf=mailto:dmarc@*******"
```

#### DKIM設定

dkimのインストール
```bash
target_mail_domain=[メールドメイン]
yum install opendkim
mkdir /etc/opendkim/keys/$target_mail_domain
opendkim-genkey -D /etc/opendkim/keys/$target_mail_domain -d $target_mail_domain -s 20210808 ※←作業日時等
chown -R opendkim:opendkim /etc/opendkim/keys/$target_mail_domain
cat /etc/opendkim/keys/$target_mail_domain/20210808.txt
20210808._domainkey     IN      TXT     ( "v=DKIM1; k=rsa; "
          "p=MIGfMA0(略)B3HZSQIDAQAB" )  ; ----- DKIM key 20210808 for [メールドメイン]
```
20210808._domainkey.対象ドメインのSPFに以下のレコードを設定する
```bash
20210808._domainkey.メールドメイン.   120     IN      TXT     "v=DKIM1; k=rsa; p=MIGfMA0(略)B3HZSQIDAQAB"
```
