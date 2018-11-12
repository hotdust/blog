# Hello World
## Bash
Language 选择 bash，然后输入以下代码：
```bash
cat << EOB
<items>
    <item autocomplete = "autocompletex" uid = "123321" arg = "argsx" >
    <title >hello</title>
    <subtitle >world</subtitle>
    <icon >icon</icon>
    </item>
</items>
EOB
```

## Python
Language 选择 python，然后输入以下代码：
```
from datetime import datetime
import sys

# get args
t = sys.argv[1]

# convert to iso time
iso_date = datetime.fromtimestamp(long(t))

# output
output = """
<items>
    <item autocomplete = "autocompletex" uid = "123321" arg = "argsx" >
    <title >{}</title>
    <subtitle >`date -d @1541668140 "+%Y-%m-%d %H:%M:%S"`</subtitle>
    <icon >icon</icon>
    </item>
</items>
""".format(iso_date)

print(output)

```



python 接收参数
方法1:
- 选择`with input as argv`
- `t = sys.argv[1]`

方法2:
- 选择`with input as query`
- `t = sys.argv[1]`

# 其它
## 1，Bundle Id
每个 workflow 要有一个唯一的 bundle id。如果两个 wrokflow 有同一个 bundle id 的话，在导入时，后一个导入的会替换前一个。