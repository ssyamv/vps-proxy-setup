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

**注册 SafePal：**
- 官网：https://www.safepal.com
- 推荐码：`399561`（使用推荐码注册可免费开通 Fiat24 虚拟 Mastercard）
