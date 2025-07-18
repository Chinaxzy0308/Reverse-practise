# 基础
#打包 ：Python脚本不能在没有安装Python的机器上运行。将脚本打包成exe文件发送给别人，即使他的电脑上没有安装Python解释器，这个exe程序也能在上面运行
#解包 ：pyinstxtractor，下载：[extremecoders-re/pyinstxtractor: PyInstaller Extractor (github.com)](https://github.com/extremecoders-re/pyinstxtractor)
#逆向 ：然后就可以在解包的文件夹下找到python的主函数，是一个pyc文件，pyc文件时python编译后的文件，可以放到在线网站直接转换成py文件，即变为可读源码

[在线Python pyc文件编译与反编译 (lddgo.net)](https://www.lddgo.net/string/pyc-compile-decompile)

[python反编译 - 在线工具 (tool.lu)](https://tool.lu/pyc/)
#mafic_number :在反编译过程中，可能会遇到反编译不成功的情况，一般来说是因为出题人改了文件头，每个python版本的文件头是不一样的，手动修改文件头，即可继续将pyc反编译为py文件

[python Magic Number对照表以及pyc修复方法 - iPlayForSG - 博客园 (cnblogs.com)](https://www.cnblogs.com/Here-is-SG/p/15885799.html)