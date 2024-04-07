# [mkcert](github.com/FiloSottile/mkcert)

### 基本使用：

- 安装本地 CA 至系统信任储存:
```bash  
mkcert -install
```
- 为单个域名生成证书和密钥文件：
```bash  
mkcert example.org
```
- 为多个域名和本地主机生成证书和密钥文件：
```bash  
mkcert example.com myapp.dev localhost 127.0.0.1 ::1
```
- 为通配符域名生成证书和密钥文件：
```bash  
mkcert "*.example.it"
```
- 卸载本地 CA，但不删除它：
```bash  
mkcert -uninstall
```

### 高级选项：

- `-cert-file FILE, -key-file FILE, -p12-file FILE`：
  自定义输出的路径。通过这些选项您可以指定生成的证书文件和密钥文件的保存位置。

- `-client`：
  生成用于客户端身份验证的证书。

- `-ecdsa`：
  使用ECDSA密钥生成证书，而不是默认的RSA密钥。

- `-pkcs12`：
  生成包含证书和密钥的 ".p12" PKCS#12 文件，这种格式又被称作 ".pfx" 文件，在某些老旧应用中可能需要使用这种格式。

- `-csr CSR`：
  根据提供的CSR (Certificate Signing Request，证书签名请求) 生成证书。这个选项与其他所有标志和参数冲突，除了 -install 和 -cert-file。

- `-CAROOT`：
  显示 CA 证书和密钥储存位置。

- `$CAROOT（环境变量）`：
  设置 CA 证书和密钥储存位置，这样您可以同时维护多个本地 CAs。

- `$TRUST_STORES（环境变量）`：
  一个以逗号分隔的列表，代表需要安装本地根 CA 的信任储存。选项包括："system"（系统），"java" 和 "nss" (包括 Firefox)。默认情况下会自动检测。

[CSR-常见问题-文档中心-腾讯云](https://cloud.tencent.com/document/product/400/5367)

> **请注意！** 你必须把这些选项放在域名列表之前。

```zsh
mkcert -key-file key.pem -cert-file cert.pem example.com *.example.com
```

### S/MIME （邮件安全证书）

用下面这种方式 `mkcert` 会生成一个 S/MIME 证书：

```bash
mkcert bor@example.com
```

### 在其它系统上安装 CA

安装 trust store 不需要 CA key（只要 CA），所以你可以导出 CA，并且使用 `mkcert` 来安装到其它机器上。

* 找到 `rootCA.pem` 文件，可以用 `mkcert -CAROOT` 找到对应目录。

* 把它 copy 到别的机器上。

* 设置 `$CAROOT` 为 `rootCA.pem` 所在目录。

* 运行 `mkcert -install`(arch linux 可以 `sudo trust anchor --store rootCA.pem`，其它发行版可以用自带的命令手动添加来信任 CA)
