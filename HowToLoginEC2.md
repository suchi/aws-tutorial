Bastionサーバから他のEC2インスタンスへのログイン方法を説明

1. 秘密鍵をBastionサーバにアップロードする

```
$ scp -i 秘密鍵 秘密鍵 ec2-user@bastionサーバのPublic IPアドレス:/home/ec2-user/
```

* 例
  ```
  $ scp -i ~/Downloads/keys/hamajo.pem ~/Downloads/keys/hamajo.pem ec2-user@52.91.74.72:/home/ec2-user/
  ```

2. BastionサーバにSSHでログインする

* 例
  ```
  $ ssh -i ~/Downloads/keys/hamajo.pem ec2-user@52.91.74.72
  ```

3. Bastionサーバから他のEC2インスタンスにSSHログインする
