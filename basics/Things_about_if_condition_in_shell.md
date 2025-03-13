# `if [ $RUNNING_MODE -ne 4 ]; then`   语法有问题么？

在 Bash 脚本中，`if [ $RUNNING_MODE -ne 4 ]; then` 这行代码的语法是有问题的。具体问题如下：

在 Bash 中，变量引用时最好用双引号括起来，以避免变量值为空或包含空格时导致的错误。例如：

```bash
if [ "$RUNNING_MODE" -ne 4 ]; then
```

如果不加双引号，当 `$RUNNING_MODE` 的值为空时，`[ -ne 4 ]` 会被解释为：

```bash
[ -ne 4 ]
```

这会导致语法错误，因为 `-ne` 前面没有操作数。

`-ne` 是用于数值比较的运算符，它要求操作数必须是数字。如果 `$RUNNING_MODE` 的值不是数字，比如是一个字符串，那么会报错。例如：

```bash
RUNNING_MODE="abc"
if [ "$RUNNING_MODE" -ne 4 ]; then
```

这会导致错误：

```shell
[: abc: integer expression expected
```

如果 `$RUNNING_MODE` 的值可能不是数字，应该使用字符串比较运算符 `!=`。

`[[ ... ]]` 是 Bash 的扩展测试命令，它比 `[ ... ]` 更强大，支持更多特性，比如模式匹配、更严格的语法检查等。例如：

```bash
if [[ "$RUNNING_MODE" -ne 4 ]]; then
```

使用 `[[ ... ]]` 可以避免一些常见的问题，比如变量为空时的错误。

因此推荐的使用方式如下：

如果 `$RUNNING_MODE` 是数值：

```bash
if [ "$RUNNING_MODE" -ne 4 ]; then
```

或者更安全地使用 `[[ ... ]]`：

```bash
if [[ "$RUNNING_MODE" -ne 4 ]]; then
```

如果 `$RUNNING_MODE` 是字符串：

```bash
if [ "$RUNNING_MODE" != "4" ]; then
```

或者更安全地使用 `[[ ... ]]`：

```bash
if [[ "$RUNNING_MODE" != "4" ]]; then
```

### 总结

- **变量引用时加双引号**：避免变量值为空或包含空格时的错误。
- **根据变量类型选择合适的比较运算符**：数值比较用 `-ne`，字符串比较用 `!=`。
- **推荐使用 `[[ ... ]]`**：更强大、更安全。
