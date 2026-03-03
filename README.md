# VPS 代理搭建记录

## 目标
搭建海外 VPS 代理，用于访问 Claude Cowork。

## VPS 信息
- **服务商**：Vultr
- **地区**：新加坡
- **套餐**：vhp-1c-1gb（$6/月，2TB 流量）
- **系统**：Ubuntu 22.04 LTS
- **IP**：`<your-vps-ip>`
- **Hostname**：singapore-proxy-01

## 搭建步骤

### 1. 购买 VPS
- 注册 Vultr 账号
- 使用支付宝充值 $10
- 创建实例：新加坡节点，Ubuntu 22.04，vhp-1c-1gb

### 2. SSH 连接

配置 SSH 别名实现免密快速登录（在本地 Mac 执行）：

```bash
# 生成 SSH 密钥（已有则跳过）
ssh-keygen -t ed25519

# 上传公钥到 VPS（只需输一次密码）
ssh-copy-id root@<your-vps-ip>
```

在 `~/.ssh/config` 中添加别名：
```
Host sg
  HostName <your-vps-ip>
  User root
  IdentityFile ~/.ssh/id_ed25519
```

之后直接 `ssh sg` 即可免密连接 VPS。

### 3. 更新系统
```bash
apt update && apt upgrade -y
```

### 4. 安装 Xray
```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

### 5. 生成 UUID
```bash
xray uuid
```

### 6. 配置 Xray（待完成）
配置文件路径：`/usr/local/etc/xray/config.json`

### 7. 配置本地客户端（待完成）
推荐使用 Clash Verge for macOS

## 进度
- [x] 购买 VPS
- [x] SSH 连接
- [x] 更新系统
- [x] 安装 Xray
- [x] 生成 UUID
- [x] 配置 Xray（VLESS + TLS）— 已升级为 VLESS + Reality
- [x] 生成自签名证书 — 升级 Reality 后不再需要
- [x] 开放防火墙 443 端口
- [x] 安装本地客户端（Clash Verge Rev）
- [x] 配置 Clash 规则（使用 Loyalsoldier 社区规则集，自动分流）
- [x] 测试代理连接（curl 返回 HTTP/2 404，cf-ray 显示 SIN ✅）
- [x] 成功访问 Claude 套餐购买页面（显示 SGD，代理正常）
- [x] 购买 Claude Pro 套餐（使用 SafePal Fiat24 虚拟卡美元支付成功）
- [ ] 登录 Claude 并使用 Cowork
- [x] 升级协议为 VLESS + Reality
- [x] 配置 SSH 免密登录（`ssh sg` 直连 VPS）
- [x] 接入 Loyalsoldier 社区分流规则集
- [x] 解决 jsdelivr 被墙问题（rule-providers 通过代理下载）
- [x] 尝试 Cloudflare WARP 解锁流媒体（失败，已回退）

## 本地配置文件
- Clash 配置：`~/singapore-proxy.yaml`
- SSH 配置：`~/.ssh/config`（别名 `sg`）
- 代理端口：`7897`

## 排错记录
- Xray 启动成功但 443 端口未监听：原因是 cert.key 权限为 600，nobody 用户无法读取，执行 `chmod 644` 解决
- Clash 配置 rule-providers 报错：改用内置 GEOSITE 规则避免外部规则集下载问题
- Clash 节点 Timeout：实际是 Xray 正常运行，通过 `xray run` 手动测试确认端口已绑定
- jsdelivr CDN 被墙导致 rule-providers 下载失败：在每个 rule-provider 中添加 `proxy: proxy` 字段，让规则文件通过代理下载（详见下方说明）
- 流媒体/Gemini 检测不通过：Vultr 数据中心 IP 被 Netflix、Disney+、Google Gemini 等服务封锁，属于 VPS 的 IP 质量限制，非配置问题
- Cloudflare WARP 解锁尝试失败：在 VPS 上部署了 WARP socks5 代理并让 Google/流媒体流量走 WARP 出口，但 Cloudflare IP 同样被这些服务识别和封锁，最终回退
- 浏览器无法访问 GitHub：rule-providers 未加载成功时，GitHub 域名没有匹配到代理规则，走了直连被墙。解决方法：在 rules 中添加常用被墙站点的保底规则，放在 rule-providers 之前
- Git push 到 GitHub 失败：Git 命令行默认不走系统代理。解决方法：`git config --global http.https://github.com.proxy http://127.0.0.1:7897`，仅对 GitHub 生效

## 升级：从 VLESS + TLS 升级到 VLESS + Reality

### 为什么要升级？
- **不再需要自签证书**：Reality 伪装为访问真实网站，无需管理证书
- **抗检测能力最强**：流量特征与正常访问大网站完全一致，DPI 几乎无法识别
- **性能更好**：XTLS Vision 直接转发 TLS，减少一层加密开销
- **配置更简单**：不用处理证书权限、过期等问题

### 升级步骤（在 VPS 上执行）

#### 1. SSH 连接到 VPS
```bash
ssh root@<your-vps-ip>
```

#### 2. 确保 Xray 是最新版本
```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

#### 3. 生成 Reality 密钥对
```bash
xray x25519
```
输出示例：
```
Private key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Public key:  YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY
```
**记下这两个值！** Private key 填服务端配置，Public key 填客户端配置。

#### 4. 生成新的 UUID（或沿用旧的）
```bash
xray uuid
```

#### 5. 生成 shortId
```bash
openssl rand -hex 8
```

#### 6. 备份旧配置
```bash
cp /usr/local/etc/xray/config.json /usr/local/etc/xray/config.json.bak
```

#### 7. 写入新配置
```bash
cat > /usr/local/etc/xray/config.json << 'EOF'
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "listen": "0.0.0.0",
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "<你的UUID>",
                        "flow": "xtls-rprx-vision"
                    }
                ],
                "decryption": "none"
            },
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                    "dest": "www.microsoft.com:443",
                    "serverNames": [
                        "www.microsoft.com",
                        "microsoft.com"
                    ],
                    "privateKey": "<你的Private Key>",
                    "shortIds": [
                        "<你的shortId>"
                    ]
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
EOF
```

**替换以下占位符：**
- `<你的UUID>` → 步骤 4 生成的 UUID
- `<你的Private Key>` → 步骤 3 生成的 Private key
- `<你的shortId>` → 步骤 5 生成的 shortId

#### 8. 重启 Xray
```bash
systemctl restart xray
```

#### 9. 检查 Xray 状态
```bash
systemctl status xray
```
确认状态为 `active (running)`。

#### 10. 确认 443 端口正在监听
```bash
ss -tlnp | grep 443
```

### 更新本地 Clash 客户端配置

升级完服务端后，需要更新 `~/singapore-proxy.yaml`。

使用 [Loyalsoldier/clash-rules](https://github.com/Loyalsoldier/clash-rules) 社区维护的分流规则集，每日自动更新：

```yaml
mixed-port: 7897
allow-lan: false
mode: rule
log-level: info

proxies:
  - name: singapore-vless-reality
    type: vless
    server: <your-vps-ip>
    port: 443
    uuid: <你的UUID>
    network: tcp
    udp: true
    tls: true
    flow: xtls-rprx-vision
    servername: www.microsoft.com
    reality-opts:
      public-key: <你的Public Key>
      short-id: <你的shortId>
    client-fingerprint: chrome

proxy-groups:
  - name: proxy
    type: select
    proxies:
      - singapore-vless-reality
      - DIRECT

rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    path: ./ruleset/reject.yaml
    interval: 86400
    proxy: proxy
  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400
    proxy: proxy
  direct:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct.txt"
    path: ./ruleset/direct.yaml
    interval: 86400
    proxy: proxy
  gfw:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/gfw.txt"
    path: ./ruleset/gfw.yaml
    interval: 86400
    proxy: proxy
  cncidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt"
    path: ./ruleset/cncidr.yaml
    interval: 86400
    proxy: proxy
  private:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/private.txt"
    path: ./ruleset/private.yaml
    interval: 86400
    proxy: proxy

rules:
  # AI 服务（优先级最高，确保走代理）
  - DOMAIN-SUFFIX,anthropic.com,proxy
  - DOMAIN-SUFFIX,claude.ai,proxy
  - DOMAIN-SUFFIX,openai.com,proxy
  - DOMAIN-SUFFIX,chatgpt.com,proxy
  - DOMAIN-SUFFIX,oaistatic.com,proxy
  - DOMAIN-SUFFIX,oaiusercontent.com,proxy

  # 常用被墙站点（保���规则，防止 rule-providers 未加载时无法访问）
  - DOMAIN-SUFFIX,github.com,proxy
  - DOMAIN-SUFFIX,githubusercontent.com,proxy
  - DOMAIN-SUFFIX,github.io,proxy
  - DOMAIN-SUFFIX,google.com,proxy
  - DOMAIN-SUFFIX,googleapis.com,proxy
  - DOMAIN-SUFFIX,googlevideo.com,proxy
  - DOMAIN-SUFFIX,youtube.com,proxy
  - DOMAIN-SUFFIX,ytimg.com,proxy
  - DOMAIN-SUFFIX,twitter.com,proxy
  - DOMAIN-SUFFIX,x.com,proxy
  - DOMAIN-SUFFIX,twimg.com,proxy
  - DOMAIN-SUFFIX,telegram.org,proxy
  - DOMAIN-SUFFIX,t.me,proxy
  - DOMAIN-SUFFIX,wikipedia.org,proxy
  - DOMAIN-SUFFIX,jsdelivr.net,proxy

  # Loyalsoldier 规则集
  - RULE-SET,private,DIRECT
  - RULE-SET,reject,REJECT
  - RULE-SET,proxy,proxy
  - RULE-SET,direct,DIRECT
  - RULE-SET,gfw,proxy
  - RULE-SET,cncidr,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,DIRECT
```

**规则集说明：**

| 规则集 | 作用 |
|-------|------|
| private | 局域网/私有地址 → 直连 |
| reject | 广告域名 → 拦截 |
| proxy | 需要代理的域名（Google、YouTube、Twitter 等） → 代理 |
| direct | 国内常用域名 → 直连 |
| gfw | GFW 封锁的域名 → 代理 |
| cncidr | 中国 IP 段 → 直连 |

**替换以下占位符：**
- `<your-vps-ip>` → 你的 VPS IP 地址
- `<你的UUID>` → 与服务端相同的 UUID
- `<你的Public Key>` → 步骤 3 生成的 **Public key**（注意：服务端用 Private key，客户端用 Public key）
- `<你的shortId>` → 与服务端相同的 shortId

### 升级后验证

在 Clash Verge Rev 中切换到新配置后，测试：
```bash
curl --proxy http://127.0.0.1:7897 -I https://claude.ai
```
看到返回 HTTP 响应头即为成功。

### 升级后可以清理的旧文件（在 VPS 上）
自签证书不再需要了：
```bash
rm -f /usr/local/etc/xray/cert.crt /usr/local/etc/xray/cert.key
```

## 下一步
- 登录 Claude 桌面客户端，开始使用 Cowork 功能

## 支付经验记录

### 注册 Claude 账号
1. **确保代理已开启**：全程使用新加坡 VPS 代理，避免使用中国 IP
2. **准备邮箱**：使用境外邮箱（Gmail 等）注册
3. **手机验证码**：Claude 注册需要境外手机号，使用接码平台 [5sim](https://5sim.net) 解决
   - 在 5sim 充值少量余额（约 $1）
   - 购买一个境外虚拟号码接收短信验证码
4. **完成注册**：填写邮箱、设置密码、输入验证码，全程顺利



### Claude Pro 订阅支付
- **套餐**：Claude Pro，SGD 25/月
- **支付工具**：SafePal 内置的 Fiat24 虚拟 Visa 卡
- **币种**：美元（USD）
- **结果**：支付成功

**踩坑记录：**
- 香港汇丰 Visa 卡（人民币账户）被 Stripe 拒绝，原因可能是跨境风控或账单地址不匹配
- Stripe 地区列表中没有 Hong Kong 选项，填新加坡地址也无效
- Fiat24 是瑞士数字银行发行的真实 Visa 卡，Stripe 可以正常支付
- Country 选 Switzerland，填瑞士地址即可

**Fiat24 充值方式：**
- 通过 SafePal App 将加密货币兑换为 USD/EUR 充值到 Fiat24 账户
- 确保余额足够（Claude Pro 约 $20 USD）

**从人民币到 SafePal 美元的完整充值流程：**
1. **注册币安**：如果还没有币安账号，可以通过以下邀请链接注册（需要科学上网）
   - 邀请链接：https://www.bsmkweb.cc/referral/earn-together/refer2earn-usdc/claim?hl=zh-CN&ref=GRO_28502_XEP4R&utm_source=default
   - **iOS 用户注意**：币安 App 在中国区 App Store 不可用，需要切换到海外 Apple ID（如香港、新加坡区）下载
2. **币安 C2C 购买 USDC 和 ETH**：在币安 App 使用人民币通过 C2C 购买 USDC（至少 10 USDC）和少量 ETH（约 0.002 ETH，用于激活和 gas 费）
3. **转账到 SafePal（必须用 Arbitrum 链）**：将币安中的 USDC 和 ETH 提币到 SafePal 钱包地址，**只能使用 Arbitrum（ARB）链**，不支持其他链
   - 激活账号需要钱包中有至少 0.002 ETH + 10 USDC（仅用于验证，不扣费）
   - **iOS 用户注意**：SafePal App 同样需要海外 Apple ID 才能下载
4. **SafePal 注册 Fiat24 银行账户**：在 SafePal App 中开通 Fiat24 账户，完成后可获得 10 ARB 奖励
5. **充值到 Fiat24**：将 USDC 存入 Fiat24 账户，最低 10 USDC，手续费 1%，之后兑换为 USD
6. **用 Fiat24 卡支付**：Fiat24 账户有余额后即可用虚拟卡在 Stripe 支付

**注册 SafePal：**
- 官网：https://www.safepal.com
- 推荐码：`399561`（使用推荐码注册可免费开通 Fiat24 虚拟 Mastercard）

## jsdelivr 被墙解决方案

国内无法直接访问 `cdn.jsdelivr.net`，导致 Clash 的 rule-providers 下载失败。

**解决方法**：在每个 rule-provider 配置中添加 `proxy: proxy` 字段，让 Clash 通过代理节点下载规则文件：

```yaml
rule-providers:
  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400
    proxy: proxy          # 关键：通过代理下载规则文件
```

这是 Mihomo（Clash Verge Rev 的核心引擎）支持的官方特性。

## Cloudflare WARP 解锁尝试（失败）

### 目的
尝试通过 WARP 改善 VPS 的出口 IP 质量，解锁 Google Gemini、Netflix 等服务。

### 操作步骤
1. 在 VPS 上安装 Cloudflare WARP 客户端
2. 设置为 socks5 代理模式（`127.0.0.1:40000`）
3. 修改 Xray 配置，添加 WARP 出口和路由规则，让 Google/流媒体流量走 WARP

### 结果
**失败**。WARP 的出口 IP 虽然是 Cloudflare IP（非数据中心标记），但 Google Gemini、Netflix、YouTube Premium 等服务同样封锁了 Cloudflare 的 IP 段。

### 结论与回退
- 已将 Xray 配置恢复为原始版本（纯 freedom 出口）
- WARP 服务已断开并禁用（未卸载，保留在 VPS 上备用）
- 如需重新启用：`systemctl enable --now warp-svc && warp-cli connect`

### 关于流媒体/AI 服务解锁

| 服务 | 状态 | 原因 |
|------|------|------|
| Claude | ✅ 正常 | Anthropic 不封锁数据中心 IP |
| ChatGPT | ✅ 正常 | OpenAI 对 VPS IP 较宽松 |
| Google Gemini | ❌ 不可用 | Google 封锁数据中心和 Cloudflare IP |
| Netflix / Disney+ | ❌ 不可用 | 流媒体严格封锁非住宅 IP |
| YouTube（普通访问） | ✅ 正常 | 普通观看不受限 |
| YouTube Premium | ❌ 不可用 | Premium 地区验证严格 |

**如需解锁流媒体/Gemini，需要住宅 IP 代理或专线机场服务，自建 VPS 无法解决。**

## 多用户共享

如果需要给朋友共享代理，只需在 Xray 配置中添加多个用户（UUID）：

```json
"clients": [
    { "id": "用户A的UUID", "flow": "xtls-rprx-vision" },
    { "id": "用户B的UUID", "flow": "xtls-rprx-vision" }
]
```

每个用户使用不同的 UUID，共享同一台服务器。$6/月 2TB 流量，几个人日常使用足够。
