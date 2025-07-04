# modian-spider
#对摩点网页的数据爬取

# -*- coding: UTF-8 -*-
import json
import random
import re
import time
import hashlib
import concurrent.futures
from typing import List, Dict
import queue
import threading
import re

import pandas
from bs4 import BeautifulSoup
import requests
from lxml import html
# 全局代理池和锁
proxy_pool = []
proxy_lock = threading.Lock()

# Initialize the results list
d = []  # 初始化结果列表
import re


def count_imgs(html_content):
    tree = html.fromstring(html_content)
    # 统计projectDetail区域img数量
    count1 = len(tree.xpath('//div[@class="project-content" and @id="projectDetail"]//img'))
    # 统计所有id为projectContentTemplate的script标签内img数量，取最大
    script_tags = tree.xpath('//script[@id="projectContentTemplate"]')
    max_count2 = 0
    for script_tag in script_tags:
        script_html = script_tag.text
        if script_html:
            count2 = len(re.findall(r'<img [^>]*>', script_html))
            if count2 > max_count2:
                max_count2 = count2
    max_count = max(count1, max_count2)
    return max_count if max_count > 0 else '未找到'
def get_image_category(p_count):
    if isinstance(p_count, str):
        return '未找到'
    if 0 <= p_count <= 6:
        return 1
    elif 6 < p_count <= 12:
        return 2
    elif 12 < p_count <= 18:
        return 3
    elif 18 < p_count <= 24:
        return 4
    else:  # 24以上
        return 5
def generate_project_info_sign(timestamp, post_id):
    # Construct the string to hash
    base_str = f"apim.modian.com/apis/mdcomment/project_infoMzgxOTg3ZDMZTgxO{timestamp}post_id={post_id}d41d8cd98f00b204e9800998ecf8427e"
    # Create MD5 hash
    return hashlib.md5(base_str.encode()).hexdigest()


def comments_count_def(url):
    # https://zhongchou.modian.com/item/27167.html
    post_id = url.split('item/')[1].replace('.html', '')
    current_timestamp = str(int(time.time()))

    headers = {
        "accept": "application/json, text/plain, */*",
        "accept-language": "zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6",
        "cache-control": "no-cache",
        "channel;": "",
        "client": "4",
        "dnt": "1",
        "mt": current_timestamp,
        "origin": "https://www.modian.com",
        "referer": "https://www.modian.com/",
        "sign": generate_project_info_sign(current_timestamp, post_id),
        "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36 Edg/136.0.0.0",

    }
    url = "https://apim.modian.com/apis/mdcomment/project_info"
    params = {
        "post_id": f"{post_id}"
    }
    response = make_request('GET', url, headers=headers, params=params)
    try:
        comments_count = response.json()["data"]["comment_count"]
        return comments_count
    except:
        return '未找到'


def fetch_proxies():
    """从API获取新代理并更新到池中"""
    global proxy_pool
    for _ in range(10):
        try:
            response = requests.get(
                "http://api.91http.com/v1/get-ip?trade_no=A841177555688&secret=1GZbjbLKKKRqCCGv&num=10&protocol=1&format=text&sep=2&auto_white=1",
                timeout=120)
            if response.status_code == 200:
                # 检查是否有频率异常提示
                if any(keyword in response.text for keyword in ["频率异常", "请求频率", "too many requests", "rate limit"]):
                    print("检测到接口请求频率异常，暂停2分钟后重试...")
                    time.sleep(120)
                    continue  # 重新请求
                proxies = response.text.strip().split('\n')
                new_pool = [{
                    'http': f'http://{proxy}',
                    'https': f'http://{proxy}'
                } for proxy in proxies]
                with proxy_lock:
                    proxy_pool = new_pool
                print(f"成功获取 {len(new_pool)} 个新代理")
                break
            else:
                print("获取代理失败")
                break
        except Exception as e:
            # 检查异常信息中是否有频率异常
            if any(keyword in str(e) for keyword in ["频率异常", "请求频率", "too many requests", "rate limit"]):
                print("检测到接口请求频率异常（异常信息），暂停2分钟后重试...")
                time.sleep(120)
                continue  # 重新请求
            print(f"获取代理时出错: {e}")
            break


def get_proxy():
    with proxy_lock:
        if proxy_pool:
            return random.choice(proxy_pool)
        else:
            return None


headers = {
    "accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7",
    "accept-language": "zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6",
    "cache-control": "no-cache",
    "dnt": "1",
    "pragma": "no-cache",
    "priority": "u=0, i",
    "referer": "https://zhongchou.modian.com/?_mpos=h_nav_discover",
    "sec-ch-ua": "\"Chromium\";v=\"136\", \"Microsoft Edge\";v=\"136\", \"Not.A/Brand\";v=\"99\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Windows\"",
    "sec-fetch-dest": "document",
    "sec-fetch-mode": "navigate",
    "sec-fetch-site": "same-origin",
    "sec-fetch-user": "?1",
    "sec-gpc": "1",
    "upgrade-insecure-requests": "1",
    "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36 Edg/136.0.0.0"
}


def make_request(method, url, headers, data=None, json=None, params=None, retries=15, delay=2):
    attempt = 0
    while attempt < retries:
        try:
            # 使用全局代理
            proxy = get_proxy()

            if method == 'POST':
                if json is not None:
                    response = requests.post(url, json=json, headers=headers, timeout=60, proxies=proxy)
                else:
                    response = requests.post(url, data=data, headers=headers, timeout=60, proxies=proxy)
            elif method == 'GET':
                response = requests.get(url, headers=headers, params=params, timeout=60, proxies=proxy)
            else:
                raise ValueError("不支持的 HTTP 方法")

            response.raise_for_status()  # 检查请求是否成功
            return response

        except requests.exceptions.HTTPError as err:
            print(f"HTTP 错误发生: {err}")
        except requests.exceptions.ProxyError:
            print("代理错误，尝试新的代理")
            print("由于代理错误，正在更新代理池。")
            fetch_proxies()  # 代理错误时更新代理池
            continue
        except Exception as err:
            print(f"其他错误发生: {err}")

        attempt += 1
        print(f"重试中 ({attempt}/{retries})...")
        time.sleep(delay)  # 等待一段时间再重试

    print("重试次数已用尽，请求失败。")
    return None


def extract_value(html_tag):
    # 定义正则表达式模式，用于匹配 value 属性值
    pattern = r'<input type="hidden" name="creater_user_id" value="(\d+)"'
    # 使用 re.search 查找匹配项
    match = re.search(pattern, html_tag)
    # 如果找到匹配项，返回捕获组中的内容（即 value 属性值）
    if match:
        return match.group(1)
    # 未找到匹配项时返回 None
    return '未找到'


def generate_sign(timestamp, user_id):
    # Construct the string to hash
    base_str = f"apim.modian.com/apis/comm/user/user_infoMzgxOTg3ZDMZTgxO{timestamp}json_type=1&user_id={user_id}d41d8cd98f00b204e9800998ecf8427e"
    # Create MD5 hash
    return hashlib.md5(base_str.encode()).hexdigest()


def get_auth_type(auth_info):
    # 检查认证类型
    if auth_info == 2:
        return 2
    if not auth_info:
        return -1  # 未认证
    if "实名认证" in auth_info or "身份认证" in auth_info:
        return 0  # 个人认证
    if "公司名称" in auth_info:
        return 1  # 企业认证

    return -1  # 未知类型


def get_follower_score(follower_count):
    """根据粉丝数量返回分数"""
    if follower_count <= 150:
        return 1
    elif follower_count <= 300:
        return 2
    elif follower_count <= 450:
        return 3
    elif follower_count <= 600:
        return 4
    else:
        return 5


def get_experience_score(project_count):
    """根据项目数量返回经验分数"""
    if project_count <= 2:
        return 1
    elif project_count <= 4:
        return 2
    elif project_count <= 6:
        return 3
    elif project_count <= 8:
        return 4
    else:
        return 5


def get_user_scores(user_id):
    """
    获取用户的各项分数

    参数:
        user_id (str): 用户ID

    返回:
        dict: 包含fa(粉丝数分数)、na(资质认证)、ex(经验分数)的字典
    """
    # 获取当前时间戳
    current_timestamp = str(int(time.time()))

    headers = {
        "accept": "application/json, text/plain, */*",
        "accept-language": "zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6",
        "cache-control": "no-cache",
        "channel;": "",
        "client": "4",
        "dnt": "1",
        "mt": current_timestamp,
        "origin": "https://www.modian.com",
        "referer": "https://www.modian.com/",
        "sign": generate_sign(current_timestamp, user_id),
        "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36 Edg/136.0.0.0",

    }

    url = "https://apim.modian.com/apis/comm/user/user_info"
    params = {
        "json_type": "1",
        "user_id": user_id
    }

    try:
        response = make_request('GET', url, headers=headers, params=params)
        data = response.json()
        # print(data)

        # 获取粉丝数
        follower_count = data["data"]["follower_count"]
        # print(int(follower_count))
        fa = get_follower_score(follower_count)

        # 获取项目数量
        project_count = 0
        for card in data['data']['cards']:
            if card['type'] == 5:
                project_count = card['num']
                break
        # print(int(project_count))
        ex = get_experience_score(project_count)

        # 获取认证信息
        try:
            auth_info = data['data']['user']['auth_info']
        except:
            auth_info = 2
        auth_type = get_auth_type(auth_info)
        # print(auth_type)
        if auth_type in [0, 1, 2]:
            na = auth_type
        else:
            na = 0
        return {
            "fa": fa,  # 发起者粉丝数分数
            "na": na,  # 发起者资质
            "ex": ex  # 发起者经验分数
        }

    except Exception as e:
        print(f"获取用户信息时出错: {e}")
        return {
            "fa": '未知',
            "na": '未知',
            "ex": '未知',
        }


def classify_supporters(count):
    """支持者数量分类"""
    if isinstance(count, str):
        try:
            count = int(count)
        except:
            return "未找到"

    if 0 <= count < 30:
        return 1
    elif 30 <= count < 60:
        return 2
    elif 60 <= count < 90:
        return 3
    elif 90 <= count < 120:
        return 4
    else:  # 120以上
        return 5


def classify_comments(count):
    """评论数分类"""
    if isinstance(count, str):
        try:
            count = int(count)
        except:
            return "未找到"

    if 0 <= count < 25:
        return 1
    elif 25 <= count < 50:
        return 2
    elif 50 <= count < 75:
        return 3
    elif 75 <= count < 100:
        return 4
    else:  # 100以上
        return 5


def classify_donations(count):
    """无偿支持人数分类"""
    if isinstance(count, str):
        try:
            count = int(count)
        except:
            return "未找到"

    if 0 <= count < 2:
        return 1
    elif 2 <= count < 4:
        return 2
    elif 4 <= count < 6:
        return 3
    elif 6 <= count < 8:
        return 4
    else:  # 8以上
        return 5


def classify_updates(count):
    """更新数分类"""
    if isinstance(count, str):
        try:
            count = int(count)
        except:
            return "未找到"

    if 0 <= count < 1:
        return 1
    elif 1 <= count < 2:
        return 2
    elif 2 <= count < 3:
        return 3
    elif 3 <= count < 4:
        return 4
    else:  # 4以上
        return 5


def classify_tiers(count):
    """档位数分类"""
    if isinstance(count, str):
        try:
            count = int(count)
        except:
            return "未找到"

    if 0 <= count < 3:
        return 1
    elif 3 <= count < 6:
        return 2
    elif 6 <= count < 9:
        return 3
    elif 9 <= count < 12:
        return 4
    else:  # 12以上
        return 5


def classify_target_amount(amount):
    """目标金额分类"""
    if isinstance(amount, str):
        try:
            amount = int(amount)
        except:
            return "未找到"

    if 0 <= amount < 2500:
        return 1
    elif 2500 <= amount < 5000:
        return 2
    elif 5000 <= amount < 7500:
        return 3
    elif 7500 <= amount < 10000:
        return 4
    else:  # 10000以上
        return 5


def extract_project_info(url, item=None):
    # 发送HTTP请求获取网页内容
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
    }

    try:
        response = make_request('GET', url, headers=headers)
        response.raise_for_status()  # 检查请求是否成功
        response.encoding = 'UTF-8'
        content = response.text
    except requests.RequestException as e:
        print(f"请求失败: {e}")
        return

    # 创建lxml解析对象
    tree = html.fromstring(content)

    # 检查众筹是否成功
    success_status = 0
    status_elem = tree.xpath('//div[@class="tanchuzhuangtai"]/@status')
    if status_elem and status_elem[0] == '众筹成功':
        success_status = 1

    # 提取支持者数量
    supporters_count = "未找到"
    support_people = tree.xpath('//div[@class="col3 support-people"]/span/text()')
    if not support_people:
        support_people = tree.xpath('//span[@backer_count]/text()')
    if support_people:
        supporters_count = support_people[0]

    # 提取评论数
    comments_count = "未找到"
    # comments_elem = tree.xpath('//input[@name="order_comment_count"]/@value')
    # if comments_elem:
    #     comments_count = comments_elem[0]

    comments_count = comments_count_def(url)
    # 提取无偿支持人数
    donation_count = "未找到"
    donation_elem = tree.xpath('//em[@class="count" and @reward_list="-3"]/text()')
    if donation_elem:
        donation_count = donation_elem[0].strip().split(' ')[1]

    # 提取更新数
    updates_count = "未找到"
    updates_elem = tree.xpath('//li[@class="pro-gengxin"]/span/text()')
    if updates_elem:
        updates_count = updates_elem[0]

    # 提取目标金额
    target_money = "未找到"
    target_money_elem = tree.xpath('//span[@class="goal-money"]/text()')
    if target_money_elem:
        target_money_text = target_money_elem[0].strip()
        if '¥' in target_money_text:
            target_money_str = target_money_text.split('¥')[-1].replace(',', '')
            try:
                target_money = int(target_money_str)
            except ValueError:
                target_money = "转换失败"

    # 统计档位数
    tier_count = "未找到"
    tier_lists = tree.xpath('//div[@class="payback-lists margin36"]//div[contains(@class, "back-list")]')
    if tier_lists:
        tier_count = len(tier_lists)

    # 统计图片数量
    # image_category = "未找到"
    p_count = count_imgs(content)
    image_category = get_image_category(p_count)
    # 检查是否包含视频
    has_video = 0
    video_icon = tree.xpath('//img[@src="https://s.moimg.net/m/images/video-play.png"]')
    if video_icon:
        has_video = 1
    else:
        video_elem = tree.xpath('//p[@class="vedio_url"]/@data-duration')
        if video_elem and video_elem[0] != '0':
            has_video = 1

    # 提取创建者用户ID
    creator_id = "未找到"
    creator_id_elem = tree.xpath('//input[@name="creater_user_id"]/@value')
    if creator_id_elem:
        creator_id = creator_id_elem[0]

    # 创建结果字典
    result = {
        'name': item['name'],  # 项目名称
        'creator_id': creator_id,  # 创建者用户ID
        'go': success_status,  # 众筹成功状态 (1成功，0其他)
        'su': classify_supporters(supporters_count),  # 支持者数量分类
        'co': classify_comments(comments_count),  # 评论数分类
        'fs': classify_donations(donation_count),  # 无偿支持人数分类
        'up': classify_updates(updates_count),  # 更新数分类
        'ge': classify_tiers(tier_count),  # 档位数分类
        'ta': classify_target_amount(target_money),  # 目标金额分类
        'vi': has_video,  # 是否包含视频 (0或1)
        'pi': image_category  # 图片分类 (1-5)
    }

    creator_item = get_user_scores(creator_id)
    result.update(creator_item)

    return result


def get_page_data(page):
    url = f"https://zhongchou.modian.com/all/top_comment/all/{page}"
    response = make_request('GET', url, headers=headers)

    soup = BeautifulSoup(response.text, 'html.parser')

    # 查找包含JSON数据的script标签
    json_pattern = re.compile(r'var PROJECT_LISTS\s*=\s*JSON\.parse\(\'([^\']+)\'\);')
    json_data = None

    # 遍历所有script标签查找匹配的JSON
    for script in soup.find_all('script'):
        script_text = script.get_text(strip=True)
        match = json_pattern.search(script_text)
        if match:
            # 获取JSON字符串并处理转义字符
            json_str = match.group(1).replace('\\\'', '\'').replace('\\\\', '\\')
            try:
                # 解析JSON数据
                json_data = json.loads(json_str)
                break
            except json.JSONDecodeError as e:
                print(f"JSON解析错误: {e}")
                continue

    return json_data


def process_single_item(item: Dict) -> Dict:
    """处理单个项目的数据爬取"""
    try:
        result = extract_project_info(f'https://zhongchou.modian.com/item/{item["id"]}.html', item)
        if result:
            result['项目id'] = item['id']  # Add project ID to the result dictionary
        print(f"成功爬取项目 {item['id']}，{result}")
        return result
    except Exception as e:
        print(f"处理项目 {item['id']} 时出错: {e}")
        return None


def process_page_items(items: List[Dict], max_workers: int = 12) -> List[Dict]:
    """使用线程池处理一页中的所有项目"""
    results = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_item = {executor.submit(process_single_item, item): item for item in items}
        for future in concurrent.futures.as_completed(future_to_item):
            item = future_to_item[future]
            try:
                result = future.result()
                if result:
                    results.append(result)
            except Exception as e:
                print(f"项目 {item['id']} 处理失败: {e}")
    return results


def process_single_page(page: int) -> List[Dict]:
    """处理单个页面的数据爬取"""
    try:
        data = get_page_data(page)
        if data:
            print(f"=== 开始爬取第 {page} 页 ===")
            # 使用线程池处理当前页的所有项目
            page_results = process_page_items(data)
            # 每页完成后返回结果
            return page_results
    except Exception as e:
        print(f"获取第 {page} 页数据时出错: {e}")
    return []



def proxy_updater(interval=60):
    while True:
        fetch_proxies()
        time.sleep(interval)


# 在多线程处理之前获取代理
fetch_proxies()  # 预先填充代理池
updater_thread = threading.Thread(target=proxy_updater, args=(120,), daemon=True)
updater_thread.start()

# 使用线程池处理多页
with concurrent.futures.ThreadPoolExecutor(max_workers=1) as executor:
    future_to_page = {executor.submit(process_single_page, page): page for page in range(1, 2)}  # range(1, 1379)
    for future in concurrent.futures.as_completed(future_to_page):
        page = future_to_page[future]
        try:
            page_results = future.result()
            d.extend(page_results)
            print(f"第 {page} 页数据已处理，当前总数据量: {len(d)}")
            # 每页完成后保存一次数据
            pandas.DataFrame(d).to_excel('data.xlsx', index=False)
        except Exception as e:
            print(f"处理第 {page} 页时出错: {e}")

# 最后再保存一次以确保所有数据都已保存
pandas.DataFrame(d).to_excel('data.xlsx', index=False)
print(f"所有数据爬取完成并保存，总共爬取 {len(d)} 条数据")