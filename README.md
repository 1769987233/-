
```python
import requests
from lxml import etree
from urllib import parse
import re
import json
import threading
from queue import Queue
import hashlib
 
 
class Spider(object):
    # 封装属性
    def __init__(self):
        self.big_set = set()
        self.conut = 2
        self.domain = "https://music.163.com"
        # 这个cookie会过期，使用requests.session()
        self.cookies = "_ntes_nnid=2a370ddafbaa8c4a4918f335705a78f9,1576592837529; _iuqxldmzr_=32; WM_TID=oK8Bh118%2BE9FFQEFRFYo%2Bk2URh7F9sz3; MUSIC_FU=50760ce67efbcac315aaaeb73c474fc759a861d304e62c6b71f00e0133fb4d09; ntes_kaola_ad=1; WM_NI=mGol9ziQhFk5o2By%2FKYkXylqOvs2ogKPPx9JsfEMB0v6mAr1V4MafqejoO30tUC4G6m6JbXoC%2FabSzq9pYscHfhOZPJ4IZBJBBvL%2Beez5KSNGy3wmNxkDExsenURdhgcanE%3D; WM_NIKE=9ca17ae2e6ffcda170e2e6eed2c23a8f88a285bc72abac8bb2c55e939a8eaab77bab8ae596b66ef4a8a9d6e82af0fea7c3b92aaf99a18dbc72ab8fa8a7ea3fadee8e9aae419bb5989aef7b8daabdd3ee6d8990a099bc42b1ba8baec13cb6eaa3a2f13b96ec8382e57ba7b6a1b0b23cb88b8dd5c63eafea8683db6581f1af83d272b3e7c0baf2698cbaa8b0d95c8aec87bbc142829ebaadaa6fa7a6fbacd24685a7fd98f046bcb58ca6fb79a7b29bb3e764a5bdaf8fc837e2a3; JSESSIONID-WYYY=CTBkJVhsHud8lUA96HIGCd7zTEct9vpBt2tA%2Fny%5C%2FQE5hhUrEaVDig6PQ7bOJ%5C8ubN9XqkNsiWPs8r1viXclodB7tvP%2BFPKWvM1Dg4%2F%2Fux%2F5JnekJB2jNS7p0BceQWsCaBNqRcuWUzy0p%2BhZHCft2hq0%2FhIIBTFHTn9EutKWN049vfzF%3A1577467835043"
        # 注意不能出现url中不能出现"#"
        self.start_url = "https://music.163.com/discover/playlist/"
        self.headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.88 Safari/537.36",
        }
        self.sings_url_queue = Queue()  # 歌单url
        self.sings_html_queue = Queue()  # 歌单响应
        self.every_sing_url_queue = Queue()  # 每一个歌单的歌曲的url
 
    # 获取cookie字典形式
    def get_cookies_self(self):
        cookies = {i.split("=")[0]: i.split("=")[1] for i in self.cookies.split(";")}
        return cookies
 
    # 发送请求获取响应
    def get_html(self, url):
        cookies = self.get_cookies_self()
        response = requests.get(url, headers=self.headers, cookies=cookies)
        return response.content.decode()
 
    # 获取全部歌单的url（歌单页）
    def parse(self, url):
        response = self.get_html(url)
        html_elm = etree.HTML(text=response)
        # 获取歌单的url
        url_list = html_elm.xpath("//ul[@id='m-pl-container']/li/div/a[1]/@href")
        # ============第1步=============
        self.sings_url_queue.put(url_list)
        #下一页的歌单url地址
        next_url = html_elm.xpath("//a[text()='下一页']/@href")[0]
        while next_url != "javascript:void(0)":
            next_url = parse.unquote(next_url)
            next_url = parse.urljoin(self.domain, next_url)  # 下一页的歌单的完整url地址
            print("第{}页完整歌单的url".format(self.conut), next_url)
            self.conut += 1
            self.parse(next_url)  # 递归调用
        #url_list.extend(next_url_list)  # 添加到大列表
        #self.sings_url_queue.put(url_list)
 
    # 每一个歌单url的响应(点进歌单页看到歌列表)
    def get_sings_html(self):
        # ============第2步=============
        while 1:
            url_list = self.sings_url_queue.get()
            #print("获得歌单的url", url_list)
            for i in url_list:
                url = parse.urljoin(self.domain, i)
                print("拼接的歌单url", url)
                cookies = self.get_cookies_self()
                response = requests.get(url, headers=self.headers, cookies=cookies)
                #print(response.status_code)
                # ============第3步=============
                self.sings_html_queue.put(response.content.decode())
 
    # 获取每一首歌曲的url
    def get_sing_list(self):
        # ============第4步=============
        while 1:
            response = self.sings_html_queue.get()
            html_elm = etree.HTML(text=response)
            sing_list = html_elm.xpath("//div[@id='song-list-pre-cache']/ul/li/a/@href")
            #print("每一个歌单的歌曲的url:", sing_list)
            # ============第5步=============
            self.every_sing_url_queue.put(sing_list)
 
    # 发送请求获取每一首歌曲响应+数据
    def get_sing_html(self):
        # ============第6步=============
        while 1:
            url = self.every_sing_url_queue.get()
            for i in url:
                every_sing_list = []
                url_num = re.findall(r"\d+", i)
                # 获取歌名+歌手
                url_name = "https://music.163.com/song?id=" + url_num[0]
                response1 = self.get_html(url_name)
                html_elm1 = etree.HTML(text=response1)
                sing_name = html_elm1.xpath("//div[@class='cnt']//em[@class='f-ff2']/text()")[0] if len(
                    html_elm1.xpath("//div[@class='cnt']//em[@class='f-ff2']/text()")) > 0 else None
                sing_singer = html_elm1.xpath("//div[@class='cnt']//span/@title")[0] if len(html_elm1.xpath(
                    "//div[@class='cnt']//span/@title")) > 0 else None
                # print(sing_name)
                # 获取评论+点赞数
                url = "https://music.163.com/weapi/v1/resource/comments/R_SO_4_" + url_num[0] + "?csrf_token="
                cookies = cookies = self.get_cookies_self()
                 
                data = {
                    "params": "/SUbxIEON9B0tm/OT7p89/1dZ2wkhK+jogYGnLdYY1BWxdfpFo7YgxUBVxoCuh6P92GpQDRBu4EF0frSd1JG2hmTex36G2Qw77CC/6s3fa3facaX8A3CUpyUUNoK8h3fN8hIZwwrQuHFVuwXxeKeoA==",
                    "encSecKey": "deba713b7bc36c398c8d9c99fa4f11a33b4beba35131db3df66188cca4bae0c8a7c4e390aeaacc1fe0b9d44baedc9c6289026e5fe2d8082a9ffab2e3eec34e5f8b2c53845ad593fdd9572fd9618a510461b05f4c49a169b3095dea055d40e365acae25313e044f3a28b341e7697f0222da29d3104ec76c0370eaffb3577e4b1b"
                }
                response_sing = requests.post(url, headers=self.headers, cookies=cookies, data=data)
                comment_dict = json.loads(response_sing.content.decode())
                try:
                    comment = comment_dict["hotComments"][0]["content"]
                    comment_num = comment_dict["hotComments"][0]['likedCount']
                except:
                    comment = "空"
                    comment_num = "0"
                if int(comment_num) > 10000:
                    every_sing_list.append(comment_num)
                    every_sing_list.append(sing_name)
                    every_sing_list.append(sing_singer)
                    every_sing_list.append(comment)
                    #print(every_sing_list)
                    m_obj = hashlib.md5()
                    m_obj.update((every_sing_list[1]+every_sing_list[2]+every_sing_list[3]).encode())
                    ret = m_obj.hexdigest()
                    #print("摘要：", ret)
                    self.big_set.update(ret)
                    if ret not in self.big_set:
                        self.save_data(every_sing_list)
                    else:
                        print("数据重复")
 
 
    # 保存数据
    def save_data(self, list_data):
        data1 , data2, data3 = self.execute_str(list_data)
        with open("music.txt", "a", encoding="utf8") as f:
            f.write(str(list_data[0]) + "  ")  # 纯数字好排序
            f.write("歌名：" + data1 + "    ")  # 歌名歌手去空格使用"_".join(str.split(" "))
            f.write("歌手：" + data2 + "    ")
            f.write("评论：" + data3 + "\n")
        print("写入成功！")
 
    # 处理字符串
    def execute_str(self, list_data):
        str1 = list_data[1]
        data1 = "_".join(str1.split(" "))
        data1 = "_".join(data1.split(" "))
 
        str2 = list_data[2]
        data2 = "_".join(str2.split(" "))
        data2 = "_".join(data2.split(" "))
 
        str3 = list_data[3].replace("\r", "")
        str3_new = str3.replace(" ", "")
        data3 = str3_new.replace("\n", "")
 
        return data1, data2, data3
 
    # 实现主要逻辑
    def run(self):
        thread_list = []
        t1 = threading.Thread(target=self.parse, args=(self.start_url,))  # 注意这里传的是元组
        thread_list.append(t1)
 
        t2 = threading.Thread(target=self.get_sings_html)
        thread_list.append(t2)
 
        t3 = threading.Thread(target=self.get_sing_list)
        thread_list.append(t3)
 
        t4 = threading.Thread(target=self.get_sing_html)
        thread_list.append(t4)
        print(thread_list)
        for t in thread_list:
            t.start()
        for t in thread_list:
            t.join()
 
 
if __name__ == "__main__":
    print("start...")
    s = Spider()
    s.run()
    print("...done")
```

