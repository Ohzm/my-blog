---
title: ADB + Pytest 安卓应用自动化测试进阶版
date: 2025-06-12 19:46:00
tags:
---

### 项目结构

android_auto_test/  
├── pages/  
│ ├── **init**.py  
│ └── alert_dialog_page.py  
├── testcases/  
│ ├── **init**.py  
│ └── test_alert_dialog.py # pytest 测试用例  
├── utils/  
│ ├── **init**.py  
│ ├── adb_utils.py  
│ ├── log_utils.py  
│ ├── decorators.py  
│ └── wait_utils.py  
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
from utils.adb_utils import adb_click_text
from utils.wait_utils import wait_for_text_appear


class AlertDialogPage:
    """
    对应页面路径: App 启动页 -> App -> Alert Dialogs
    """

    def go_to_alert_dialogs(self):
        """
        执行跳转到 Alert Dialogs 的操作
        """
        logger.info("跳转到 Alert Dialogs 页面")
        wait_for_text_appear("App", timeout=5)
        adb_click_text("App")
        adb_click_text("Alert Dialogs")

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
    测试：启动页跳转到 Alert Dialog 页面，断言目标文本
    """
    page = AlertDialogPage()
    page.go_to_alert_dialogs()
    assert wait_for_text_appear("OK CANCEL DIALOG WITH A MESSAGE", timeout=5)
    logger.info("测试通过：页面跳转成功，目标文本出现")

```

#### adb_utils.py

```
"""
ADB 封装方法，包括启动 App、点击文本等
"""

import re
import subprocess
from utils.log_utils import logger
import xml.etree.ElementTree as ET


# 是否开启调试日志（建议放配置项中统一控制）
debug_mode = False


def log_info(msg):
    logger.info(f"{msg}")


def log_debug(msg):
    if debug_mode:
        logger.debug(f"[ADB] {msg}")


def adb_shell(cmd: str) -> str:
    """
    执行 adb shell 命令并返回输出结果
    """
    full_cmd = f"adb shell {cmd}"
    result = subprocess.run(full_cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, encoding="utf-8")
    output = result.stdout.strip()

    if "cat /sdcard/ui_dump.xml" in cmd:
        log_debug(f"执行命令: {cmd}，返回: XML 内容省略")
    elif "uiautomator dump" in cmd:
        log_debug(f"执行命令: {cmd} -> UI hierarchy dumped")
    else:
        log_debug(f"执行命令: {cmd} -> {output}")

    return output


def adb_start_app(package_name: str, activity_name: str):
    """
    启动指定 App（包名 + 启动 Activity）
    """
    log_info(f"启动 App: {package_name}/{activity_name}")
    adb_shell(f"am start -n {package_name}/{activity_name}")


def parse_ui_xml_nodes():
    """
    获取并解析当前 UI 层级所有节点
    :return: 所有 <node> 节点列表
    """
    adb_shell("uiautomator dump /sdcard/ui_dump.xml")
    xml = adb_shell("cat /sdcard/ui_dump.xml")
    try:
        tree = ET.fromstring(xml)
        return list(tree.iter("node"))
    except Exception as e:
        logger.error(f"解析 XML 失败: {e}")
        return []


def adb_click(x: int, y: int):
    """
    通过坐标点击屏幕
    """
    log_debug(f"点击坐标: ({x}, {y})")
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


def adb_click_text(text: str):
    """
    通过文本查找元素并点击
    """
    log_info(f"点击文本: {text}")
    coords = get_element_bounds_by_text(text)
    if coords:
        adb_click(*coords)
    else:
        logger.error(f"未找到指定文本: {text}")
        raise ValueError(f"未找到指定文本: {text}")


def adb_pull(remote_path: str, local_path: str):
    """
    从设备拉取文件到本地
    """
    log_info(f"拉取文件: {remote_path} 到本地: {local_path}")
    try:
        result = subprocess.run(["adb", "pull", remote_path, local_path],
                                capture_output=True, text=True, timeout=10)
        if result.returncode == 0:
            log_info(f"拉取成功: {result.stdout.strip()}")
        else:
            logger.error(f"拉取失败: {result.stderr.strip()}")
    except Exception as e:
        logger.error(f"拉取过程中出现异常: {e}")

```

#### log_utils.py

```
"""
日志工具 统一日志格式与级别
"""

import os
import sys
import logging
from datetime import datetime
from colorama import init, Fore, Style

# 初始化 colorama
init(autoreset=True)

# 通过环境变量控制是否启用 debug
DEBUG_MODE = "--debug" in sys.argv


class ColoredFormatter(logging.Formatter):
    """
    控制台彩色日志格式
    """
    def format(self, record):
        level_color = {
            logging.DEBUG: Fore.CYAN,
            logging.INFO: Fore.GREEN,
            logging.WARNING: Fore.YELLOW,
            logging.ERROR: Fore.RED,
            logging.CRITICAL: Fore.MAGENTA
        }.get(record.levelno, Fore.WHITE)

        time_str = self.formatTime(record, self.datefmt)
        message = f"[{time_str}] {record.getMessage()}"
        return f"{level_color}{message}{Style.RESET_ALL}"


def init_logger(name=__name__):
    """
    初始化日志记录器，支持控制台和文件日志输出
    """
    logger = logging.getLogger(name)
    if not logger.handlers:
        logger.setLevel(logging.DEBUG if DEBUG_MODE else logging.INFO)

        # 日志格式设置
        # formatter = logging.Formatter("[%(asctime)s] [%(levelname)s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S")

        # 控制台输出：带颜色 简洁
        ch = logging.StreamHandler()
        ch.setFormatter(ColoredFormatter(datefmt="%Y-%m-%d %H:%M:%S"))
        logger.addHandler(ch)

        # 文件输出：完整格式
        os.makedirs("logs", exist_ok=True)
        filename = f"logs/{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"
        fh = logging.FileHandler(filename, encoding="utf-8")
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
from utils.adb_utils import adb_shell, adb_pull


def capture_failure(capture_success=False):
    """
    装饰器：用于测试用例，失败时自动截图并保存，必要时也支持成功截图
    :param capture_success: 是否在测试通过时也截图（默认 False）
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            case_name = func.__name__
            timestamp = time.strftime("%Y%m%d_%H%M%S")
            file_prefix = f"{case_name}_{timestamp}"
            remote_path = f"/sdcard/{file_prefix}.png"
            local_dir = "screenshots"
            local_path = os.path.join(local_dir, f"{file_prefix}.png")

            try:
                result = func(*args, **kwargs)

                if capture_success:
                    # 成功截图（可选）
                    adb_shell(f"screencap -p {remote_path}")
                    logger.info(f"测试通过，截图保存至设备: {remote_path}")
                    if not os.path.exists(local_dir):
                        os.makedirs(local_dir)
                    adb_pull(remote_path, local_path)
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
                adb_pull(remote_path, local_path)
                logger.info(f"已拉取截图到本地: {local_path}")

                raise  # 继续抛出异常供 pytest 标红
        return wrapper
    return decorator

```

#### wait_utils.py

```
"""
等待工具类 封装动态轮询等待某文本出现
"""

import time
from utils.log_utils import logger
from utils.adb_utils import parse_ui_xml_nodes


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
    logger.info(f"等待文本出现: {target_text}")
    start_time = time.time()
    while time.time() - start_time < timeout:
        texts = get_ui_texts()
        if target_text in texts:
            logger.info(f"文本已出现: {target_text}")
            return True
        time.sleep(interval)
    logger.warning(f"等待超时，未找到文本: {target_text}")
    return False

```

#### conftest.py

```
import pytest
from utils.adb_utils import adb_start_app


@pytest.fixture(scope="function", autouse=True)
def setup_app():
    """
    每个测试用例执行前自动启动 App
    """
    adb_start_app("io.appium.android.apis", ".ApiDemos")

```

#### pytest.ini

```
[pytest]
addopts = -v -s --alluredir=allure-results
testpaths = testcases

```
