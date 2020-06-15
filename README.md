# python-learning
一些不怎么牛逼的python文件
# -*- coding:utf-8 -*-
# @Time :2020/6/10 22:42
# @Author :welgenh
# @File : 豆瓣爬虫.py

from  bs4 import BeautifulSoup           #网页解析，获取数据
import re                                #正则表达式进行文字匹配
import urllib.request,urllib.error       #制定URL，获取网页数据
import xlwt                              #进行excel解析
import sqlite3                           #进行SQLite 数据库操作
import time
def main():
    baseurl='https://movie.douban.com/top250?start='
    #1.爬取网页

    #2.解析数据
    #3.保存数据
    datalist = getData(baseurl)
    # savepath=r'豆瓣电影头50.xls'
    # saveData(datalist,savepath)

    saveData2(datalist)
def askURL(url):
    head={'User-Agent':' Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.97 Mobile Safari/537.36'}
    request=urllib.request.Request(url,headers=head)
    try:
        response=urllib.request.urlopen(request)
        html=response.read().decode('utf-8')
        # print(html)
    except urllib.error.URLError as e:         #识别特定的urllib错误
        if hasattr(e,'code'):
            print(e,'code')
        if hasattr(e,'reason'):
            print(e.reason)
    return html

#1.爬取网页
def getData(baseurl):
    datalist=[]
    #调用获取页面信息的函数10次
    findLink=re.compile(r'<a href="(.*?)">')
    findImgSrc=re.compile(r'<img.*src="(.*?)"',re.S)
    findtitle=re.compile(r'<span class="title">(.*)</span>')
    findRating=re.compile(r'<span class="rating_num" property="v:average">(\d\.\d)</span>')
    findJudge=re.compile(r'<span>(.*)人评价</span>')
    findInq=re.compile(r'<span class="inq">(.*?)</span>')
    findBd=re.compile(r'<p class="">(.*?)</p>',re.S)  #多行文本忽视换行符
    for i in range(10):
        url=baseurl+str(i*25)
        #根据得到的浏览器返回的信息用html接收，创建BeautifulSoup对象用'html.parser'解析器解析，
        #再调用bs4创建的对象soup功能find_all（非正则的findall）找到符合的项目并遍历成单个
        #根据正则匹配单个项目里对应符合的内容（一定要复制Edit as httml上再改动）
        #再根据所得内容用replace， re.sub替代，.strip()筛选出所需内容
        html=askURL(url)                        #保存获取到的网页信息
        soup=BeautifulSoup(html,'html.parser')  #html是解析对象，解析器是'html.parser'
        for item in soup.find_all('div',class_="item"):  #参数一和参数二同时满足的项目
            data=[]
            item=str(item)
            #影片详情的规则
            Link=(re.findall(findLink,item))[0]
            ImgSrc=(re.findall(findImgSrc,item))[0]
            title=re.findall(findtitle,item)
            if len(title)>1:
                title2=title[1].replace('/', '')
            else:
                title2 =''#利于输出到excel保存
            title=title[0]
            Ratin=(re.findall(findRating,item))[0]
            Judge=(re.findall(findJudge,item))[0]
            Inq=re.findall(findInq,item)
            if len(Inq)!=0:
                Inq=Inq[0].replace('。','')
            else:
                Inq=''
            Bd = re.findall(findBd, item)[0]
            Bd = re.sub(r"<br(\s+)?/>(\S+)?"," ",Bd)
            Bd = re.sub('/','',Bd)
            data=[Link,ImgSrc,title,title2,Ratin,Judge,Inq,Bd.strip()]
            print(data[0])
            datalist.append(data)
        time.sleep(1)
    return datalist

def saveData(datalist,savepath):
    print('save...')
    workbook = xlwt.Workbook(encoding='utf-8',style_compression=0)  # 创建workbook对象 style_compression=0样式的压缩效果
    worksheet = workbook.add_sheet('豆瓣电影Top250',cell_overwrite_ok=True)  #cell_overwrite_ok 单元格默认Trun为新内容覆盖旧内容
    col=('电影详情链接','图片链接','影片名','影片英文名','评分','评价数','概括','相关信息')
    for i in range(250):
        if i<8:
            worksheet.write(0, i*2, col[i])                   #每隔一行空一行
        print('第%d条'%i)
        data=datalist[i]
        for j in range(8):                                    # 后面越来越多
            worksheet.write(i+1, j*2, '%s' % data[j])         #行列数-1=下标,0,2,4,6,8

    workbook.save(savepath)
    print('保存成功')


def saveData2(datalist):
    # dbpath = 'D:\Users\Administrator\AppData\Local\Programs\Python\Python37\python.exe "D:/Program Files/JetBrains/something/进阶/sql/sqlite3基础/'
    conn=built_data('movieTop250.db')
    cur=conn.cursor()

    for data in datalist:
        for index in range(8):
            if index==(5 or 6):
                continue
            else:
                data[index] = '"' + str(data[index]) + '"'
        sql_data='''
                insert into movieTop250(Link,ImgSrc,title,title2,Ratin,Judge,Inq,Bd)
                values (%s)'''%",".join(data)  #将data列表以%+，的形式传入到%s
        cur.execute(sql_data)
        print(sql_data)
        conn.commit()
    conn.commit()
    conn.close()
    print('ok')
def built_data(dbpath):
    #[Link,ImgSrc,title,title2,Ratin,Judge,Inq,Bd.strip()]
    #id 顺序，整合为自增长标题
    conn = sqlite3.connect(dbpath)
    sql='''
        create table movieTop250
        (id integer primary key autoincrement,
        Link text,
        ImgSrc text,
        title varchar,
        title2 varchar,
        Ratin numeric,
        Judge numeric,
        Inq text,
        Bd text)
    '''
    cursor = conn.cursor()
    cursor.execute(sql)
    return conn

if __name__ == '__main__':

    main()




