# 语法
- [linux shell:[1] ()、(())、[]、[[]]、{}使用方法](https://www.jianshu.com/p/b2630cb10b4a)
- [bash中的空格](https://www.jianshu.com/p/1bb27cb7cae5)


# 看的书
- Advanced-Bash-Scripting-Guide-in-Chinese：这个还算不错，讲 shell 命令的一些原理，让你知道为什么会出现这样的情况。但有的地方像列表说明，比较枯燥。



# 经验
## 1, 双引号会保留变量中的空格
例如：
echo `ls -l`：会把输出的内容，按空格切分。切分后的内容，按行输出。
echo "`ls -l`"：会按 ls -l 命令的原样输出。

## 2, 双引号还会影响比较

### 例子1：字符串直接比较
#### 不带双引号
(1) if [[ abc == ab? ]]; then echo "equal"; else echo "not equal"; fi
结果： equal

(2) if [[ abc == ab ]]; then echo "equal"; else echo "not equal"; fi
结果： not equal

#### 带双引号
(1) if [[ "abc" == "ab?" ]]; then echo "equal"; else echo "not equal"; fi
结果： not equal

(2) if [[ "abc" == "ab" ]]; then echo "equal"; else echo "not equal"; fi
结果： not equal


### 例子2：变量比较

前提设置：var1=abc; var2=ab;

#### 不带双引号
(1) if [[ $var1 == $var2? ]]; then echo "equal"; else echo "not equal"; fi
结果： equal

(2) if [[ $var1 == $var2 ]]; then echo "equal"; else echo "not equal"; fi
结果： not equal

#### 带双引号
(1) if [[ "$var1" == "$var2?" ]]; then echo "equal"; else echo "not equal"; fi
结果： not equal

(2) if [[ "$var1" == "$var2" ]]; then echo "equal"; else echo "not equal"; fi
结果： not equal


### 结论：
可见，带上双引号后，就是原样比较。不带双引号，是 [[]] 内模式比较，就像正则比较。



## 3，{0..3}，{a..z} 也可以生产序列使用

## 4， cd -~ 回到上个目录

## 5，循环 或 if 写到一行时，需要加 ； 号，什么时候加呢？当出现关键字时，在关键字前面加。


