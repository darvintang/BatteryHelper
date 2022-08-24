# 利用SMC保护MacBookPro的电池



​        因锂电池的特性，最好不要总是保持最满电量，虽然说macOS已经有了优化电池充电，但是它的优化学习时间长，基本上不生效。于是研究[AlDente](https://github.com/davidwernhart/AlDente)发现它利用SMC实现了在笔记本连接电源的情况下限制充电。

​        然后从[smcFanControl](https://github.com/hholtmann/smcFanControl)找到了相对较成熟的SMC，[AlDente](https://github.com/davidwernhart/AlDente)内有`swift`实现的SMC，但它打包需要再次封装，为了省时间降低门槛我选择了[smcFanControl](https://github.com/hholtmann/smcFanControl)，该项目仓库里的`smc-command`文件夹下就是SMC的源码，C语言实现，在该文件夹下`make`一下就编译好SMC的可执行文件了。(可执行文件在文章附件压缩包内)

​        SMC直接使用比较麻烦，所以就写了一个脚本操作SMC。

​        SMC的使用方法：
```shell
smc -f #获取风扇信息
smc -k key -r #读取指定key的值
smc -k key -w value #设置指定key的值，设置需要root权限
```

​        通过查询资料找到了几个关键的key

​        BRSC：电池真实电量比

​        B0UC：系统显示的电量比

​        B0RM：当前电池容量

​        AC-W：电源连接情况

​        CH0B：充电状态，通过修改该key的值来限制电池充电

​        然后就以上5个key 封装了一个简单的限制充电的脚本

```shell
#/bin/zsh
set -e

# smc可执行文件路径，自行修改，建议存放到opt目录下
smc="/opt/tools/bin/smc"

# 记录电池电量
# 存储电池电量的文件位置，因设置限制充电必须root权限，故直接写到root目录下了
oldCountPath="/var/root/.battery.raw"

# 为配合开机自启设置，如果能获取到家目录就存放在家目录下
if [ "$HOME" != "" ];then
	oldCountPath="$HOME/.battery.raw"
fi

if [ ! -f $oldCountPath ]; then
	touch $oldCountPath
fi

# 历史电量
oldCount=0
# 读取历史电量
value="$(cat $oldCountPath)"
if [ "$value" = "" ]; then
	oldCount=0
else
	oldCount=`expr "$value"`
fi

# 记录充电状态
# 存储充电状态的文件位置，因设置限制充电必须root权限，故直接写到root目录下了
oldStatusPath="/var/root/.battery.status"

# 为配合开机自启设置，如果能获取到家目录就存放在家目录下
if [ "$HOME" != "" ];then
	oldStatusPath="$HOME/.battery.status"
fi

if [ ! -f $oldStatusPath ]; then
	touch $oldStatusPath
fi

# 历史状态
oldStatus=""
# 读取历史状态
value="$(cat $oldStatusPath)"
if [ "$value" != "" ]; then
	oldStatus=$value
fi

# 读取当前电池电量百分比，如果想确保显示的和限制的一致就用B0UC
# BRSC B0UC
h="$($smc -k B0UC -r)"
# 从输出信息里读取到电池电量百分比的值
r="${h##*\[ui16\]  }"
count=`expr ${r%%\ \(b*} / 16 / 16`

# 读取电池容量
value="$($smc -k B0RM -r)"
value="${value##*\[ui16\]  }"
passed=${value%%\ \(b*}

# 输出日志
function loger {
	# 将当前电量写入到文件
	echo $count > $oldCountPath
	# 将当前电池状态写入到文件
	echo $1 > $oldStatusPath
	if [ $count != $oldCount -o "$oldStatus" != "$1" ]; then
		echo "[$(date "+%Y-%m-%d %H:%M:%S")] 当前电量: $count%	容量：$passed mA/h	$1"
	fi
	exit 0
}

# 读取电源连接情况
# 电源是否连接
value="$($smc -k AC-W -r)"
if [[ $value =~ "ff)" ]]; then
	loger "直流供电"
	exit 0
fi

isJoin=true
value="$($smc -k CH0B -r)"
if [[ "$value" =~ "02)" ]]; then
	isJoin=false
fi

# 如果历史状态是"直流供电"
if [ "$oldStatus" = "直流供电" ]; then
	isJoin=true
fi

# 读取参数，第一个参数表示是否限制充电，第二个参数是最低电量百分比，第三个是最高电量百分比
if [ "$1" = "true" ]; then
	if [ "$2" = "" ]; then
		min=50
	else
		min=`expr $2`
	fi

	if [ "$3" = "" ]; then
		max=80
	else
		max=`expr $3`
	fi

	if [ $count -gt $max ]; then
		# 当前电池电量大于等于设置的就限制充电，如果当前已经限制了就不再设置了
		if [ $isJoin = true ]; then
			result=$($smc -k CH0B -w 02)
			if [ "$result" = "" ]; then
	   			loger "限制充电"
	   		else
	   			loger "$result"
	   		fi
		fi
	elif [ $count -le $min ]; then
		# 当前电池电量小于等于设置的就允许充电，如果当前已经允许了就不再设置了
		if [ $isJoin = false ]; then
			result=$($smc -k CH0B -w 00)
			if [ "$result" = "" ]; then
	   			loger "允许充电"
	   		else
	   			loger "$result"
	   		fi
	   	fi
	fi
fi

if [ $isJoin = true ]; then
	loger "交流充电"
fi

loger "交流供电"
```

​        上面的那个脚本是限制充电的一层封装，要跟随系统开机自启还需要一些配置。

​        开机启动的[详细说明](https://wild-flame.github.io/guides/docs/mac-os-x-setup-guide/preference_and_settings/launch)在这里就不说了，网上文章很多。直接附上plist文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>cn.tcoding.battery.helper</string>
	<key>ProgramArguments</key>
	<array>
		<string>/opt/tools/bin/blimit</string>
		<string>true</string>
		<string>50</string>
		<string>80</string>
	</array>
	<key>StartInterval</key>
	<integer>30</integer>
	<key>RunAtLoad</key>
	<true/>
	<key>StandardErrorPath</key>
	<string>/Users/username/.battery/error.log</string>
	<key>StandardOutPath</key>
	<string>/Users/username/.battery/out.log</string>
</dict>
</plist>
```

​        注意脚本的路径和label要和文件名一致，修改日志的路径，建议把这个自启配置放在`/Library/LaunchDaemons`文件夹。

> smc可执行文件和脚本放置在/opt/tools/bin文件夹下，如果不放这个文件夹自行修改

> 仓库里的plist文件需要修改用户名之后使用