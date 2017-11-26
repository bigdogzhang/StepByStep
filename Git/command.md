# Git常用命令汇总

从远程服务器克隆Repo：git clone https://......

查看当前branch发生了哪些改变，如哪些文件被追加，哪些文件被修改等信息：git status  

添加某个或某些文件：git add file1 file2 file3

添加某文件夹下所有的文件，包括该文件夹在内：git add directory/*  

添加当前目录下所有的文件：git add *  

添加当前目录下所有的txt文件：git add *.txt

提交：

git commit file1 file2 file3 -m "message"

git commit * -m "message"

Pull:  git pull

Push:  git push

