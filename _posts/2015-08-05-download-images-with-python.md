---
layout: post
title:  "down load images with python"
date:   2015-08-05 13:17:58
categories: jekyll update
---
用python批量下载一个网页中的图片，需要用到扩展库来解析html代码。有三种方法，一是用微软提供的扩展库win32com来操作IE，二是用selenium的webdriver，三是用python自带的HTMLParser解析。win32com可以获得类似js里面的document对象，不过是只读的（文档都没找到）。selenium则提供了Chrome，IE，FireFox等的支持，每种浏览器都有execute_script和find_element_by_xx方法，可以方便的执行js脚本（包括修改元素）和读取html里面的元素。不足是selenium只提供对python2.6和2.7的支持。HTMLParser则是需要自己写个类继承基类，重写解析元素的方法。个人感觉selenium用起来更方便，很容易操作html里的元素。

代码如下：

win32com：

{% highlight python%}
#将滚动条滑到底,最多滑动20000像素
#模拟键盘右键，查看多张图片
import sys
import win32com.client,win32api
import urllib.request
import time
import os

def main():
    #获取参数
    url=sys.argv[1]
    #操作IE
    ie=win32com.client.Dispatch("InternetExplorer.Application")
    ie.Navigate(url)
    ie.Visible=True
    last_url=''
    dir_name=''
    while last_url!=url:
        print('\nThe URL is:',url,'\n')
        while ie.ReadyState != 4:    
            time.sleep(1)
        while ie.Document.readyState != "complete": 
            time.sleep(1)
        #滑动滚动条
        win=ie.Document.parentWindow
        lastY=-1;
        for i in range(40):
            win.scrollTo(0,500*i)
            nowY=win.pageYOffset
            if(nowY==lastY):
                break
            lastY=nowY
            time.sleep(0.4)
        print('Document load state:',ie.Document.readyState)
        doc=ie.Document
        #第一次需要创建目录
        if(dir_name==''):
            root_dir='E:\\img'
            dir_name=root_dir+'\\'+doc.title
            dir_name=dir_name.replace('|','-')
            if(os.path.exists(root_dir)!=True):
                os.mkdir(root_dir)
            if(os.path.exists(dir_name)!=True):
                os.mkdir(dir_name)
        all_image=doc.images
        print('共有',all_image.length,'张图片')
        count=0;
        for img in all_image:
            if(img.id=='b_img'):
                count=count+1
                print(count,img.src)
                time.sleep(1)
                img_file=urllib.request.urlopen(img.src)
                byte=img_file.read()
                print(count,'donwload complete!','-'*10,'size:','{:.3}'.format(byte.__len__()/1024),'KB')
                if(byte.__len__()>7000):
                    file_name=img.src.replace('/','_')
                    file_name=file_name.replace(':','_')
                    end=file_name.__len__()
                    if(file_name.rfind('!')!=-1):
                        end=file_name.rfind('!')
                    if(file_name.rfind('?')!=-1):
                        end=file_name.rfind('?')
                    file_name=file_name[:end]
                    write_file=open(dir_name+'\\'+file_name,'wb')
                    write_file.write(byte)
                    write_file.close()
                    print(count,file_name,'complete!')
        #下一张
        last_url=url
        win32api.keybd_event(39,0)
        time.sleep(1)
        url=ie.Document.url
        print(last_url,url)
    #ie.Quit()
if __name__ == '__main__':
    main()
{% endhighlight %}
selenium:
{% highlight c++%}
# -*- coding: cp936 -*-
import sys
import urllib
import time
import os
from selenium import webdriver

def main():
    #获取参数
    url=sys.argv[1]
    #操作IE
    driver=webdriver.Chrome()
    driver.get(url)
    driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
    #创建目录
    dir_name=driver.find_element_by_tag_name('title').text
    print dir_name
    root_dir='E:\\img'
    dir_name=root_dir+'\\'+dir_name
    dir_name=dir_name.replace('|','-')
    if(os.path.exists(root_dir)!=True):
        os.mkdir(root_dir)
    if(os.path.exists(dir_name)!=True):
        os.mkdir(dir_name)
    images=driver.find_elements_by_tag_name('img')
    count=0
    for image in images:
        count=count+1
        image_url=str(image.get_attribute('src'))
        img_file=urllib.urlopen(image_url)
        byte=img_file.read()
        print count,'donwload complete!','-'*10,'size:',byte.__len__()/1024,'KB'
        if(byte.__len__()>7000):
            file_name=image_url.replace('/','_')
            file_name=file_name.replace(':','_')
            end=file_name.__len__()
            if(file_name.rfind('!')!=-1):
                end=file_name.rfind('!')
            if(file_name.rfind('?')!=-1):
                end=file_name.rfind('?')
            file_name=file_name[:end]
            write_file=open(dir_name+'\\'+file_name,'wb')
            write_file.write(byte)
            write_file.close()
            print count,file_name,'complete!'

    driver.quit()
if __name__ == '__main__':
    main()
{% endhighlight %}

HTMLParser:
{% highlight python%}
# import modules used here -- sys is a very standard one
import sys
import urllib.request
# Gather our code in a main() function

from html.parser import HTMLParser
class MyHTMLParser(HTMLParser):
    def handle_starttag(self,tag,attrs):
        if(tag=='img'):
            for attr in attrs:
                if(attr[0]=='src'):
                    img_file=urllib.request.urlopen(attr[1])
                    byte=img_file.read()
                    #文件大于1000b则生成文件,添加计数，下载多少图片，显示html代码
                    if(byte.__len__()>1000):
                        file_name=attr[1].replace('/','_')
                        file_name=file_name.replace(':','_')
                        end=file_name.__len__()
                        if(file_name.rfind('!')!=-1):
                            end=file_name.rfind('!')
                        if(file_name.rfind('?')!=-1):
                            end=file_name.rfind('?')
                        file_name=file_name[:end]
##                        print(file_name)
                        write_file=open('E:\\img\\'+file_name,'wb')
                        write_file.write(byte)
                        write_file.close()

def main():
    #获取参数
    url=sys.argv[1]
    print('\nThe URL is:',url,'\n')
    #读取url所指向的资源
    html_file=urllib.request.urlopen(url)
    byte_content=html_file.read()
    #将html网页保存起来
    url_file=open('E:\\img\\html\\result.htm','wb')
    url_file.write(byte_content)
    url_file.close()
    #从字节转换为字符串
    s=str(byte_content, encoding = "utf-8")
    #print(s)
    #bytes.decode(html_file.read())
    parser=MyHTMLParser(strict=False)
    parser.feed(s)
# Standard boilerplate to call the main() function to begin
# the program.
if __name__ == '__main__':
    main()
{% endhighlight %}
