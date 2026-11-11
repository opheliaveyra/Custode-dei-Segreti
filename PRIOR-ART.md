# Prior art — 别人已经走过的路（2026-11-03 调研）

调研范围：GitHub 仓库搜索（agent credential vault / llm agent secrets / mcp secrets vault 等）
+ 官方产品页与文档。结论先行：**做“真保管箱”的仓库很多，做“给普通用户、粘给新模型就能用的
行为协议”的，暂时没搜到——那正是本仓库的位置。** 已有实现绝大多数面向开发者/企业，
管的是 API Key，不是个人网站密码 + 验证码，也没有“怎么跟用户说话”这一层。

## 1. Infisical/agent-vault（约 2,468 ★，本领域最大开源实现）

https://github.com/Infisical/agent-vault

- 形态：单个 Go 二进制，MITM 代理 + 保管箱。Agent 环境里只有假占位符
  （如 `__anthropic_api_key__`），请求经过代理时才被换成真凭据再转发。
- 可借鉴：
  - **占位符替换**：除浏览器填表外，API 调用走“出口替换”是第二种注入点。
  - **Access Bundle + Service 规则**：按 host / 方法 / 路径限定一条凭据能用在哪。
  - **限时 Session**：会话有期限、可撤销，撤销约 1 分钟内生效；不是长期常驻授权。
  - **严格模式**：不匹配任何规则的主机直接拒绝（deny），而不是默认放行。
  - **部署假设写在明处**：保管箱必须跑在与 Agent 不同的机器上，同机则保证不成立。

## 2. BnayaZil/agent-vault-cli（小仓库，形态与个人浏览器登录最接近）

https://github.com/BnayaZil/agent-vault-cli

- 形态：CLI + OS 钥匙串，经 CDP 直连浏览器填表；Agent 只调 `vault login`，
  只拿到 success / failed。
- 可借鉴：
  - **Origin 从页面读，不听 Agent 说**：防 Agent 被诱导把密码填进钓鱼站。
  - **“能防 / 不能防”对照表**：把威胁模型写成一页表格，诚实标注防不住的
    （如恶意 Agent 主动翻 DOM/网络）。
  - **反模式点名**：会把凭据“返回给 Agent”的 MCP 工具并没有解决
    “别让密钥进入 LLM 上下文”这个问题。

## 3. 1Claw — Browser Bridge（博客 + 文档，非纯仓库形态）

https://1claw.co/blog/log-in-without-giving-your-agent-the-password

- 可借鉴：
  - **按名字请求填充**：Agent 只发 `{binding_id}`，只收到 `{status: "filled"}`；
    请求里连 URL 都不许带，导航由桥按预先绑定自己做。
  - **Host 精确匹配，拒绝通配符**：创建绑定时就拒绝 `*.example.com`，
    不存一个“看着放行、实际对不上”的假宽松配置。
  - **三把凭据分开**：哪台机器 / 哪个人 / 哪个 Agent，三者齐了才能填；
    取结果的通道明确拒绝 Agent 的令牌。

## 4. 1Password for Agents（闭源产品，2026-10-22 发布）

https://1password.com/blog/1password-for-ai-agents

- 形态：Agent 用自己的数字密钥证明身份 → 用户在自己设备上逐次批准 →
  1Password 直接填入，密码与验证码都不经模型。Muse 本身就在其首批支持名单里。
- 可借鉴：
  - **逐任务、设备端批准**（含生物识别），授权不延续到下一次会话。
  - **Agentic Mode 锁定**：Agent 一接管浏览器，保险库自动收紧到“本次批准的那几条”，
    其余条目不可达。
  - 教训：协议层应先写明“优先接已有集成”，而不是人人再造一个保险库。

## 5. OpenClaw — SecretRef（文档）

https://docs.openclaw.ai/zh-CN/gateway/secrets

- 可借鉴：
  - **哨兵值（sentinel）**：运行时用 `oc-sent-v1-...` 这类不透明值占位，
    请求离开进程前的最后一跳才替换成真值。
  - **明文残留审计**：提供 `secrets audit --check` 扫描配置/生成文件里还有没有明文。
  - **诚实的边界**：文档明说 SecretRef 不是进程隔离——只要明文还在 Agent 能读的
    磁盘路径上，就能被绕过。协议要敢写自己防不住什么。

## 6. 其他值得记一笔的

- **Anthropic Managed Agents 的 vaults**（二手调研口径）：`secret_value` 只写不可读回、
  双层 host 白名单、只许注入 header/body 不许进 URL 查询串（URL 会进日志）。
- **CyberArk Secretless Broker / HashiCorp Vault Agent Injector**：更早的
  “客户端不持有凭据、由 broker 完成认证握手”的老祖宗模式，SPEC 里应致敬谱系。
- **Agent-to-agent 委派收窄**（如 id.wispera 的 delegation chains）：
  凭据权限转交给子 Agent 时只能变窄、不能变宽——多 Agent 场景（如 JiuJiu-Crew）用得上。
- 几个同名/近名小仓库（Agent-Vault/Agent-Vault、hdez41/password-manager-ai-agents、
  Rxcon/1password-mcp）：GitHub 搜索能查到条目与简介，但本次抓取页面返回 404，
  只作线索、未深读，不据此下结论。

## 7. 补充批次（GitHub 全量搜索回来后新增，2026-11-03）

- **BetterWright/betterwright（约 347 ★）** https://github.com/BetterWright/betterwright
  给 Agent 的常驻 Playwright 浏览器，自带加密凭据库：按 URL 门控填充、处理过的值
  从结果里脱敏、MFA 等步骤有人类接管（human handoff）。最诚实的一句：
  “凭据填充是受信任操作，不是保密保证”——填完的密码仍在页面 DOM 里。
  可借鉴：结果脱敏（不只是日志脱敏）、URL 门控、接管模式。
- **omaekumiko2-create/OpenKeylet（约 134 ★，有中文 README）**
  https://github.com/omaekumiko2-create/OpenKeylet
  本地优先的 MCP 密码管理器，无云账号：Agent 只能发现条目名/字段名/可用动作；
  每个模块有独立可见性开关（Hidden 默认 / Visible 须显式打开）；TOTP 只派生
  当前 6 位码、种子永不返回；本地活动记录不记秘密值；占位符 `{{openkeylet:module}}`
  在本地解析。可借鉴：逐模块可见性、TOTP 派生而非给种子、本地优先路线。
## 8. 第二批深读（2026-11-03 下午）

- **notinferred/keypaste（约 752 ★）** https://github.com/notinferred/keypaste
  KeePass 兼容（KDBX 单文件、无账号无网络），面向“跟 AI Agent 一起干活的人”。
  可借鉴：**批准分级**——拒绝 / 允许一次 / 允许 1 小时 / 始终允许某范围；
  申请时 Agent 必须给出理由，而批准界面明确标注“这句话是 Agent 写的，
  只能当作它的说法（claim），不能当事实”；生产环境（prod）永远要当场批准、
  不给常驻；审计日志做哈希链防篡改；引用写法 `kp://project/profile/KEY`
  可以安全提交进仓库；轮换密码永不打印。
- **OlegSotnikov/sallyport（约 190 ★）** https://github.com/OlegSotnikov/sallyport
  Mac 上的保管箱 App：**Agent 得到的是“操作”，不是钥匙**——保管箱自己执行
  已认证的 HTTP/SSH/MCP 动作；接口上根本没有“显示凭据”命令，也没有导出/
  恢复通道；授权走固定阶梯（库就绪 → 标记项逐次批准 → 新进程会话批准 →
  审计后执行），审计行由 Secure Enclave 签名。可借鉴：能让保管箱自己执行
  就不要“释放给填充器”；无揭示通道要做成接口事实、而不只是口头政策。
  它也诚实标注：目标服务把敏感数据回显在结果里时，不在隔离保证范围内。
- **sigcli/sigcli（约 292 ★）** https://github.com/sigcli/sigcli
  “给 Agent 访问权，不给凭据”：用户在真浏览器里正常登录一次（含 SSO/MFA），
  它提取并加密保存的是**会话**，而不是收走密码本身。可借鉴：凭据类型优先序——
  能 OAuth/Passkey/会话就不收密码；配置文件在校验层**结构性拒绝** secret 字段
  （写进去就报错，只能走专门的添加命令）；IdP 的 TOTP 按 host 严格后缀匹配。
- **joelhooks/agent-secrets（约 112 ★）** https://github.com/joelhooks/agent-secrets
  Age 加密 + 会话租约（TTL 自动过期）+ 自动轮换钩子 + **killswitch**
  （撤销全部租约并轮换、擦除）+ 只增不改的审计日志。可借鉴的是租约/轮换/
  killswitch；**不借鉴**的是它的默认租约会把原始值交回 Agent——对 API 令牌
  也许够用，对网站密码不合本协议的标准，这条差异要在 SPEC 里写明。
- **Garudex-Labs/caracal（约 227 ★）** https://github.com/Garudex-Labs/caracal
  口号“Authority, not credentials”：Agent 携带的是短时签名的**授权令
  （mandate）**，委派时只能缩小；网关在调用时注入真凭据；默认拒绝、
  动作前先决策；**一键 kill switch** 撤销某个 Agent 时，它派生出去的一切
  同时失效（撤销要传播）；审计是防篡改证据链，高风险动作另设人类批准门。
  可借鉴：撤销传播、决策在动作之前、授权令代替凭据。

## 对照本仓库初稿：我们已对的 / 要补的

已对（与主流实现一致）：模型只拿不透明引用、批准门、验证码短暂性、删除归用户、
如实声明“文档不能凭空变出保管箱”。

要补（已记入 SPEC 修订）：
1. 注入点补第二种：出口代理占位符替换（API 场景）。
2. Origin 以页面/请求实际目标为准，不以 Agent 口述为准；host 精确匹配、拒绝通配符。
3. 条目只写不可读回；禁止把凭据注入 URL 查询串。
4. Session 写明期限与撤销生效时间；不匹配规则时默认拒绝。
5. 增加“明文残留自查”作为一致性清单一项；增加“委派只能收窄”。
6. README 增加“能防 / 不能防”对照表（学 agent-vault-cli 的写法）。
7. 结果/回复同样要脱敏，不只是日志（学 BetterWright）；验证码优先“人类接管”模式，其次才是引用转交。
8. 可见性按条目/字段逐个设，默认隐藏、显式才可见；TOTP 只给派生码、永不给种子（学 OpenKeylet）。
9. 批准分级（一次 / 限时 / 某范围始终）+ 申请必须附理由且理由只算 Agent 的说法 + 高风险永远当场批（学 keypaste）。
10. 优先让保管箱自己执行操作、接口上不设揭示命令（学 Sallyport）；凭据优先序：OAuth/Passkey/会话 > 密码（学 sigcli）。
11. Killswitch 与撤销传播：撤销一个 Agent，它派生的租约/委派同时失效（学 caracal、agent-secrets）；审计做防篡改链。
12. 明确不学 agent-secrets 的“租约交回原始值”：本协议里密码类凭据永不以值形式回到 Agent。

我们的差异化（现成仓库里没见到的）：
- 面向**个人用户换模型**的场景：网站密码 + 验证码，而不只是开发者 API Key；
- **可粘贴的双语短版协议**，不需要部署任何东西；
- **沟通规则**：边界只说一次、不反复说教、尊重用户自己的密码管理方式。
