# 代码定位与定向验证

## 调用链

1. `src/app/context_menu.cpp` 创建 `CmdOpenTerminal` 菜单项。
2. `src/app/app_commands.cpp` 的 `TerminalTarget()` 保留现有目录选择语义；`CmdOpenTerminal` 将目录和 `s.appPrefs.terminal_program` 传给 `OpsManager::OpenTerminal()`。
3. `src/app/breadcrumb_menu.cpp`、`src/app/app_input.cpp` 中其他路径菜单入口也传入同一偏好。
4. `src/ops/ops_manager.cpp` 的 `OpenTerminal(dir, program)` 将程序复制到队列，用 `__context_terminal` 区分地址栏终端请求。
5. 打开工作线程调用 `LaunchContextTerminal()`；地址栏仍走原来的 `__terminal` / `OpenProgramIn()` 路径。
6. `src/ops/shell_command.cpp` 新增 `TerminalProgramArguments()` 和 `LaunchContextTerminal()`，再复用现有 `LaunchTerminal()` 的 ShellExecute/提权/取消/错误处理。

不要把原来的 `TerminalCommandLine()` 或 `OpenProgramIn()` 顺带改成读取配置；第一阶段明确不改地址栏。

## 偏好与设置 UI

- `src/common/terminal_program.h`：程序名称/路径校验，供 app 和 ops 共用，避免 ops 依赖 app 层。
- `src/app/app_prefs.h/.cpp`：`terminal_program` 字段、JSON、旧配置兼容及重置。
- `src/app/settings_controller.h/.cpp`：`SetTerminalProgram()` 校验并保存。
- `src/app/settings_ui.cpp`：文件选择器、恢复默认、设置搜索目的地。
- `src/app/app_runtime.cpp`：偏好映射到 VM、悬停完整路径。
- `src/ui/ui_view_model.h`：`settings_terminal_program`、`SettingsTerminalProgram` 点击类型。
- `src/ui/ui_renderer_internal.h`、`settings_layout_sections.h`：卡片和按钮几何。
- `src/ui/ui_settings_core.cpp`：绘制、长路径省略显示。
- `src/ui/ui_hit_test.cpp`、`ui_settings_view.cpp`：点击范围及搜索跳转位置。
- `tools/l10n/catalog.tsv`：新增文案 id 5630–5636；其余本地化文件由生成器更新。

## 定向测试

在 x64 Visual Studio developer prompt 中，使用已配置好的构建目录：

```powershell
cmake --build build --target pulse pulse_shell_command_test pulse_app_controllers_test pulse_terminal_settings_test
.\build\pulse_shell_command_test.exe --terminal-only
.\build\pulse_app_controllers_test.exe --terminal-preferences
.\build\pulse_terminal_settings_test.exe
```

逐个确认退出码为 0；测试失败保留诊断。`pulse_terminal_settings_test` 使用隐藏窗口和生产渲染对象，不启动完整 AppState，不改个人偏好；截图写入仓库 `bench_data/terminal-review/`。

`--terminal-only` 会启动复制到隔离目录的自建测试子程序，等待退出后清理；并非启动用户配置中的终端。真实终端还需按 README 的待办人工验证。

`--terminal-preferences` 只运行本功能偏好用例及地址栏解析用例，避免运行控制器测试里的其他功能。新增偏好用例和启动用例也已经接入对应测试程序的默认执行路径。

本地化检查需要 Python 和 `tools/l10n/requirements.txt` 中指定版本的 OpenCC：

```powershell
python tools/l10n/gen_strings.py --check
python tools/l10n/check_no_pick.py
git diff --check
```

如需改文案，只改 catalog 后运行生成器；不要手改生成文件。不要复制本机工具绝对路径作为项目依赖。

## 本机工具准备与日志

本次从空构建目录配置 Release，`PULSE_WITH_SELFTEST=OFF`、`PULSE_WITH_TEST_HOOKS=OFF`，使用已有 LumaText 包。首次构建主程序自然会构建其依赖；不是运行了所有测试或所有构建目标。

本机使用 VS 2022 的 MSVC 19.44，CMake/Ninja 来自本机另一套 VS 安装。OpenCC 1.4.2 安装在忽略的 `bench_data/terminal-review/deps/`，只用于运行生成器。新设备按仓库标准安装依赖即可，无需复刻这些路径。

本机日志位于忽略目录：

- `bench_data/terminal-review/build.log`：初次主程序和两个测试目标构建。
- `bench_data/terminal-review/ui-build.log`：界面测试首次构建。
- `bench_data/terminal-review/rebuild.log`：路径校验修正后的构建。
- `bench_data/terminal-review/final-ui-build.log`：长路径省略显示修正后的构建。

这些本机日志没有推送；实际运行结果已记录在 README 中。没有生成发布安装包，也没有创建 PR。
