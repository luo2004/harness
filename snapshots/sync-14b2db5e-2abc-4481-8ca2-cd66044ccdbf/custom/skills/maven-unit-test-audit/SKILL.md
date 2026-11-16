---
name: maven-unit-test-audit
description: 在 Windows/PowerShell 下审计 Java Maven 多模块仓库的单测，把默认 mvn test 改造成纯内存快速回归：按模块显式设 JAVA_HOME、把依赖真实 MySQL/Redis/HTTP 端口的用例标 @Tag("integration") 并用 surefire excludedGroups 排除、逐模块跑 mvn -B -pl <module> test 汇报 Tests run/Failures/Errors 与耗时，最后用 brv curate 沉淀结论并用 checklist 校验改动落盘。当用户说“单测太慢/连了真实库”“把测试改成纯内存”“隔离集成测试”“逐模块验证单测”“审计单测”“把单测结论沉淀到 brv”时使用。
whenToUse: 用户要求治理 Maven 多模块项目的单元测试（提速、去外部依赖、隔离集成用例、逐模块核验结果），或要把改造结论写入 ByteRover 上下文树并用脚本确认改动确实写入文件。不适用于单纯跑一次测试或前端/浏览器测试。
---

# Maven 多模块单测审计（Windows / PowerShell）

目标：默认 `mvn test` 变成快速、纯内存的回归；真实中间件用例不删除，只隔离并可显式执行；每条结论都有命令输出支撑。

示例仓库 `E:\program\ngzda-system` 的模块与 JDK：`ngzda-svc` / `ngzda-runtime` / `ngzda-ai-rt` / `ngzda-mcp-report` 用 JDK 17（`D:\Program Files\Java\jdk-17`），`ngzda-iam` 用 JDK 21（`D:\Program Files\Java\jdk21.0.12.1`）。换仓库时先用 `Get-ChildItem 'D:\Program Files\Java'` 确认实际路径，不要照抄。

## Steps

1. **盘点测试规模与外部依赖**（不要用 `Select-String -Recurse` 扫全库，实测 120s 都跑不完）。用 grep 工具一次扫完：
   - `grep(pattern="@SpringBootTest|@DataJpaTest|@MybatisTest|@WebMvcTest|@JsonTest|@MockBean|org.testcontainers|h2database|@Sql|@ActiveProfiles", path=<repo>, include="*.java")`
   - 统计各 `src/test` 下的类数：`Get-ChildItem -Recurse -Directory -Filter test | Where-Object { $_.FullName -match "\\src\\test$" }` 再计数（这一步轻量，可以留在 PowerShell）。
   - 判定“真外部依赖”的信号：真实 DB/Redis/MQ 连接、固定本地端口（如 `127.0.0.1:28080`）、真实 HTTP 调用、落盘文件、`@SpringBootTest` 起完整上下文。

2. **逐模块确定 JDK 并显式设置**。`$env:JAVA_HOME` 是进程级的，所以一个后台任务只跑一个模块的 JDK，不要在同一会话里混用两种 JDK 的模块。开始前先 `java -version` 记录默认 JDK，避免默认 JDK 8 静默编译失败。

3. **先跑基线，逐模块串行**（`run_in_background=true`，日志 Tee 到 `tmp` 便于事后 grep）：
   ```powershell
   $env:JAVA_HOME='D:\Program Files\Java\jdk-17'
   mvn -B -pl ngzda-svc test 2>&1 | Tee-Object -FilePath tmp\test-svc.log | Select-Object -Last 60
   ```
   每个模块一条独立命令（`pb-*` 后台任务），不要合并成一个 `mvn -pl a,b test`。

4. **需要传 `-D` 时不要写多个裸参数**。PowerShell 会把 `-Dtest=!X -Dsurefire...` 拆开，mvn 报 `Unknown lifecycle phase` 并跑 0 个用例。改用环境变量或整体加引号：
   ```powershell
   $env:MAVEN_ARGS='-Dtest=!SmsDataTest -Dsurefire.failIfNoSpecifiedTests=false'
   mvn -B -pl ngzda-svc test
   $env:MAVEN_ARGS=$null   # 用完清掉，避免污染后续模块
   ```
   `-Dgroups=integration`、`-Dtest=...` 同理。

5. **给真外部依赖用例打标**。每个类两刀：先加 `import org.junit.jupiter.api.Tag;`，再在类注解块加 `@Tag("integration")`（放在 `@SpringBootTest` / `@ExtendWith` 旁的类级注解处）。
   ```java
   import org.junit.jupiter.api.Tag;
   ...
   @Slf4j
   @Tag("integration")
   public class JraftRuntimeNodeTest { ... }
   ```
   本轮示例：`SmsDataTest`（svc，真实短信/Spring 上下文）、`JraftRuntimeNodeTest`（runtime，占 28080-28082 端口）、`RedisClusterIntegrationSmokeTest`（iam，真实 Redis）、`HttpClientUtilTest`（ai-rt，真实 HTTP）。

6. **每个“有打标用例的模块”都要加 surefire 排除**——这是最容易漏的一步：只在 3 个 pom 加了 `excludedGroups`，第 4 个模块（ai-rt）的 `HttpClientUtilTest` 仍会在默认 `mvn test` 里发起外部 HTTP。逐个确认：
   ```powershell
   grep(pattern="excludedGroups|@Tag\\(\"integration\"\\)", path=<repo>, include="pom.xml,*.java")
   ```
   需要新增的模块 pom 片段（如父 pom 有 `pluginManagement`，也同步一份，保证子模块继承）：
   ```xml
   <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-surefire-plugin</artifactId>
       <configuration>
           <!-- 真外部依赖（真实端口/DB/Redis/MQ/HTTP/落盘）的用例标记 @Tag("integration")，默认不参与快速回归 -->
           <excludedGroups>integration</excludedGroups>
       </configuration>
   </plugin>
   ```

7. **改动 pom/源码前先读后改**。用 pwsh `Get-Content` 看过文件后直接 `edit` 会连续报 `file changed since it was read`；必须先用 read 工具按 `offset`/`limit` 重读同一区段，再原样重放 edit。批量文本替换涉及 BOM/换行时，按 `powershell-source-patching` 技能的 `[System.IO.File]::ReadAllText/WriteAllText` + `UTF8Encoding($false)` 方式做，并回读首字节与 CRLF/LF 计数。

8. **改造后逐模块重跑并串行验证**（同一命令模板，换 `-pl` 与 `JAVA_HOME`）。每个后台任务单独等待，用 `job_output(wait=true)` 收集，别只看最后一个。

9. **从日志里取真结论**，而不是只找 `BUILD SUCCESS`：
   ```powershell
   Select-String -Path tmp\test-svc.log -Pattern 'Tests run: \d+, Failures: \d+, Errors: \d+, Skipped: \d+$|BUILD (SUCCESS|FAILURE)|Total time:' |
     ForEach-Object { $_.Line.Trim() }
   ```
   记录每个模块的 `Tests run / Failures / Errors / Skipped` 与 `Total time`。

10. **证明被排除的用例仍可显式执行**（不能靠删测试换取速度）：
    ```powershell
    $env:MAVEN_ARGS='-Dgroups=integration'
    $env:JAVA_HOME='D:\Program Files\Java\jdk-17'
    mvn -B -pl ngzda-ai-rt test
    $env:MAVEN_ARGS=$null
    ```

11. **失败模块必须给具体根因，不能只报“断言失败，另开工单”**。看日志里的 `ERROR]   ` / `Caused by` 行，并对照已知 JDK 陷阱：JDK 21 + Mockito 会提示 `Please add Mockito as an agent to your build`（需 `-javaagent` 或 `-Dnet.bytebuddy.experimental=true`，同 surefire `argLine` 里的 `--add-opens`）；`@WebMvcTest` 切片缺 bean 时会在 `ApplicationContext` 启动阶段报 `NoSuchBeanDefinitionException`，需要 `@MockBean`/`@Import` 补齐。写明“类名 + 现象 + 判定原因”，或明确标注仍待排查。

12. **守住无关改动**。收尾前 `git diff --stat` 应只包含本轮 pom 与测试类；先把工作区快照存下来再动手：
    ```powershell
    git diff > tmp\ut-strategy.patch
    git diff --stat | Select-Object -Last 20
    ```
    发现无关文件（前端配置、证书等既有未提交改动）不要回退、不要提交。

13. **用 checklist 脚本证明每条改动真的落盘**（不是口头声称）：
    ```powershell
    $checks = @(
      @('ngzda-svc/pom.xml','excludedGroups'),
      @('ngzda-runtime/pom.xml','excludedGroups'),
      @('ngzda-iam/pom.xml','excludedGroups'),
      @('ngzda-ai-rt/pom.xml','excludedGroups'),
      @('ngzda-svc/src/test/java/com/cndatacom/ngzda/modules/service/job/util/SmsDataTest.java','@Tag("integration")')
    )
    foreach ($c in $checks) {
      if ((Get-Content -Raw $c[0]) -match [regex]::Escape($c[1])) { "OK  $($c[0])" } else { "FAIL $($c[0])" }
    }
    ```
    输出里出现任何 FAIL 就回到对应步骤修，不要进总结。

14. **沉淀到 brv 前先规避 flag 误解析**。`brv curate "-Dgroups=integration ..."` 会被当成命令行选项，报 `Nonexistent flag: -Dgroups=integration`。做法：把正文先 `write` 到 `tmp\curateN.txt`（首字符不要是 `-`），再以变量传入，并附证据文件：
    ```powershell
    $c = Get-Content -Raw tmp\curate2.txt
    brv curate $c -f pom.xml -f ngzda-svc/pom.xml -f ngzda-iam/pom.xml --detach
    ```
    多条上下文逐条 `--detach` 排队；长正文不要前台跑（会超时被 abort）。

15. **确认 curate 真的完成**：
    ```powershell
    for ($i=0; $i -lt 70; $i++) { $v = (brv curate view 2>&1 | Out-String); if ($v -notmatch 'processing') { break }; Start-Sleep -Seconds 10 }
    $v
    Get-ChildItem .brv\context-tree -Recurse -File -Filter *.md | Select-Object FullName
    ```
    期望节点落在 `.brv/context-tree/testing/`（可含派生的 `build`/`maven` 节点），状态 `completed`。

16. **汇报用对照表收尾**，一行一个模块：模块 → JDK → 用例数 → 耗时 → 结果；失败模块附具体根因；再列本轮改动文件（pom + 测试类）与 checklist 全 OK 的结论。不要只回一句“全部通过”。

## Pitfalls

- **`Select-String -Recurse` 扫全库会卡死**（>120s 被移到后台），改用 grep 工具并限定 `path`/`include`。
- **多个裸 `-D` 传给 mvn** 会被 PowerShell 拆参数，报 `Unknown lifecycle phase` 且 0 用例；用 `$env:MAVEN_ARGS` 或整体引号，用完清空。
- **只给部分模块加 `excludedGroups`** 会让漏掉的模块（本轮是 ai-rt）继续在默认测试里连真实 HTTP；按“打标类所在模块 = 必须排除的模块”逐一核对。
- **用 pwsh 读过文件后立刻 edit** 会连续报 `file changed since it was read`（本轮连挂 5 次）；先 read 工具按 offset/limit 重读再重放 edit。
- **`Set-Content -Encoding UTF8` 在 PowerShell 5.1 会写入 BOM**（首字节 `EF BB BF`），污染 Java 源文件；改用 `[System.IO.File]::ReadAllText` + `New-Object System.Text.UTF8Encoding($false)` + `WriteAllText`，并回读首字节与 CRLF/LF 计数确认换行风格未变。
- **拿 `BUILD SUCCESS` 当结论**不够：必须报出 `Tests run / Failures / Errors`；本轮 ai-rt 的 `HttpClientUtil` 超时修复只有对比 206 用例/40s 才看得出效果。
- **`brv curate` 正文以 `-` 开头**会触发 `Nonexistent flag`；写文件再传，`--detach` 排队，前台长调用会 abort。
- **只报“既有失败”不给根因**不可接受；读 `ERROR]`/`Caused by` 后指名具体原因（切片缺 bean、JDK21 Mockito agent 等）。
- 改动 pom 时注意重复依赖（例如无 version 的 `junit:junit`），顺手清理前先确认不影响其他模块按需继承。

## Verification

- 逐模块日志：`tmp\test-<module>.log` 各自出现 `Tests run: N, Failures: 0, Errors: 0` 与 `BUILD SUCCESS`，并记录 `Total time`。
- 排除生效：默认 `mvn test` 的日志里不再出现被打标类的类名（如 `SmsDataTest`、`JraftRuntimeNodeTest`、`HttpClientUtilTest`）。
- 隔离可逆：`$env:MAVEN_ARGS='-Dgroups=integration'` 重跑对应模块能执行到这些类。
- 改动范围：`git diff --stat` 只含预期 pom 与测试类；`tmp\ut-strategy.patch` 已保存快照。
- 落盘校验：checklist 脚本每行 `OK`，无 `FAIL`。
- 沉淀完成：`brv curate view` 中相关 `cur-*` 为 `completed`，`.brv/context-tree/testing/` 下存在 `.md` 节点。
