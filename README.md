# -
#知音漫客爬虫
# -*- coding: utf-8 -*-

from urllib import request, parse
from selenium import webdriver
import re
import sys
import os
#import importlib

#importlib.reload(sys)
non_bmp_map = dict.fromkeys(range(0x10000, sys.maxunicode + 1), 0xfffd)#打印时过滤掉的会出错的字符-表
class ErrorCode(Exception):#自定义错误
    '''自定义错误码:
        1: 章节格式或数值不正确
        2: 中断下载'''

    def __init__(self, code):
        self.code = code

    def __str__(self):
        return repr(self.code)
#第一部分提取目录网页源代码：
surl = input('请输入漫画目录URL链接：')#漫画目录URL
def uget_html(iurl):
    headers = {
        'User-Agent': '''Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.87 Safari/537.36 QIHU 360SE'''
        }
    req = request.Request(url=iurl, headers=headers)
    url_open = request.urlopen(req)
    webdaima = url_open.read()
    html_text = webdaima.decode('utf-8')
    #print(html_text.translate(non_bmp_map))#打印网页源代码（自动过滤）
    return html_text

def sget_html(iurl):
    browser = webdriver.PhantomJS()
    browser.get(iurl)
    page_text = browser.page_source
    browser.quit()
    return page_text

def saveimg(img_url, name, ipath):
    if not os.path.isdir(ipath):#若没有该文件夹则自动创建
        n = 0
        mpath = []
        while not os.path.isdir(ipath):
            ipath1 = os.path.split(ipath)
            ipath = ipath1[0]
            mpath.insert(0, ipath1[1])
            n = n + 1
        for x in range(n):
            ipath = os.path.join(ipath, mpath[x])
            os.mkdir(ipath)
    epath = os.path.join(ipath, name)
    if os.path.isfile(epath):
        print('%s已存在，自动跳过！建议检查该图片是否已下载完整！'%epath)
        return None
    request.urlretrieve(img_url, filename=epath)

#第二部分分析出每章节URL：
zhze_index = re.compile('<li class="item" data-id="\d{1,6}"><a href="(\d{1,6}\.html)" title="(\S+)">\S+<\/a>')
zhze_comic_name = re.compile('<div class="title-warper"><h1 class="title" data-comic_id="\d+">(\S+)<\/h1>')
comic_index_html = uget_html(surl)
comic_index_list = zhze_index.findall(comic_index_html)
comic_name = zhze_comic_name.findall(comic_index_html)[0]
comic_index_html = ''#清空储存的网页代码
print('''---------------- 漫画《%s》章节：-----------------
序号| 章节 |               URL'''%comic_name)
comic_chapter = []
comic_chapter_num = len(comic_index_list)
num = comic_chapter_num
for x in comic_index_list:#打印漫画的章节-网页URL
    print('---------------------------------------------')
    comic_chapter_name = x[1]
    comic_chapter_url = surl+x[0]
    #方案B：(组成新的list-tuple，获得完整URL)
    print(' %s|  %s  |%s' %(num,comic_chapter_name,comic_chapter_url))
    chapter_tuple = (comic_chapter_name, comic_chapter_url)
    comic_chapter.insert(0, chapter_tuple)#从1话-末尾话顺序
    num = num - 1
    #方案A：（创建dict储存 章节-URL）(缺点：无序，章节选择困难)
    #comic_chapter = {}
    #comic_chapter[comic_chapter_name] = comic_chapter_url
    #print(comic_chapter)

#第三部分选择章节：
def zjxz(mstr):#章节选择
    if re.match('\d{1,4}-\d{1,4}', mstr):#多章
        m = mstr.split('-')
        m2 = int(m[1])
        m1 = int(m[0]) - 1
        if not m1 < m2:
            print('【错误】请输入正确的序号数值！ （序号1应小于序号2）')
            raise ErrorCode(1)
    elif re.match('\d{1,4}', mstr):#单章
        m1 = int(mstr) - 1
        m2 = m1 + 1
    elif mstr == 'all':
        m1 = 0
        m2 = comic_chapter_num
    else:
        print('【错误】请输入正确的序号格式! 例子：输入“1-22”或“23”')
        raise ErrorCode(1)
    if m2 <= comic_chapter_num:
        return range(m1, m2)
    else:
        print('【错误】该漫画没有%s章节！最多只有%s章！'%(m2, comic_chapter_num))
        raise ErrorCode(1)

print('''【章节选择】： 请输入要下载的章节序号.
【格式1-多章下载】:(序号1)-(序号2)  如：1-22（无空格|减号：- |序号1<序号2）
【格式2-单章下载】：(序号)  如：23（无空格）
【下载全部】：请输入all''')
xuanze = input()
#第四部分下载所选章节：
def pageurl2hd_list(firstpage_url, page_num_min, page_num_max):
    pages_hd_list = []
    #方案A用正则表达式替换页码和清晰度标识：（放弃）（缺点：复杂）
    #zhze_page_url = re.compile('(http:\/\/\S+\/\S+\/\S+\/\S+\/\S+\/)\d+\.jpg-zymk.middle')
    #方案B切割字符串：
    n1 = page_num_min
    n2 = page_num_max + 1
    p1 = firstpage_url.split('1.jpg-zymk.middle')[0]
    for pnum in range(n1, n2):
        p2 = '%s%s.jpg-zymk.high'%(p1,pnum)
        pages_hd_list.append(p2)
    return pages_hd_list

#正则表达式放循环外
zhze_page = re.compile('<img src="(http:\/\/\S+\.jpg-zymk.middle)" class="comicimg".+<p class="page-info">第 (\d{1,3}) 页\/共 (\d{1,3}) 页<\/p>')
for y in zjxz(xuanze):
    chapter_url = comic_chapter[y][1]
    chapter_html = sget_html(chapter_url)
    #print(chapter_html)#打印网页源代码
    page_info = zhze_page.findall(chapter_html)
    #print(page_info)
    page_url = page_info[0][0]
    page_num_min = int(page_info[0][1])
    page_num_max = int(page_info[0][2])
    comic_chapter_now = comic_chapter[y][0]
    page_num_now = page_num_min
    print('漫画《%s》章节：<%s> |共%s页|'%(comic_name,comic_chapter_now, page_num_max))
    save_path = 'd:\\mh\\漫画\\%s\\%s'%(comic_name,comic_chapter_now)
    for dlpg in pageurl2hd_list(page_url, page_num_min, page_num_max):
        save_name = '%s.jpg'% page_num_now
        saveimg(dlpg, save_name, save_path)
        print('''第%s页下载完成  |  共%s页\n(URL:%s)'''%(page_num_now, page_num_max, dlpg))
        page_num_now = page_num_now + 1
        all_page_num = all_page_num + 1
        if all_page_num == 1580:
            print('---累计下载了1580页漫画，休息一下---')
            time.sleep(5)
    print('漫画《%s》章节:<%s>|共%s页|下载完成！'%(comic_name,comic_chapter_now,page_num_now-1))
print('·已将所选章节保存至%s\n·任务完成！程序自动退出！'%save_path.split(comic_chapter_now)[0])
