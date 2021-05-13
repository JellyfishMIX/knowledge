# 检查现有 ssh 密钥

在生成 SSH 密钥之前，您可以检查是否有任何现有的 SSH 密钥。

**注：** DSA 密钥 (SSH-DSS) 不再受支持。 现有密钥将继续运行，但您不能将新的 DSA 密钥添加到您的 GitHub 帐户。

1. 打开 Terminal（终端）。

2. 输入 `ls -al ~/.ssh` 以查看是否存在现有 SSH 密钥：

   ```shell
   $ ls -al ~/.ssh
   # Lists the files in your .ssh directory, if they exist
   ```

3. 检查目录列表以查看是否已经有 SSH 公钥。 默认情况下，公钥的文件名是以下之一：

   - *id_rsa.pub*
   - *id_ecdsa.pub*
   - *id_ed25519.pub*