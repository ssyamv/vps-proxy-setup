# VPS 代理搭建记录

## 目标

搭建海外 VPS 代理，用于访问 Claude。

## VPS 信息

| 项目 | 值 |
|------|-----|
| 服务商 | Vultr |
| 地区 | 新加坡 |
| 套餐 | vhp-1c-1gb（$6/月，2TB 流量） |
| 系统 | Ubuntu 22.04 LTS |
| IP | `<your-vps-ip>` |
| Hostname | singapore-proxy-01 |

---

## 一、VPS 服务端搭建

### 1. 购买 VPS

- 注册 Vultr 账号
- 使用支付宝充值 $10
- 创建实例：新加坡节点，Ubuntu 22.04，vhp-1c-1gb

### 2. SSH 连接

在本地 Mac 执行，配置免密快速登录：

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

### 5. 生成密钥材料

```bash
# 生成 UUID
xray uuid

# 生成 Reality 密钥对（Private key 填服务端，Public key 给客户端）
xray x25519

# 生成 shortId
openssl rand -hex 8
```

记录以下值，客户端配置时需要用到：

- UUID
- Public Key（客户端用）
- Private Key（服务端用，不外传）
- Short ID

### 6. 配置 Xray（VLESS + Reality）

```bash
# 备份旧配置（如有）
cp /usr/local/etc/xray/config.json /usr/local/etc/xray/config.json.bak

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

多用户共享时，在 `clients` 数组中添加多个 UUID：

```json
"clients": [
    { "id": "用户A的UUID", "flow": "xtls-rprx-vision" },
    { "id": "用户B的UUID", "flow": "xtls-rprx-vision" }
]
```

### 7. 启动并验证

```bash
systemctl restart xray
systemctl status xray       # 确认 active (running)
ss -tlnp | grep 443         # 确认 443 端口在监听
```

---

## 二、本地客户端配置

> **重要提示：**
> - **浏览器访问 claude.ai**：只需 mixed-port 模式即可
> - **Claude 桌面应用 + Claude Cowork 功能**：需要开启 Clash 的 TUN 模式，否则 Cowork 无法连接

### 方案 A：仅使用自建 VPS（独立配置）

适合没有第三方 VPN 订阅的场景。配置文件 `~/singapore-proxy.yaml`（含敏感信息，已加入 .gitignore）。

使用 [Loyalsoldier/clash-rules](https://github.com/Loyalsoldier/clash-rules) 社区维护的分流规则集：

```yaml
mixed-port: 7897
allow-lan: false
mode: rule
log-level: info

proxies:
  - name: SG-新加坡VPS
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
      - SG-新加坡VPS
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
  - DOMAIN-SUFFIX,anthropic.com,proxy
  - DOMAIN-SUFFIX,claude.ai,proxy
  - DOMAIN-SUFFIX,openai.com,proxy
  - DOMAIN-SUFFIX,github.com,proxy
  - DOMAIN-SUFFIX,githubusercontent.com,proxy
  - DOMAIN-SUFFIX,google.com,proxy
  - DOMAIN-SUFFIX,youtube.com,proxy
  - DOMAIN-SUFFIX,telegram.org,proxy
  - DOMAIN-SUFFIX,jsdelivr.net,proxy
  - RULE-SET,private,DIRECT
  - RULE-SET,reject,REJECT
  - RULE-SET,proxy,proxy
  - RULE-SET,direct,DIRECT
  - RULE-SET,gfw,proxy
  - RULE-SET,cncidr,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,DIRECT
```

验证：

```bash
curl --proxy http://127.0.0.1:7897 -I https://claude.ai
```

### 方案 B：已有第三方 VPN 订阅，仅让 Claude 走自建 VPS

适合日常使用第三方订阅、只希望 Claude 强制走自建 VPS 的场景。日常流量继续走第三方订阅，访问 Claude 时自动切到自建 VPS，无需手动切换。

**前提条件：**

- 已安装 [Clash Verge Rev](https://github.com/clash-verge-rev/clash-verge-rev/releases)
- 已在 Clash Verge 中导入第三方 VPN 订阅并能正常使用

**节点参数：**

| 参数 | 值 |
|------|-----|
| 服务器地址 | `<your-vps-ip>` |
| 端口 | `443` |
| UUID | `<你的UUID>` |
| 协议 | VLESS + REALITY |
| 传输 | TCP |
| Flow | xtls-rprx-vision |
| SNI | `www.microsoft.com` |
| 客户端指纹 | chrome |
| Public Key | `<你的Public Key>` |
| Short ID | `<你的shortId>` |

#### 第一步：找到订阅对应的扩展文件

1. 打开 Clash Verge，点击左侧「**订阅**」
2. 找到日常使用的 VPN 订阅，鼠标悬停后点击「**···**」菜单
3. 选择「**扩展配置**」（可见四个选项：Merge、Rules、Proxies、Groups）

#### 第二步：配置 Proxies 扩展（注入节点）

点击「**Proxies**」右侧的编辑按钮，将内容替换为（把占位符换成实际值）：

```yaml
prepend:
  - name: SG-新加坡VPS
    type: vless
    server: <your-vps-ip>
    port: 443
    uuid: <你的UUID>
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    servername: www.microsoft.com
    client-fingerprint: chrome
    reality-opts:
      public-key: <你的Public Key>
      short-id: <你的shortId>

append: []
delete: []
```

#### 第三步：配置 Rules 扩展（注入分流规则）

点击「**Rules**」右侧的编辑按钮，将内容替换为：

```yaml
prepend:
  - DOMAIN-SUFFIX,anthropic.com,SG-新加坡VPS
  - DOMAIN-SUFFIX,claude.ai,SG-新加坡VPS
  - DOMAIN-SUFFIX,claude.com,SG-新加坡VPS
  - DOMAIN-SUFFIX,claudeusercontent.com,SG-新加坡VPS
  - DOMAIN-SUFFIX,intercom.io,SG-新加坡VPS
  - DOMAIN-SUFFIX,pki.goog,SG-新加坡VPS
  - DOMAIN,api.anthropic.com,SG-新加坡VPS
  - DOMAIN,cdn.anthropic.com,SG-新加坡VPS
  - DOMAIN,e-cdn.anthropic.com,SG-新加坡VPS
  - DOMAIN,statsig.anthropic.com,SG-新加坡VPS

append: []
delete: []
```

#### 第四步：重新激活订阅

回到「订阅」页面，点击 VPN 订阅的「**···**」菜单，选择「**激活**」，等待 Clash Verge 重新加载配置。

**验证：** 在「代理」页面确认节点列表中出现「SG-新加坡VPS」，然后用浏览器访问 [claude.ai](https://claude.ai) 验证连通性。

> **注意：** 规则直接引用节点名 `SG-新加坡VPS`，不经过 proxy-groups。若在 Merge 扩展中使用 `proxy-groups`，会按名称覆盖原有分组的 proxies 列表，导致 `proxy not found` 报错。

---

## 三、进度记录

- [x] 购买 VPS
- [x] SSH 免密登录（`ssh sg`）
- [x] 更新系统
- [x] 安装 Xray
- [x] 配置 VLESS + Reality
- [x] 安装本地客户端（Clash Verge Rev）
- [x] 配置 Clash 分流规则
- [x] 测试代理连接（SIN 节点正常）
- [x] 注册 Claude 账号（接码平台 5sim）
- [x] 购买 Claude Pro（SafePal Fiat24 虚拟卡）
- [x] 接入 Loyalsoldier 社区分流规则集
- [x] 解决 jsdelivr 被墙问题
- [x] 配置第三方订阅 + 自建 VPS 分流并用

---

## 四、排错记录

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Xray 443 端口未监听 | cert.key 权限 600，nobody 用户无法读取 | `chmod 644`（升级 Reality 后无需证书，此问题已消除） |
| rule-providers 下载失败 | `cdn.jsdelivr.net` 国内被墙 | 每个 rule-provider 加 `proxy: proxy`，让 Clash 通过代理下载 |
| 浏览器无法访问 GitHub | rule-providers 未加载时 GitHub 走直连被墙 | 在 rules 靠前位置加保底规则 |
| Git push 失败 | Git 不走系统代理 | `git config --global http.https://github.com.proxy http://127.0.0.1:7897` |
| Clash Verge Merge 的 proxy-groups 陷阱 | Merge 中的 proxy-groups 会覆盖原有分组，导致 `proxy not found` | 用 Proxies 扩展注入节点，Rules 扩展注入规则，规则直接引用节点名称 |

---

## 五、支付经验记录

### 注册 Claude 账号

1. 确保全程使用新加坡 VPS 代理，避免使用中国 IP
2. 使用境外邮箱（Gmail 等）注册
3. 手机验证码：使用接码平台 [5sim](https://5sim.net)（充值约 $1，购买虚拟号码接收验证码）

### Claude Pro 订阅支付

- **套餐**：Claude Pro，SGD 25/月
- **支付工具**：SafePal 内置的 Fiat24 虚拟 Visa 卡（瑞士数字银行，Stripe 可正常支付）
- Country 选 Switzerland，填瑞士地址

**踩坑：**

- 香港汇丰 Visa 卡（人民币账户）被 Stripe 拒绝
- Stripe 地区列表中没有 Hong Kong，填新加坡地址也无效

**人民币 → SafePal 美元充值流程：**

1. 注册币安，C2C 购买 USDC（至少 10 USDC）和少量 ETH（约 0.002 ETH，用于 gas 费）
   - iOS 用户需切换海外 Apple ID 下载币安 App
2. 提币到 SafePal：必须使用 **Arbitrum（ARB）链**
3. 在 SafePal App 中开通 Fiat24 账户（注册推荐码：`399561`，可免费开通 Fiat24 虚拟 Mastercard，完成后获 10 ARB 奖励）
   - iOS 用户同样需要海外 Apple ID
4. 将 USDC 存入 Fiat24（最低 10 USDC，手续费 1%，兑换为 USD）
5. 用 Fiat24 卡在 Stripe 支付

---

## 六、服务解锁情况

| 服务 | 状态 | 原因 |
|------|------|------|
| Claude | 正常 | Anthropic 不封锁数据中心 IP |
| ChatGPT | 正常 | OpenAI 对 VPS IP 较宽松 |
| Google Gemini | 不可用 | Google 封锁数据中心和 Cloudflare IP |
| Netflix / Disney+ | 不可用 | 流媒体严格封锁非住宅 IP |
| YouTube（普通访问） | 正常 | 普通观看不受限 |
| YouTube Premium | 不可用 | Premium 地区验证严格 |

曾尝试 Cloudflare WARP 改善出口 IP 质量，WARP 的 Cloudflare IP 同样被 Gemini、Netflix 封锁，已回退。

**如需解锁流媒体或 Gemini，需住宅 IP 代理或专线机场，自建 VPS 无法解决。**

---

## 七、本地文件说明

| 文件 | 说明 |
|------|------|
| `~/singapore-proxy.yaml` | Clash 独立配置（含敏感信息，已 .gitignore） |
| `~/code/vps-proxy-setup/dounai-merge.yaml` | 第三方订阅扩展配置草稿（含敏感信息，已 .gitignore） |
| `~/.ssh/config` | SSH 别名配置（`ssh sg` 直连 VPS） |
