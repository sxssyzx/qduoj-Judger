# QDU OJ Judger测评器

修正内容：

- 修复arm设备无法测评的故障
- 对安全问题进行分析

-------------

Judger for QDU OnlineJudge

-------------



## 修复思路

arm上无法完成测评是因为cpp的runtime与x86上的不同，导致部分调用被白名单模式过滤。因此仅需调整seccomp的策略即可。



于`/src/child.c`中，141行附近，对rule进行修正，调整为安全系数更低的general即可

原来

```cpp
// load seccomp
if (_config->seccomp_rule_name != NULL) {
    if (strcmp("c_cpp", _config->seccomp_rule_name) == 0) {
        if (c_cpp_seccomp_rules(_config) != SUCCESS) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("c_cpp_file_io", _config->seccomp_rule_name) == 0) {
        if (c_cpp_file_io_seccomp_rules(_config) != SUCCESS) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("general", _config->seccomp_rule_name) == 0) {
        if (general_seccomp_rules(_config) != SUCCESS ) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("golang", _config->seccomp_rule_name) == 0) {
        if (golang_seccomp_rules(_config) != SUCCESS ) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("node", _config->seccomp_rule_name) == 0) {
        if (node_seccomp_rules(_config) != SUCCESS ) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    ......
```

现在

```cpp
// load seccomp
// patched by Haolin
if (_config->seccomp_rule_name != NULL) {
    if (strcmp("c_cpp", _config->seccomp_rule_name) == 0) {
        // if (c_cpp_seccomp_rules(_config) != SUCCESS) {
        if (general_seccomp_rules(_config) != SUCCESS) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("c_cpp_file_io", _config->seccomp_rule_name) == 0) {
        // if (c_cpp_file_io_seccomp_rules(_config) != SUCCESS) {
        if (general_seccomp_rules(_config) != SUCCESS) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("general", _config->seccomp_rule_name) == 0) {
        if (general_seccomp_rules(_config) != SUCCESS ) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("golang", _config->seccomp_rule_name) == 0) {
        // if (golang_seccomp_rules(_config) != SUCCESS ) {
        if (general_seccomp_rules(_config) != SUCCESS ) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    else if (strcmp("node", _config->seccomp_rule_name) == 0) {
        // if (node_seccomp_rules(_config) != SUCCESS ) {
        if (general_seccomp_rules(_config) != SUCCESS ) {
            CHILD_ERROR_EXIT(LOAD_SECCOMP_FAILED);
        }
    }
    ......
```





## 安全隐患

审计代码后，发现Judger本身并不安全！整套Judger完全依托于docker的沙箱机制提供安全兜底。

- cpp和c语言测评下采用系统api白名单模式，但带来兼容性问题。同时，正如作者在文档中指出，用户可以使用mmap绕过限制进而入侵测评环境。
- 在其他语言下使用系统api黑名单模式，仅屏蔽fork和socket等操作

直接后果是，用户如果知道Judger的判题逻辑和目录结构，极有可能攻击并且成功（~~自动AC机~~）



## 未来

有两个方向

- 更换Windows平台，使用操作系统级的限制应可以解决类似安全问题，但代码丢失跨平台性。
- 使用customized compiler，在编译期就让这些漏洞不可能发生，但需对每种语言的编译器都进行相应调整。



个人倾向于后者
