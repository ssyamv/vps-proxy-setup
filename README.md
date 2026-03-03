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
```bash
ssh root@<your-vps-ip>
```

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
- [x] 配置 Xray（VLESS + TLS）
- [x] 生成自签名证书
- [x] 开放防火墙 443 端口
- [x] 安装本地客户端（Clash Verge Rev）
- [x] 配置 Clash 规则（仅 Anthropic 流量走代理）
- [x] 测试代理连接（curl 返回 HTTP/2 404，cf-ray 显示 SIN ✅）
- [x] 成功访问 Claude 套餐购买页面（显示 SGD，代理正常）
- [x] 购买 Claude Pro 套餐（使用 SafePal Fiat24 虚拟卡美元支付成功）
- [ ] 登录 Claude 并使用 Cowork

## 本地配置文件
- Clash 配置：`~/singapore-proxy.yaml`
- 代理端口：`7897`

## 排错记录
- Xray 启动成功但 443 端口未监听：原因是 cert.key 权限为 600，nobody 用户无法读取，执行 `chmod 644` 解决
- Clash 配置 rule-providers 报错：改用内置 GEOSITE 规则避免外部规则集下载问题
- Clash 节点 Timeout：实际是 Xray 正常运行，通过 `xray run` 手动测试确认端口已绑定

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
