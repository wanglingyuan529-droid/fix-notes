# PowerShell 5.1 往返读写 UTF-8 中文源码：闭合引号被吞，编译报"未结束的字符串文字"

> 日期：2026-09-17 · 环境：Windows 11 (x86_64) · PowerShell 5.1 · JDK 25

## 现象

批量给 14 个 Java 文件套 `try-with-resources` 后编译失败，报的是**语法错误**：

```
TestAge.java:16: 错误: 未结束的字符串文字
    System.out.println("请输入性别：");
                       ^
```

但代码是我刚改的，结构上没问题。细看那几行：**中文串的闭合引号不见了**。

## 根因

那批文件第一步（Node 脚本转换）是好的，坏在第二步——我用 PowerShell 做"清理多余分号"：

```powershell
$t = Get-Content -LiteralPath $file -Raw        # ← 元凶
$t = $t -replace 'new Scanner\(System\.in\);\)', 'new Scanner(System.in))'
[System.IO.File]::WriteAllText($file, $t, [System.Text.UTF8Encoding]::new($false))
```

**PowerShell 5.1 的 `Get-Content` 默认用系统 ANSI 代码页（中文 Windows = GBK/936）解码**，
而文件是 **UTF-8 无 BOM**。中文在 UTF-8 下是 3 字节，被当 GBK 读时：

```
"性别" 的 UTF-8 字节：E6 80 A7 E5 88 AB
按 GBK 解：E6 80 → 一个汉字，A7 E5 → 一个汉字，88 AB → 一个汉字
```

GBK 是**双字节且前导字节 ≥ 0x81**，只要配对时"吞掉"了紧随其后的 `"`(0x22) 或 `\`，
**那个引号就凭空消失** —— 于是字符串没闭合，语法直接崩。

关键点：**读进来的字符串已经错了**，再按 UTF-8 写回去只是把错误固化，损坏不可逆。

## 步骤（恢复）

### 1. 回滚到损坏前的干净提交

坏提交从未推送过，直接丢弃：

```powershell
git reset --hard c61d18c     # 迁移到 Maven 布局那次提交（干净、UTF-8 正确）
```

### 2. 用 Node 一步做完（不再分两步）

```javascript
const fs = require("fs");
const src = fs.readFileSync(full, "utf8");          // 明确 utf8
// ...转换...
fs.writeFileSync(full, out.join(eol), "utf8");      // 明确 utf8
```

顺带把第一步遗留的小瑕疵（`try (Scanner sc = ...;)` 的多余分号）在同一个正则里解决，
**一次成型，不存在"第二步再修一下"的机会**。

### 3. 编译验证

```powershell
javac -encoding UTF-8 -d <临时目录> @<源文件列表>
# → exit 0，69 源文件 → 69 class
```

## 坑

1. **损坏是静默的**：编辑器里看着只是"乱码"，实际**语法的引号已经没了**——
   乱码看得见，被吞的引号看不见。
2. **同一批文件分两步改、其中一步用错工具，全毁**。批量文本处理要么一步到位，要么每步都验证。
3. **`Get-Content`/`Set-Content` 在 PS 5.1 没有安全的 UTF-8 默认值**（`-Encoding utf8` 还会写入 BOM，是另一个坑）；
   PowerShell 7+ 默认 UTF-8 无 BOM，但本机不一定有。
4. **javac 默认编码不用管**：JDK 18+（JEP 400）源码默认按 UTF-8 读，
   所以干净状态下**不带 `-encoding` 也能编译通过**——这次的失败是真实字节损坏，不是编码参数问题。

## 验证

- `javac` 全量编译 exit 0
- 抽查被损坏过的文件（`TestAge.java`、`StringDemo5.java`）：中文提示文本完整、`try (...)` 无多余分号
- 运行行为不变（学生管理系统输入 `4` 正常打印"再见"退出）

## 教训

1. **凡是涉及中文/UTF-8 的批量文本处理，不要用 PowerShell 5.1 做 `Get-Content` → 改 → `Set-Content` 往返。**
   用 Node / Python（显式指定 `utf8`），或用编辑工具，它们对编码是确定的。
2. **改完必须编译/校验**，不能只看文件"写进去了"——这次是编译器把静默损坏揪出来的。
3. **未推送的坏提交可以放心丢弃**：`git reset --hard` 是本场景最干净的止损，
   比"在坏文件上继续修补"可靠得多。
4. **一步一验证**：批量脚本要么一口气做完，要么每步跑一次校验；中间插入一次工具切换，风险就翻倍。
