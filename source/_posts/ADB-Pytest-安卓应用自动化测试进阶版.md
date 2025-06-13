---
title: ADB + Pytest 安卓应用自动化测试进阶版
date: 2025-06-12 19:46:00
tags:
---

### 项目结构

android_auto_test/  
├── pages/  
│ └── alert_dialog_page.py  
├── testcases/  
│ └── test_alert_dialog.py # pytest 测试用例  
├── utils/  
│ ├── adb_utils.py  
│ ├── log_utils.py  
│ ├── decorators.py  
│ └── wait_utils.py  
│ └── ui_utils.py  
│ └── config.py  
├── allure-results/ # pytest 运行时生成（allure serve）  
├── logs/ # pytest 运行时生成  
├── screenshots/ # pytest 运行时生成  
├── conftest.py  
└── pytest.ini

---

#### alert_dialog_page.py

```
"""
Alert Dialog 页面逻辑
"""

from utils.log_utils import logger
from utils.wait_utils import wait_for_text_appear
from utils.adb_utils import click_by_text, get_element_bounds_by_text


class AlertDialogPage:
    """
    Alert Dialog 页面逻辑
    """

    def go_to_alert_dialogs(self):
        """
        操作页面路径:：App 启动页 -> App -> Alert Dialogs
        """
        click_by_text("App")
        wait_for_text_appear("Alert Dialogs", timeout=5)
        click_by_text("Alert Dialogs")

    def click_ok_cancel_dialog(self):
        logger.debug("点击 'OK CANCEL DIALOG WITH A MESSAGE'")
        click_by_text("OK CANCEL DIALOG WITH A MESSAGE")

    def click_ok_button(self):
        logger.info("点击 OK 按钮")
        click_by_text("OK")

    def get_result_text(self):
        """
        获取 Alert Dialog 操作后的结果文本，例如 "LIST DIALOG"
        """
        nodes = get_element_bounds_by_text("LIST DIALOG")
        if not nodes:
            logger.warning("未找到结果文本")
            return ""
        return "LIST DIALOG"  # 可以进一步用 adb 截图 OCR 或使用 dump 判断文本内容

```

#### test_alert_dialog.py

```
"""
pytest 测试用例：调用页面对象和等待工具，验证 Alert Dialog 页面跳转与元素存在
"""

import allure
from utils.log_utils import logger
from utils.decorators import capture_failure
from utils.wait_utils import wait_for_text_appear
from pages.alert_dialog_page import AlertDialogPage


@allure.feature("Alert Dialog 测试")
@allure.story("跳转并验证页面文本")
@capture_failure()
def test_alert_dialog_navigation():
    """
    测试：启动页跳转到 Alert Dialog 页面，目标文本出现
    """
    page = AlertDialogPage()
    page.go_to_alert_dialogs()
    assert wait_for_text_appear("OK CANCEL DIALOG WITH A MESSAGE", timeout=5)
    logger.info("测试通过：页面跳转成功，目标文本出现")


@allure.feature("Alert Dialog 测试")
@allure.story("点击 OK 并验证结果")
@capture_failure()
def test_alert_dialog_click_ok():
    """
        测试：启动页跳转到 Alert Dialog 页面，目标文本点击交互
    """
    page = AlertDialogPage()
    page.go_to_alert_dialogs()
    page.click_ok_cancel_dialog()
    page.click_ok_button()
    assert wait_for_text_appear("LIST DIALOG", timeout=5)
    logger.info("测试通过：点击 OK 后正确")

```

#### adb_utils.py

```
"""
ADB 工具：封装包括启动 App、点击文本等
"""

import re
import time
import subprocess

from utils.log_utils import logger
from utils.ui_utils import parse_ui_xml_nodes
# from utils.wait_utils import wait_for_text_appear


# # 标志首次启动
# is_first_launch = True


def adb_shell(cmd: str) -> str:
    """
    执行 adb shell 命令并返回结果
    """
    full_cmd = f"adb shell {cmd}"
    result = subprocess.run(full_cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, encoding="utf-8")
    output = result.stdout.strip()
    logger.debug(f"[ADB SHELL] {cmd} -> {output[:100]}")  # 最多显示前 100 字符
    return output


def start_app(pkg: str, activity: str):
    """
    启动 App（包名 + 启动 Activity）
    """
    logger.info(f"启动 App: {pkg}/{activity}")
    adb_shell(f"am start -n {pkg}/{activity}")
    time.sleep(1)


def restart_app(pkg: str, activity: str):
    """
    使用 -S 参数实现近似冷启动，避免模拟器因 force-stop 崩溃
    """
    logger.info(f"使用 -S 参数重启 App: {pkg}/{activity}")
    adb_shell(f"am start -S -n {pkg}/{activity}")
    time.sleep(1)


def stop_app(pkg: str):
    """
    关闭 App（包名）
    """
    logger.info(f"关闭 App: {pkg}")
    adb_shell(f"am force-stop {pkg}")


# def press_back():
#     """
#     模拟点击安卓返回键
#     """
#     logger.info("执行返回操作")
#     adb_shell("input keyevent 4")


# def back_to_home(max_retry=5):
#     """
#     返回首页，直到检测到首页元素，如 “App”
#     """
#     for i in range(max_retry):
#         if wait_for_text_appear("App", timeout=2):
#             logger.info(f"[返回首页] 成功（第{i+1}次）")
#             return True
#         logger.debug(f"[返回首页] 尝试第{i+1}次返回")
#         press_back()
#         time.sleep(1)
#     raise RuntimeError("多次返回后仍未回到首页")


# def restart_to_home(pkg: str, activity: str):
#     """
#     首次启动或热重启 App 并返回首页
#     """
#     global is_first_launch
#     if is_first_launch:
#         logger.info("首次启动 App 并进入首页")
#         is_first_launch = False
#     else:
#         logger.info("热重启 App 并返回首页")
#
#     start_app(pkg, activity)
#     back_to_home()


def click_at(x: int, y: int):
    """
    通过坐标点击屏幕
    """
    logger.debug(f"点击坐标: ({x}, {y})")
    adb_shell(f"input tap {x} {y}")


def get_element_bounds_by_text(target_text: str):
    """
    查找包含指定文本的 UI 元素，解析 bounds 获取中点坐标
    """
    for node in parse_ui_xml_nodes():
        if node.attrib.get("text") == target_text:
            bounds = node.attrib.get("bounds")
            match = re.findall(r"\[(\d+),(\d+)]", bounds)
            if match and len(match) == 2:
                x1, y1 = map(int, match[0])
                x2, y2 = map(int, match[1])
                return int((x1 + x2) / 2), int((y1 + y2) / 2)
    return None


def click_by_text(text: str):
    """
    通过文本查找并点击
    """
    coords = get_element_bounds_by_text(text)
    if coords:
        logger.info(f"点击文本: {text}")
        click_at(*coords)
    else:
        logger.error(f"未找到文本: {text}")
        raise ValueError(f"找不到文本: {text}")


def pull_file(remote: str, local: str):
    """
    从设备拉取文件到本地
    """
    logger.info(f"拉取文件: {remote} -> {local}")
    try:
        result = subprocess.run(["adb", "pull", remote, local],
                                capture_output=True, text=True, timeout=10)
        if result.returncode == 0:
            logger.info(f"拉取成功: {result.stdout.strip()}")
        else:
            logger.error(f"拉取失败: {result.stderr.strip()}")
    except Exception as e:
        logger.error(f"拉取异常: {e}")

```

#### log_utils.py

```
"""
日志工具：统一日志格式与级别
"""

import os
import logging
from datetime import datetime
from utils.config import DEBUG_MODE
from rich.logging import RichHandler


def init_logger(name=__name__):
    """
    初始化日志器，支持控制台彩色和文件日志输出
    """
    logger = logging.getLogger(name)
    if not logger.handlers:
        logger.setLevel(logging.DEBUG if DEBUG_MODE else logging.INFO)

        # 日志格式设置
        # formatter = logging.Formatter("[%(asctime)s] [%(levelname)s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S")

        # 控制台日志（Rich 彩色）
        ch = RichHandler(markup=True, show_time=True, show_level=True, show_path=False)
        cf = logging.Formatter("%(message)s")
        ch.setFormatter(cf)
        logger.addHandler(ch)

        # 文件日志（完整信息）
        os.makedirs("logs", exist_ok=True)
        log_file = f"logs/{datetime.now():%Y%m%d_%H%M%S}.log"
        fh = logging.FileHandler(log_file, encoding="utf-8")
        fh.setFormatter(logging.Formatter(
            "[%(asctime)s] [%(levelname)s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S"
        ))
        logger.addHandler(fh)

    return logger


# 全局日志对象
logger = init_logger()

```

#### decorators.py

```
import os
import time
import functools
from utils.log_utils import logger
from utils.adb_utils import adb_shell, pull_file


def capture_failure(capture_success=False):
    """
    装饰器：用于测试用例，失败时自动截图并保存，也支持成功截图
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            case_name = func.__name__   # 获取函数名作为用例名
            timestamp = time.strftime("%Y%m%d_%H%M%S")  # 当前时间戳
            file_prefix = f"{case_name}_{timestamp}"    # 构建文件名前缀
            remote_path = f"/sdcard/{file_prefix}.png"  # 设备中的截图保存路径
            local_dir = "screenshots"   # 本地保存路径
            local_path = os.path.join(local_dir, f"{file_prefix}.png")  # 本地保存完整路径

            try:
                result = func(*args, **kwargs)  # 执行原始测试函数

                if capture_success:
                    # 成功截图（可选）
                    adb_shell(f"screencap -p {remote_path}")
                    logger.info(f"测试通过，截图保存至设备: {remote_path}")
                    if not os.path.exists(local_dir):
                        os.makedirs(local_dir)
                    pull_file(remote_path, local_path)
                    logger.info(f"成功截图已拉取到本地: {local_path}")

                return result

            except Exception as e:
                # 捕获异常并截图
                logger.error(f"测试失败：{case_name} - {e}", exc_info=True)
                # 截图保存到设备
                adb_shell(f"screencap -p {remote_path}")
                logger.info(f"已截图保存至设备: {remote_path}")
                # 确保本地目录存在
                if not os.path.exists(local_dir):
                    os.makedirs(local_dir)
                # 拉取截图到本地
                pull_file(remote_path, local_path)
                logger.info(f"已拉取截图到本地: {local_path}")

                raise  # 继续抛出异常供 pytest 标红
        return wrapper
    return decorator

```

#### wait_utils.py

```
"""
等待工具：封装动态轮询等待某文本出现
"""

import time
from utils.log_utils import logger
from utils.ui_utils import parse_ui_xml_nodes


def get_ui_texts():
    """
    获取当前 UI 所有可见文本内容（解析 uiautomator xml）
    """
    return [
        node.attrib.get("text")
        for node in parse_ui_xml_nodes()
        if node.attrib.get("text")
    ]


def wait_for_text_appear(target_text: str, timeout=10, interval=1):
    """
    等待某个文本出现在 UI 中
    """
    logger.debug(f"等待文本出现: {target_text}")
    start_time = time.time()
    while time.time() - start_time < timeout:
        texts = get_ui_texts()

        if target_text in texts:
            logger.debug(f"文本已出现: {target_text}")
            return True

        time.sleep(interval)

    logger.warning(f"等待超时，未找到文本: {target_text}")
    return False

```

#### ui_utils.py

```
"""
UI 解析工具：封装从设备获取并解析 UI XML 节点方法
"""

import subprocess
from utils.log_utils import logger
import xml.etree.ElementTree as ET


def adb_shell(cmd: str) -> str:
    result = subprocess.run(f"adb shell {cmd}", shell=True,
                            stdout=subprocess.PIPE, stderr=subprocess.PIPE, encoding="utf-8")
    return result.stdout.strip()


def parse_ui_xml_nodes():
    """
    获取并解析当前 UI 层级所有节点
    """
    adb_shell("uiautomator dump /sdcard/ui_dump.xml")
    xml_content = adb_shell("cat /sdcard/ui_dump.xml")
    try:
        tree = ET.fromstring(xml_content)
        return list(tree.iter("node"))
    except Exception as e:
        logger.error(f"解析 XML 失败: {e}")
        return []

```

#### conftest.py

```
import pytest

from utils.adb_utils import restart_app
from utils.wait_utils import wait_for_text_appear

PACKAGE_NAME = "io.appium.android.apis"
LAUNCH_ACTIVITY = ".ApiDemos"


@pytest.fixture(scope="function", autouse=True)
def setup_app():
    """
    每个用例执行前通过 am start -S 重启 App（替代冷启动）
    """
    restart_app(PACKAGE_NAME, LAUNCH_ACTIVITY)

    # 等待首页稳定加载，比如“App”文本出现
    assert wait_for_text_appear("App", timeout=10), "首页未加载完成"

    yield

```

#### pytest.ini

```
[pytest]
# addopts 是 pytest 的“额外命令行参数”，相当于你每次运行 pytest 都自动带上的参数
addopts = -v -s --alluredir=allure-results

# 这里告诉 pytest 只去 testcases 目录下查找测试文件和测试用例，而不是默认搜索当前目录或递归全部
testpaths = testcases

```
