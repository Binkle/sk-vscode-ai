# code.sh

``` mermaid
flowchart TD
    A[开始] --> B{确定操作系统类型}
    
    B -->|macOS| C[定义 realpath 函数]
    C --> D[设置 ROOT 路径]
    
    B -->|其他操作系统| E[设置 ROOT 路径]
    E --> F{检测是否为 WSL 环境}
    F -->|是| G[设置 IN_WSL=true]
    F -->|否| H
    D --> H
    G --> H
    
    H[定义 code 函数] --> I[切换到项目根目录]
    I --> J[根据操作系统确定应用名称和可执行文件路径]
    J --> K{是否设置 VSCODE_SKIP_PRELAUNCH}
    K -->|否| L[执行 preLaunch.js 脚本]
    K -->|是| M
    L --> M
    
    M{是否以 --builtin 参数启动} -->|是| N[启动内置扩展管理模式并返回]
    M -->|否| O[设置开发环境变量]
    O --> P[设置测试扩展配置]
    P --> Q{是否包含 --extensionTestsPath 参数}
    Q -->|是| R[不禁用测试扩展]
    Q -->|否| S[禁用测试扩展]
    R --> T[启动 VS Code]
    S --> T
    
    H --> U[定义 code-wsl 函数]
    U --> V[获取 WSL 主机 IP]
    V --> W[设置 DISPLAY 环境变量]
    W --> X{Electron 可执行文件是否存在}
    X -->|是| Y[设置 WSL 环境变量]
    Y --> Z[获取 WSL 扩展位置]
    Z --> AA{是否找到 WSL 扩展}
    AA -->|是| AB[使用 wslCode-dev.sh 启动 VS Code]
    
    H --> AC{主脚本执行逻辑}
    AC --> AD{是否在 WSL 环境且无 DISPLAY 变量}
    AD -->|是| AE[调用 code-wsl 函数]
    AD -->|否| AF{是否在 WSLg 环境}
    AF -->|是| AG[使用 --disable-gpu 参数调用 code 函数]
    AF -->|否| AH{是否在 Docker 环境}
    AH -->|是| AI[使用 --disable-dev-shm-usage 参数调用 code 函数]
    AH -->|否| AJ[直接调用 code 函数]
    
    AE --> AK[返回最后一个命令的退出码]
    AG --> AK
    AI --> AK
    AJ --> AK
    AK --> AL[结束]
```
``` SHELL
#!/usr/bin/env bash

# 设置脚本遇到错误就退出
set -e

# 根据操作系统类型确定项目根目录路径和环境
# $OSTYPE 是一个 Bash 内置的环境变量，它是由 Bash shell 自动设置的，无需用户手动定义。这个变量包含了当前操作系统的类型信息。
# Bash 在启动时会自动检测操作系统类型并设置 $OSTYPE 变量。常见的值包括：
# - darwin* - 表示 macOS 系统
# - linux-gnu - 表示大多数 Linux 发行版
# - msys - 表示 Git Bash 或 MinGW 环境 (Windows)
# - cygwin - 表示 Cygwin 环境 (Windows)
if [[ "$OSTYPE" == "darwin"* ]]; then
	# 这行代码定义了一个名为 realpath 的自定义函数，用于获取文件的绝对路径。在 macOS 系统中，默认不包含 realpath 命令（而 Linux 系统通常有此命令），所以脚本自己实现了这个功能。
	# 1. realpath() - 定义一个名为 realpath 的函数
	# 2. [[ $1 = /* ]] - 判断第一个参数 $1 是否以 / 开头：
	# 	- $1 是函数的第一个参数，即传入的路径
	# 	- /* 匹配所有以 / 开头的字符串
	# 	- 这实际上是在检查路径是否已经是绝对路径（在 Unix/Linux/macOS 中，绝对路径都以 / 开头）
	# 3. && - 逻辑与操作符，如果前面的条件为真，则执行后面的命令
	# 4. echo "$1" - 如果路径已经是绝对路径，直接输出这个路径
	# 5. || - 逻辑或操作符，如果前面的条件为假，则执行后面的命令
	# 6. echo "$PWD/${1#./}" - 如果路径不是绝对路径，则将其转换为绝对路径：
	# 	- $PWD 是当前工作目录的环境变量
	# 	- ${1#./} 是一个参数展开，它从参数 $1 中移除开头的 ./（如果存在）
	# 		- 如果路径是 ./file.txt，则 ${1#./} 会得到 file.txt
	# 		- 如果路径是 file.txt，则 ${1#./} 仍然是 file.txt
	# 	- 通过将当前目录和处理后的路径拼接，得到绝对路径
	realpath() { [[ $1 = /* ]] && echo "$1" || echo "$PWD/${1#./}"; }
	# 1. $0 - 这是一个特殊变量，表示当前脚本的名称或路径。例如，如果脚本是通过 ./scripts/code.sh 运行的，$0 的值就是 ./scripts/code.sh
	# 2. "$(realpath "$0")" - 使用前面定义的 realpath 函数将脚本路径转换为绝对路径
	# 	- 例如，如果 $0 是 ./scripts/code.sh，而当前目录是 /Users/user/vscode
	# 	- 则 realpath "$0" 的结果将是 /Users/user/vscode/scripts/code.sh
	# 3. dirname "$(realpath "$0")" - 使用 dirname 命令获取上述路径的目录部分
	# 	- 接着上面的例子，这会返回 /Users/user/vscode/scripts
	# 4. $(dirname "$(realpath "$0")") - 将上一步的结果作为值通过命令替换获取
	# 5. dirname "$(dirname "$(realpath "$0")")" - 再一次使用 dirname 获取上一级目录
	# 	- 继续我们的例子，这会返回 /Users/user/vscode
	# 6. ROOT=$(...) - 将最终得到的路径赋值给 ROOT 变量
	ROOT=$(dirname "$(dirname "$(realpath "$0")")")
else
	# 在 Linux 上获取项目根目录
	# 1. readlink -f $0 - 使用 readlink 命令获取 $0 的绝对路径
	# 	- 例如，如果 $0 是 ./scripts/code.sh，readlink -f $0 的结果将是 /Users/user/vscode/scripts/code.sh
	# 2. dirname "$(readlink -f $0)" - 使用 dirname 命令获取上述路径的目录部分
	# 	- 接着上面的例子，这会返回 /Users/user/vscode/scripts
	# 3. dirname "$(dirname "$(readlink -f $0)")" - 再一次使用 dirname 获取上一级目录
	# 	- 继续我们的例子，这会返回 /Users/user/vscode
	ROOT=$(dirname "$(dirname "$(readlink -f $0)")")
	# 检测是否在 WSL 环境中运行
	# 1. grep -qi Microsoft /proc/version：
	# 	- grep 是一个文本搜索工具，用于在文件中查找指定模式
	# 	- -q 参数表示"静默模式"，不输出匹配结果，只返回退出状态（找到匹配返回0，否则返回非0值）
	# 	- -i 参数表示忽略大小写，这样无论 "Microsoft" 是大写还是小写都能匹配
	# 	- /proc/version 是 Linux 系统中的一个特殊文件，包含了内核版本信息
	# 	- 在 WSL 环境中，这个文件通常包含 "Microsoft" 字样，因为 WSL 是由微软开发的
	# 2. &&：
	# 	- 这是逻辑与运算符，只有当左侧命令成功执行（返回状态码为0）时，才会执行右侧命令
	# 	- 在这里，只有当在 /proc/version 中找到 "Microsoft" 时，才会继续检查 powershell.exe 是否存在
	# 3. type powershell.exe > /dev/null 2>&1：
	# 	- type 命令用于显示命令的类型（内置命令、外部命令等）或位置
	# 	- powershell.exe 是要检查的命令，它是 Windows PowerShell 的可执行文件，只有在 Windows 环境或 WSL 中才能找到
	# 	- > /dev/null 将标准输出重定向到 /dev/null（相当于丢弃输出）
	# 	- 2>&1 将标准错误（文件描述符2）重定向到标准输出（文件描述符1），也就是说标准错误也会被丢弃
	# 	- 结合起来，这个命令只检查 powershell.exe 是否存在，不显示任何输出
	# 4. then IN_WSL=true：
	# 	- 如果前面的条件都满足（即找到 "Microsoft" 且 powershell.exe 命令存在），则执行这条语句
	# 	- IN_WSL=true 将变量 IN_WSL 设置为 true，标记当前环境为 WSL 环境
	if grep -qi Microsoft /proc/version && type powershell.exe > /dev/null 2>&1; then
		IN_WSL=true
	fi
fi

# 主要的 code 函数，用于在开发环境中启动 VS Code
function code() {
	# 切换到项目根目录
	cd "$ROOT"

	# 根据操作系统确定应用名称和可执行文件路径
	if [[ "$OSTYPE" == "darwin"* ]]; then
		# 获取 macOS 应用程序名称
		NAME=`node -p "require('./product.json').nameLong"`
		CODE="./.build/electron/$NAME.app/Contents/MacOS/Electron"
	else
		# 获取 Linux/Windows 应用程序名称
		NAME=`node -p "require('./product.json').applicationName"`
		CODE=".build/electron/$NAME"
	fi

	# 预启动处理：获取 electron、编译代码、处理内置扩展等
	# 1. if [[ -z "${VSCODE_SKIP_PRELAUNCH}" ]]; then：
	# 	- [[ ... ]] 是 Bash 的条件测试结构，比传统的 [ ... ] 提供更多功能
	# 	- -z 是一个字符串测试操作符，用于检查字符串是否为空（zero length）
	# 	- "${VSCODE_SKIP_PRELAUNCH}" 获取环境变量 VSCODE_SKIP_PRELAUNCH 的值
	# 		- ${} 是参数扩展语法
	# 		- 双引号确保即使变量包含空格也能正确处理
	# 	- 整个条件意味着：如果环境变量 VSCODE_SKIP_PRELAUNCH 未设置或为空，则执行下面的代码块
	# 2. node build/lib/preLaunch.js：
	# 	- 使用 Node.js 运行 build/lib/preLaunch.js 脚本
	# 	- 这个脚本可能执行以下任务（根据注释和上下文推断）：
	# 		- 获取和设置 Electron（VS Code 运行的基础框架）
	# 		- 编译必要的源代码
	# 		- 处理内置扩展
	# 		- 其他启动 VS Code 之前需要的准备工作
	# 3. fi：
	# 	- 结束 if 条件块
	# 调用 例如   VSCODE_SKIP_PRELAUNCH=1 ./scripts/code.sh
	if [[ -z "${VSCODE_SKIP_PRELAUNCH}" ]]; then
		node build/lib/preLaunch.js
	fi

	# 特殊模式：管理内置扩展
	# 1. if [[ "$1" == "--builtin" ]]; then：
	# 	- $1 是脚本接收的第一个命令行参数
	# 	- [[ "$1" == "--builtin" ]] 检查这个参数是否等于字符串 --builtin
	# 	- 如果条件成立，则执行下面的代码块
	# 2. exec "$CODE" build/builtin：
	# 	- exec 命令用于执行给定的命令，并让该命令取代当前的 shell 进程，不会创建新进程
	# 	- $CODE 是前面定义的变量，包含 VS Code 执行文件的路径（在 macOS 上是 ./.build/electron/$NAME.app/Contents/MacOS/Electron，在其他系统上是 .build/electron/$NAME）
	# 	- build/builtin 是传递给 VS Code 的参数，指定要处理内置扩展
	# 	- 整个命令的意思是：用 VS Code 可执行文件替换当前 shell 进程，并让 VS Code 处理内置扩展
	# 3. return：
	# 	- 这个命令会从当前函数（即 code 函数）返回
	# 	- 由于前面使用了 exec，这行实际上不会被执行，因为 exec 成功后当前脚本就已经被替换了
	# 	- 但是，如果 exec 因某种原因失败了，这个 return 会确保函数不继续执行下面的代码
	if [[ "$1" == "--builtin" ]]; then
		exec "$CODE" build/builtin
		return
	fi

	# 设置开发环境配置变量
	# Configuration
	# export NODE_ENV=development：
	#	- 设置 Node.js 环境为开发模式
	#	- 这会影响许多 Node.js 应用和库的行为，通常会启用更详细的日志记录、错误消息和调试功能
	#	- 生产环境通常设置为 production，会优化性能并减少日志输出
	export NODE_ENV=development
	# export VSCODE_DEV=1：
	#	- 标记当前环境为 VS Code 开发环境
	#	- 这会启用 VS Code 的开发特性，如源码映射、热重载等
	#	- 可能会影响 VS Code 的许多内部行为，使其更适合开发和调试
	export VSCODE_DEV=1
	# export VSCODE_CLI=1：
	#	- 标记当前环境为 VS Code CLI 模式
	#	- 这会启用 VS Code 的命令行接口（CLI）模式
	#	- 可能会影响 VS Code 的许多内部行为，使其更适合开发和调试
	export VSCODE_CLI=1
	# export ELECTRON_ENABLE_STACK_DUMPING=1：
	#	- 启用 Electron 的堆栈跟踪功能
	#	- 当 Electron 应用程序崩溃时，会生成详细的堆栈跟踪信息
	#	- 有助于诊断崩溃原因
	export ELECTRON_ENABLE_STACK_DUMPING=1
	# export ELECTRON_ENABLE_LOGGING=1：
	#	- 启用 Electron 的日志记录功能
	#	- 这会生成详细的日志信息，有助于调试 Electron 应用程序
	#	- 可能会影响 Electron 的性能，因此只在需要调试时使用
	export ELECTRON_ENABLE_LOGGING=1

	# 设置是否禁用测试扩展
	# 1. DISABLE_TEST_EXTENSION="--disable-extension=vscode.vscode-api-tests"：
	#	- 设置一个变量，其值是用于禁用 VS Code API 测试扩展的命令行参数
	#	- vscode.vscode-api-tests 是 VS Code 的 API 测试扩展，用于测试 VS Code 扩展 API
	DISABLE_TEST_EXTENSION="--disable-extension=vscode.vscode-api-tests"
	# 2.if [[ "$@" == *"--extensionTestsPath"* ]]; then：
	#	- $@ 表示传递给脚本的所有参数
	#	- *"--extensionTestsPath"* 是一个模式匹配，检查参数列表中是否包含 --extensionTestsPath 字符串
	#	- 这是检测是否正在运行扩展测试的一种方式
	# 3. DISABLE_TEST_EXTENSION=""：
	#	- 如果检测到运行扩展测试（通过 --extensionTestsPath 参数），则清空 DISABLE_TEST_EXTENSION 变量
	#	- 这意味着在运行扩展测试时，不会禁用 API 测试扩展，因为测试可能需要它
	if [[ "$@" == *"--extensionTestsPath"* ]]; then
		DISABLE_TEST_EXTENSION=""
	fi

	# 启动 VS Code
	# Launch Code
	# 1. exec "$CODE" . $DISABLE_TEST_EXTENSION "$@"：
	#	- "$CODE" 是前面定义的变量，包含 VS Code 可执行文件的路径
	#	- . 表示当前目录
	#	- $DISABLE_TEST_EXTENSION 是前面定义的变量，包含禁用 API 测试扩展的命令行参数
	#	- "$@" 表示传递给脚本的所有参数
	exec "$CODE" . $DISABLE_TEST_EXTENSION "$@"
}

# WSL 环境下的特殊启动函数
function code-wsl()
{
	# 获取 WSL 主机 IP 地址，用于设置 X11 显示
	# 1. echo "" | powershell.exe -noprofile -Command "& {(Get-NetIPAddress | Where-Object {\$_.InterfaceAlias -like '*WSL*' -and \$_.AddressFamily -eq 'IPv4'}).IPAddress | Write-Host -NoNewline}"：
	#	- echo "" 是输出一个空字符串，用于捕获命令的输出
	#	- powershell.exe 是 Windows PowerShell 的可执行文件
	#	- -noprofile 参数表示不加载 PowerShell 配置文件
	#	- -Command 参数表示执行指定的命令
	#	- Get-NetIPAddress 是 PowerShell 的一个 cmdlet，用于获取网络接口的 IP 地址
	#	- Where-Object 是 PowerShell 的一个 cmdlet，用于过滤对象
	#	- \$_.InterfaceAlias -like '*WSL*' 是 PowerShell 的一个表达式，用于匹配接口别名中包含 'WSL' 的网络接口
	#	- \$_.AddressFamily -eq 'IPv4' 是 PowerShell 的一个表达式，用于匹配地址族为 IPv4 的网络接口
	#	- .IPAddress 是 PowerShell 的一个属性，用于获取网络接口的 IP 地址
	#	- Write-Host -NoNewline 是 PowerShell 的一个 cmdlet，用于输出结果并禁止换行
	HOST_IP=$(echo "" | powershell.exe -noprofile -Command "& {(Get-NetIPAddress | Where-Object {\$_.InterfaceAlias -like '*WSL*' -and \$_.AddressFamily -eq 'IPv4'}).IPAddress | Write-Host -NoNewline}")
	# 2. export DISPLAY="$HOST_IP:0"：
	#	- 设置 DISPLAY 环境变量，用于跨平台通信
	#	- $HOST_IP 是前面获取的 WSL 主机 IP 地址
	#	- 0 是 X11 显示的默认端口号
	#	- 这个命令将 WSL 主机的 IP 地址和端口号组合成 DISPLAY 环境变量
	export DISPLAY="$HOST_IP:0"

	# 在 WSL shell 中运行
	# 1. ELECTRON="$ROOT/.build/electron/Code - OSS.exe"：
	#	- 设置 ELECTRON 变量，包含 VS Code 可执行文件的路径
	#	- $ROOT 是前面定义的变量，包含项目根目录的路径
	#	- .build/electron/Code - OSS.exe 是 VS Code 可执行文件的路径
	ELECTRON="$ROOT/.build/electron/Code - OSS.exe"
	# 2. if [ -f "$ELECTRON"  ]; then：
	#	- 检查 ELECTRON 变量指定的文件是否存在
	if [ -f "$ELECTRON"  ]; then
		# 3. local CWD=$(pwd)：
		#	- 获取当前工作目录
		#	- local 是一个关键字，用于声明一个局部变量
		#	- CWD 是当前工作目录的缩写
		#	- $(pwd) 是一个命令替换，用于获取当前工作目录的完整路径
		local CWD=$(pwd)
		# 4. cd $ROOT：
		#	- 切换到项目根目录
		#	- $ROOT 是前面定义的变量，包含项目根目录的路径
		cd $ROOT
		# 5. export WSLENV=ELECTRON_RUN_AS_NODE/w:VSCODE_DEV/w:$WSLENV：
		#	- 设置 WSL 环境变量，用于跨平台通信
		#	- ELECTRON_RUN_AS_NODE/w 表示将 ELECTRON_RUN_AS_NODE 环境变量传递给子进程
		export WSLENV=ELECTRON_RUN_AS_NODE/w:VSCODE_DEV/w:$WSLENV
		# 6. local WSL_EXT_ID="ms-vscode-remote.remote-wsl"：
		#	- 设置 WSL_EXT_ID 变量，包含 WSL 扩展的 ID
		local WSL_EXT_ID="ms-vscode-remote.remote-wsl"
		# 7.查找 WSL 扩展的位置
		# 	- echo "" | VSCODE_DEV=1 ELECTRON_RUN_AS_NODE=1 "$ROOT/.build/electron/Code - OSS.exe" "out/cli.js" --locate-extension $WSL_EXT_ID：
		#		- echo "" 是输出一个空字符串，用于捕获命令的输出
		#		- VSCODE_DEV=1 设置 VS Code 开发环境变量
		#		- ELECTRON_RUN_AS_NODE=1 设置 ELECTRON_RUN_AS_NODE 环境变量
		#		- "$ROOT/.build/electron/Code - OSS.exe" 是 VS Code 可执行文件的路径
		#		- "out/cli.js" 是 VS Code 的命令行接口脚本
		#		- --locate-extension $WSL_EXT_ID 是传递给 VS Code 的参数，用于查找 WSL 扩展的位置
		local WSL_EXT_WLOC=$(echo "" | VSCODE_DEV=1 ELECTRON_RUN_AS_NODE=1 "$ROOT/.build/electron/Code - OSS.exe" "out/cli.js" --locate-extension $WSL_EXT_ID)
		# 7. cd $CWD：
		#	- 切换回原来的工作目录
		#	- $CWD 是前面获取的当前工作目录
		cd $CWD
		# 8. if [ -n "$WSL_EXT_WLOC" ]; then：
		#	- 检查 WSL_EXT_WLOC 变量是否不为空
		#	- 如果 WSL_EXT_WLOC 不为空，则执行下面的代码块
		if [ -n "$WSL_EXT_WLOC" ]; then
			# local WSL_CODE=$(wslpath -u "${WSL_EXT_WLOC%%[[:cntrl:]]}")/scripts/wslCode-dev.sh
			# 1. wslpath -u "${WSL_EXT_WLOC%%[[:cntrl:]]}"：
			#		- wslpath 是 Windows Subsystem for Linux 的一个命令，用于将 WSL 路径转换为 Windows 路径
			#		- -u 参数表示将 WSL 路径转换为 Windows 路径
			#		- "${WSL_EXT_WLOC%%[[:cntrl:]]}" 是 WSL_EXT_WLOC 变量的值，去除其中的换行符
			# 2. /scripts/wslCode-dev.sh：
			#		- 这是 WSL 扩展的脚本路径
			local WSL_CODE=$(wslpath -u "${WSL_EXT_WLOC%%[[:cntrl:]]}")/scripts/wslCode-dev.sh
			# 9. $WSL_CODE "$ROOT" "$@"：
			#		- 执行 WSL_CODE 脚本，传递项目根目录和所有参数
			#		- "$ROOT" 是项目根目录的路径
			#		- "$@" 是传递给脚本的所有参数
			$WSL_CODE "$ROOT" "$@"
			# 10. exit $?：
			#		- 退出脚本，并返回最后一个命令的退出码
			exit $?
		else
			# 11. echo "Remote WSL not installed, trying to run VSCode in WSL."：
			#		- 输出提示信息，表示 WSL 扩展未安装，尝试在 WSL 中运行 VS Code
			echo "Remote WSL not installed, trying to run VSCode in WSL."
		fi
	fi
}

# 主脚本执行逻辑，根据不同环境选择合适的启动方式
# 1. if [ "$IN_WSL" == "true" ] && [ -z "$DISPLAY" ]; then：
#		- 检查 IN_WSL 变量是否为 true，并且 DISPLAY 变量是否为空
#		- 如果条件成立，则执行下面的代码块
if [ "$IN_WSL" == "true" ] && [ -z "$DISPLAY" ]; then
	# 2. code-wsl "$@"：
	#		- 调用 code-wsl 函数，传递所有参数
	code-wsl "$@"
# 3. elif [ -f /mnt/wslg/versions.txt ]; then：
#		- 检查 /mnt/wslg/versions.txt 文件是否存在
#		- 如果条件成立，则执行下面的代码块
elif [ -f /mnt/wslg/versions.txt ]; then
	# 4. code --disable-gpu "$@"：
	#		- 调用 code 函数，传递所有参数，并禁用 GPU
	code --disable-gpu "$@"
# 5. elif [ -f /.dockerenv ]; then：
#		- 检查 /.dockerenv 文件是否存在
#		- 如果条件成立，则执行下面的代码块
elif [ -f /.dockerenv ]; then
	# Docker 环境下的特殊处理
	# Workaround for https://bugs.chromium.org/p/chromium/issues/detail?id=1263267
	# Chromium does not release shared memory when streaming scripts
	# which might exhaust the available resources in the container environment
	# leading to failed script loading.
	# 6. code --disable-dev-shm-usage "$@"：
	#		- 调用 code 函数，传递所有参数，并禁用共享内存
	code --disable-dev-shm-usage "$@"
else
	# 默认情况下直接调用 code 函数
	# 7. code "$@"：
	#		- 调用 code 函数，传递所有参数
	code "$@"
fi

# 脚本结束，返回最后一个命令的退出码
exit $?

``` 