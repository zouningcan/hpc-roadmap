# 模块 09 练习：安全和密码学

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。

## A 概念题

**A1** 计算并比较两种密码的熵：a) 从 10 万词的词典随机挑 4 个词；b) 8 个随机字母数字字符（62 种字符）。哪个更强？
*考察点：熵公式 log₂(可能性ⁿ) 的应用——反直觉：四个普通词往往更强。*

**A2** 哈希函数的四条性质是什么？"同输入必同输出"这条为什么是它无数应用的前提？
*考察点：确定性是完整性校验/内容寻址的根基。*

**A3** 对称和非对称加密各用什么钥匙结构？SSH 登录时"私钥从不离开本机"是怎么做到的（哪种操作、谁持有哪把钥匙）？
*考察点：两把钥匙模型；挑战-应答签名的流程。*

**A4** 为什么存密码要用"故意慢"的 KDF 而不是直接 sha256？盐（salt）解决什么问题？
*考察点：慢 = 爆炸成本；盐防彩虹表/同密码同哈希。*

**A5** 🕷 你的 gh token 有 `delete_repo` 和 `admin:org` 权限，但只需要推一个小仓库。这违反了什么原则？正确的做法是什么？
*考察点：最小权限原则——本项目真实案例。*

## B 动手题

**B1（哈希手感的实验）**：
```bash
printf 'hello' | sha256sum
printf 'Hello' | sha256sum     # 只差一个字母，哈希差多少？
printf 'hello' | sha256sum     # 再跑一遍，变吗？
touch f1 && cp f1 f2 && sha256sum f1 f2    # 内容相同的两个文件呢？
```
回答：哪两条性质分别被第 2、3 个实验印证？
*考察点：雪崩效应（差之毫厘谬以千里）与确定性。*

**B2（官方练习：AES 加解密往返）**：
```bash
echo "top secret 超算配置" > secret.txt
openssl aes-256-cbc -salt -in secret.txt -out secret.enc    # 输入一个口令
cat secret.enc | head -c 100                                 # 密文长什么样？
openssl aes-256-cbc -d -in secret.enc -out secret.dec        # 同口令解密
cmp secret.txt secret.dec && echo "往返一致"
# 再试：解密时故意输错口令，观察现象
rm secret.txt secret.dec secret.enc
```
*考察点：openssl 对称加密的实际操作；口令即密钥材料的来源（KDF）。*

**B3（SSH 密钥与指纹，承接模块 05）**：
```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub    # 查看公钥指纹
cat ~/.ssh/id_ed25519.pub | head -c 60  # 公钥开头长什么样（算法名）
```
回答：指纹是公钥的什么（哈希）？为什么登录新服务器时核对指纹能防中间人攻击？
*考察点：信任首次使用；HelloHPC 签到题同款知识。*

## C 挑战题

**C1（官方练习：镜像校验）** 模拟"从不可信镜像下载、用可信渠道核对"：
```bash
# 1. 造一个"官方发布"的文件与校验和
dd if=/dev/urandom of=bigdata.bin bs=1M count=10 2>/dev/null
sha256sum bigdata.bin > SHA256SUMS
# 2. 模拟镜像被篡改：改一个字节
cp bigdata.bin mirror-download.bin
printf 'x' | dd of=mirror-download.bin bs=1 seek=100 conv=notrunc 2>/dev/null
# 3. 校验
sha256sum -c <(sed 's/bigdata.bin/mirror-download.bin/' SHA256SUMS)
```
`-c` 报了什么？这个流程为什么"镜像不可信也没关系"？
*考察点：完整性校验的信任模型——哈希清单必须走可信渠道。*

**C2（熵与爆破时间估算）** 按 10⁴ 次/秒的爆破速度，估算 A1 两种密码各需多久被穷举（给出计算过程）。再回答：为什么"在线场景 40 bit 够、离线场景要 80 bit"？
*考察点：熵→时间的换算；威胁模型决定安全边界。*

**C3（签名提交，进阶）** 给 git 配置 GPG 签名（`git commit -S`），或至少解释：`git tag -s v1.0` 与 `git tag -a v1.0` 的区别，以及接收方 `git tag -v` 验证的是什么。
*考察点：私钥签名/公钥验证在发布流程中的落地。*
