# 设置ACME自动网站证书更新


如果你使用的是 Let's Encrypt 颁发的证书，并且你的证书存储在 /home/ubuntu/.acme.sh/ 目录下，这通常意味着你使用的是 acme.sh 这个工具来管理你的 SSL 证书。acme.sh 是一个用于自动安装和续订 Let's Encrypt SSL 证书的 Shell 脚本。

以下是更新证书的步骤：

更新证书：
使用 acme.sh 更新证书，你可以运行以下命令：

bash
~/.acme.sh/acme.sh --upgrade --install-cert -d plmxs.top -d www.plmxs.top --ecc
这个命令会更新 plmxs.top 和 www.plmxs.top 的证书（包括通配符证书，如果有的话），并且使用 ECC 曲线。如果你的证书不是 ECC 类型，可以去掉 --ecc 参数。

检查证书状态：
你可以使用以下命令来检查证书的状态：

bash
~/.acme.sh/acme.sh --list
这将列出所有通过 acme.sh 管理的证书及其状态。

重新加载 Nginx：
更新证书后，你需要重新加载 Nginx 以使新的证书生效。可以使用以下命令：

bash
sudo nginx -s reload
或者如果你使用的是 systemd 管理 Nginx 服务：

bash
sudo systemctl reload nginx
设置自动续订：
acme.sh 默认会每天检查证书是否需要续订，并且自动续订即将到期的证书。你可以确认这个设置是否已经启用：

bash
~/.acme.sh/acme.sh --cron
如果返回结果表明定时任务未设置，你可以使用以下命令来启用它：

bash
~/.acme.sh/acme.sh --upgrade --auto-upgrade
备份证书：
在更新证书之前，建议备份现有的证书文件，以防更新过程中出现问题。

---

## 免费的证书网站ZeroSSL

ZeroSSL 提供了免费的 SSL 证书，这些证书可以通过自动化的方式生成并管理。以下是将 ZeroSSL 生成的 SSL 证书导入到 Nginx 的一般步骤：

1. **生成 SSL 证书**：
   - 访问 ZeroSSL 网站并注册账号。
   - 使用 ZeroSSL 的界面或 REST API 生成 SSL 证书。你可以选择 90 天的证书或 1 年的证书，甚至通配符证书。
   - 完成必要的验证步骤，如电子邮件验证、服务器上传或 CNAME 验证。

2. **下载证书文件**：
   - 一旦证书生成，ZeroSSL 会提供证书文件供下载，通常包括：
     - 证书文件（`.crt` 或 `.pem` 格式）
     - 私钥文件（`.key` 格式）
     - 可选的中间证书文件（`.ca-bundle` 或 `.pem` 格式）

3. **配置 Nginx**：
   - 打开你的 Nginx 配置文件，通常位于 `/etc/nginx/nginx.conf` 或 `/etc/nginx/sites-available/` 目录下的特定站点配置文件。
   - 在 `server` 块中，设置 `ssl_certificate` 和 `ssl_certificate_key` 指令指向你下载的证书和私钥文件。例如：
     ```nginx
     server {
         listen 443 ssl;
         server_name example.com;

         ssl_certificate /path/to/your_certificate.crt;
         ssl_certificate_key /path/to/your_private_key.key;

         # 如果需要，也可以包含中间证书
         ssl_trusted_certificate /path/to/your_intermediate_certificate.pem;

         location / {
             # 你的配置...
         }
     }
     ```
   - 确保路径指向正确的证书文件位置。

4. **重启 Nginx**：
   - 保存配置文件后，重启 Nginx 以应用更改。你可以使用以下命令：
     ```
     sudo service nginx restart
     ```
   - 或者使用：
     ```
     sudo systemctl restart nginx
     ```

5. **验证配置**：
   - 通过访问 `https://yourdomain.com` 来验证 SSL 证书是否已正确安装。
   - 你可以使用在线 SSL 检查工具来验证配置是否正确。

6. **自动化续签**：
   - ZeroSSL 支持自动化续签，你可以利用他们的 REST API 或 ACME 集成来自动续签证书，避免证书过期。

请注意，具体的步骤可能会根据你的服务器环境和 Nginx 配置有所不同。如果你需要更详细的帮助，可以查看 ZeroSSL 的开发者部分或 Nginx 的官方文档。

这个公司还提供了个机器人，用来获得证书。不过看起来不好用。没有自动更新的设置呢。https://github.com/zerossl/zerossl-bot