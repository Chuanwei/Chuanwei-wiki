---
title: Nmap扫描结果xml处理
date: 2019-10-06 16:09:11
updated: 2019-10-06 16:09:11
categories:
- 渗透测试
- 信息收集
tags: [nmap,csv]
---

因需要批量对域名的端口进行扫描后进一步渗透，其他扫描工具域名和ip无法对应，测试起来比较麻烦，因此写了个工具将nmap的xml扫结果处理为csv，csv结果文件字段有：主机名,ip,端口,状态,协议,服务,版本,操作系统类型,其他信息。扫描结果清晰易于管理，方便进行下一步渗透测试。
<!--more-->

### 代码如下：

```
#!/usr/bin/python3
# -*- coding: utf-8 -*-
"""
@author:Chuanwei
@file:nmap_to_csv.py
@time:2019/10/06
"""
"""
使用方法：
nmap扫描输出xml文件：nmap -sS -O -sV -iL test.txt  -v -T4 -Pn -oX test.xml
单个：python nmap_to_csv.py test.xml
批量：python nmap_to_csv.py 1.xml 2.xml 3.xml
处理结果输出为csv文件，名称和源文件名称一样。
"""
import xml.etree.ElementTree as ET
import sys

def usage():
    print ("使用方法: %s 1.xml 2.xml 3.xml ...... " % __file__)
def parseNmap(filename,out_filename):
    try:
        tree=ET.parse(filename)
        root=tree.getroot()
    except Exception as e:

        print (e)
        return {}
    with open(out_filename,"w") as f:
        f.write("主机名,ip,端口,状态,协议,服务,版本,操作系统类型,其他信息\n")
        for host in root.iter('host'):
            if host.find('status').get('state') == 'down':
                continue
            ip=host.find('address').get('addr',None)
            hostname = host.find('hostnames').find('hostname').get('name',None)
            if not ip and not hostname:
                continue
            if host.find('ports').find('port') == None:
                output = hostname + "," + ip + ",,,,,,," + "\n"
                f.write(output)
            else:    
                for ports in host.iter('port'): 
                    port = ports.get('portid','')
                    status = ports.find('state').get('state','')
                    protocol = ports.get('protocol','') 
                    service = ports.find('service').get('name','')
                    product = ports.find('service').get('product','') 
                    version = ports.find('service').get('version','')
                    ostype = ports.find('service').get('ostype','')
                    extrainfo = ports.find('service').get('extrainfo','')
                    output = hostname + "," + ip + "," + port + "," + status + "," +  protocol + "," + service + ","+ version + ","+ ostype + ","+ extrainfo + "\n"
                    f.write(output)
    print(out_filename + "文件已生成！")
def main(args):
    for xml_file in args[1:]:
        print("处理：" + xml_file)
        out_filename = xml_file.strip(".xml")+ ".csv"
        parseNmap(xml_file,out_filename)
if __name__ == "__main__":
    if len(sys.argv) < 2:
        sys.exit(usage())
    else:
        main(sys.argv)
        
```
### 使用方法



```
nmap扫描输出xml文件：nmap -sS -O -sV -iL test.txt  -v -T4 -Pn -oX test.xml
单个：python nmap_to_csv.py test.xml
批量：python nmap_to_csv.py 1.xml 2.xml 3.xml
处理结果输出为csv文件，名称和源文件名称一样。
```
