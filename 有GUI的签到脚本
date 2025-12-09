import sys
import os
import configparser
import threading
from datetime import datetime
from pathlib import Path

import pyautogui
from apscheduler.schedulers.background import BackgroundScheduler
from PyQt6.QtCore import (Qt, QThread, pyqtSignal, QTimer, QObject)
from PyQt6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout, QGridLayout,
    QGroupBox, QLabel, QLineEdit, QPushButton, QTextEdit, QMessageBox,
    QSpinBox, QFrame, QSizePolicy
)
from PyQt6.QtGui import QFont, QPalette, QColor, QIcon

# -------------------------- 全局配置 --------------------------
CONFIG_PATH = Path("qiandao_config.ini")
LOG_FILE = "qiandao_log.txt"
SCREENSHOT_DIR = Path("screenshots")
SCREENSHOT_DIR.mkdir(exist_ok=True)

# -------------------------- 日志重定向 --------------------------
import logging


class QtLogHandler(logging.Handler, QObject):
    log_signal = pyqtSignal(str)

    def __init__(self):
        logging.Handler.__init__(self)
        QObject.__init__(self)

    def emit(self, record):
        msg = self.format(record)
        self.log_signal.emit(msg)


def setup_logging(log_widget):
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    logger.handlers.clear()

    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)

    file_handler = logging.FileHandler(LOG_FILE, encoding="utf-8")
    file_handler.setLevel(logging.INFO)

    qt_handler = QtLogHandler()
    qt_handler.log_signal.connect(log_widget.append_log)
    qt_handler.setLevel(logging.INFO)

    formatter = logging.Formatter(
        "%(asctime)s - %(levelname)s - %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )
    for handler in [console_handler, file_handler, qt_handler]:
        handler.setFormatter(formatter)
        logger.addHandler(handler)

    return logger


# -------------------------- 核心优化：智能签到线程 --------------------------
class SignInThread(QThread):
    finish_signal = pyqtSignal(bool, str)
    log_signal = pyqtSignal(str)

    # 配置参数（可根据实际情况调整）
    RETRY_TIMES = 3  # 每个步骤重试次数
    ELEMENT_TIMEOUT = 20  # 元素等待超时（秒）
    SHORT_WAIT = 0.5  # 短等待（秒）
    LONG_WAIT = 2  # 长等待（秒）

    def __init__(self):
        super().__init__()
        # 目标坐标配置（可根据实际情况修改）
        self.coords = {
            "browser_icon": (508, 1055),
            "bookmark": (1385, 116),
            "game_subtag": (1446, 168),
            "game_link": (1131, 340),
            "sign_tag": (1870, 713),
            "sign_btn": (1740, 302)
        }

    def wait_for_pixel_color(self, pos, target_color=None, timeout=10):
        """
        智能等待：检测目标位置像素颜色（可选）
        pos: 坐标(x,y)
        target_color: 目标RGB颜色（如(255,0,0)），None则仅等待窗口激活
        timeout: 超时时间
        """
        start_time = datetime.now()
        while (datetime.now() - start_time).total_seconds() < timeout:
            # 检查鼠标位置是否在前台窗口
            try:
                pyautogui.moveTo(pos, duration=0.2)
                if target_color is None:
                    return True
                # 检测像素颜色
                current_color = pyautogui.pixel(*pos)
                if current_color == target_color:
                    return True
            except Exception:
                pass
            pyautogui.sleep(self.SHORT_WAIT)
        self.log_signal.emit(f"⚠️ 等待坐标{pos}超时（{timeout}秒）")
        return False

    def click_with_retry(self, pos, desc, target_color=None):
        """
        带重试的点击操作
        pos: 坐标(x,y)
        desc: 操作描述（日志用）
        target_color: 点击后等待的目标颜色
        """
        for retry in range(self.RETRY_TIMES):
            try:
                self.log_signal.emit(f"🔍 尝试{retry + 1}/{self.RETRY_TIMES}：{desc}")
                # 移动鼠标到目标位置（平滑移动）
                pyautogui.moveTo(pos, duration=0.3)
                # 点击
                pyautogui.click(pos)
                self.log_signal.emit(f"✅ 点击{desc}成功")

                # 等待元素加载/颜色匹配
                if target_color:
                    if self.wait_for_pixel_color(pos, target_color, self.ELEMENT_TIMEOUT):
                        return True
                else:
                    pyautogui.sleep(self.LONG_WAIT)
                    return True
            except Exception as e:
                self.log_signal.emit(f"❌ 点击{desc}失败：{str(e)}")
                pyautogui.sleep(self.LONG_WAIT)
        self.log_signal.emit(f"❌ {desc}重试{self.RETRY_TIMES}次仍失败")
        return False

    def activate_browser_window(self):
        """激活浏览器窗口（确保在前台）"""
        self.log_signal.emit("📌 激活浏览器窗口...")
        # 先点击浏览器图标
        if self.click_with_retry(self.coords["browser_icon"], "浏览器图标"):
            # 等待浏览器窗口加载
            pyautogui.sleep(3)
            # 再次点击确保窗口前置
            pyautogui.click(self.coords["browser_icon"])
            return True
        return False

    def run(self):
        try:
            self.log_signal.emit("🚀 开始智能签到流程...")
            pyautogui.FAILSAFE = True
            pyautogui.PAUSE = self.SHORT_WAIT
            screen_width, screen_height = pyautogui.size()
            self.log_signal.emit(f"🖥️ 屏幕分辨率：{screen_width} x {screen_height}")

            # 步骤1：激活浏览器窗口
            if not self.activate_browser_window():
                raise Exception("浏览器窗口激活失败")

            # 步骤2：打开游戏页面（带重试）
            self.log_signal.emit("📖 打开游戏书签...")
            if not self.click_with_retry(self.coords["bookmark"], "书签栏"):
                raise Exception("点击书签栏失败")
            if not self.click_with_retry(self.coords["game_subtag"], "游戏子标签"):
                raise Exception("点击游戏子标签失败")
            if not self.click_with_retry(self.coords["game_link"], "游戏链接"):
                raise Exception("点击游戏链接失败")

            # 步骤3：二次验证（确保页面打开）
            self.log_signal.emit("🔄 二次验证游戏页面...")
            pyautogui.sleep(self.LONG_WAIT)
            if not self.click_with_retry(self.coords["bookmark"], "书签栏（二次）"):
                raise Exception("二次点击书签栏失败")
            if not self.click_with_retry(self.coords["game_subtag"], "游戏子标签（二次）"):
                raise Exception("二次点击游戏子标签失败")
            if not self.click_with_retry(self.coords["game_link"], "游戏链接（二次）"):
                raise Exception("二次点击游戏链接失败")

            # 步骤4：等待页面加载（智能超时）
            self.log_signal.emit("⏳ 等待游戏页面加载（最大20秒）...")
            if not self.wait_for_pixel_color(self.coords["game_link"], None, 20):
                raise Exception("游戏页面加载超时")

            # 步骤5：点击签到标签
            self.log_signal.emit("📝 点击签到标签...")
            if not self.click_with_retry(self.coords["sign_tag"], "签到标签"):
                raise Exception("点击签到标签失败")

            # 步骤6：等待签到面板加载
            self.log_signal.emit("⏳ 等待签到面板加载（最大15秒）...")
            if not self.wait_for_pixel_color(self.coords["sign_btn"], None, 15):
                raise Exception("签到面板加载超时")

            # 步骤7：点击签到按钮
            self.log_signal.emit("🎯 点击签到按钮...")
            if not self.click_with_retry(self.coords["sign_btn"], "签到按钮"):
                raise Exception("点击签到按钮失败")

            # 步骤8：验证签到结果
            pyautogui.sleep(3)
            screenshot_name = f"qiandao_{datetime.now().strftime('%Y%m%d_%H%M%S')}.png"
            screenshot_path = SCREENSHOT_DIR / screenshot_name
            pyautogui.screenshot(str(screenshot_path))

            msg = f"🎉 签到成功！截图已保存至：{screenshot_path}"
            self.log_signal.emit(msg)
            self.finish_signal.emit(True, msg)

        except Exception as e:
            error_msg = f"❌ 签到失败：{str(e)}"
            self.log_signal.emit(error_msg)
            self.finish_signal.emit(False, error_msg)


# -------------------------- 主界面类（无修改） --------------------------
class SignInGUI(QMainWindow):
    def __init__(self):
        super().__init__()
        self.scheduler = None
        self.sign_in_thread = None
        self.last_sign_in_time = None
        self.config = self.load_config()

        self.setWindowTitle("自动签到助手 v2.0（优化版）")
        self.setMinimumSize(800, 600)
        self.setStyleSheet(self.get_style_sheet())

        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        main_layout = QVBoxLayout(central_widget)
        main_layout.setSpacing(20)
        main_layout.setContentsMargins(30, 30, 30, 30)

        # 1. 标题栏
        title_layout = QHBoxLayout()
        title_label = QLabel("🎯 自动签到助手（智能优化版）")
        title_label.setFont(QFont("微软雅黑", 24, QFont.Weight.Bold))
        title_label.setAlignment(Qt.AlignmentFlag.AlignCenter)
        title_layout.addWidget(title_label)
        main_layout.addLayout(title_layout)

        # 2. 功能区
        func_group = QGroupBox("核心功能")
        func_group.setFont(QFont("微软雅黑", 14))
        func_layout = QGridLayout(func_group)
        func_layout.setSpacing(15)
        func_layout.setContentsMargins(20, 20, 20, 20)

        # 定时时间设置
        func_layout.addWidget(QLabel("定时签到时间："), 0, 0, Qt.AlignmentFlag.AlignRight)
        self.hour_spin = QSpinBox()
        self.hour_spin.setRange(0, 23)
        self.hour_spin.setValue(int(self.config.get("hour", 8)))
        self.hour_spin.setFixedWidth(80)
        func_layout.addWidget(self.hour_spin, 0, 1)

        func_layout.addWidget(QLabel("时"), 0, 2)

        self.minute_spin = QSpinBox()
        self.minute_spin.setRange(0, 59)
        self.minute_spin.setValue(int(self.config.get("minute", 30)))
        self.minute_spin.setFixedWidth(80)
        func_layout.addWidget(self.minute_spin, 0, 3)

        func_layout.addWidget(QLabel("分"), 0, 4)

        # 保存配置按钮
        save_btn = QPushButton("💾 保存时间设置")
        save_btn.clicked.connect(self.save_config)
        func_layout.addWidget(save_btn, 0, 5)

        # 手动签到按钮
        manual_btn = QPushButton("🚀 立即手动签到")
        manual_btn.clicked.connect(self.start_manual_sign_in)
        func_layout.addWidget(manual_btn, 1, 0, 1, 3)

        # 启动/停止定时任务按钮
        self.timer_btn = QPushButton("▶️ 启动定时签到")
        self.timer_btn.clicked.connect(self.toggle_scheduler)
        func_layout.addWidget(self.timer_btn, 1, 3, 1, 3)

        # 状态显示
        func_layout.addWidget(QLabel("当前状态："), 2, 0, Qt.AlignmentFlag.AlignRight)
        self.status_label = QLabel("未启动")
        self.status_label.setStyleSheet("color: #ff6b6b; font-weight: bold;")
        func_layout.addWidget(self.status_label, 2, 1, 1, 2)

        # 最后签到时间
        func_layout.addWidget(QLabel("最后签到："), 2, 3, Qt.AlignmentFlag.AlignRight)
        self.last_sign_label = QLabel("从未执行")
        func_layout.addWidget(self.last_sign_label, 2, 4, 1, 2)

        main_layout.addWidget(func_group)

        # 3. 日志区
        log_group = QGroupBox("运行日志")
        log_group.setFont(QFont("微软雅黑", 14))
        log_layout = QVBoxLayout(log_group)
        log_layout.setContentsMargins(20, 20, 20, 20)

        self.log_text = QTextEdit()
        self.log_text.setReadOnly(True)
        self.log_text.setFont(QFont("Consolas", 10))
        log_layout.addWidget(self.log_text)

        # 清空日志按钮
        clear_log_btn = QPushButton("🗑️ 清空日志")
        clear_log_btn.clicked.connect(lambda: self.log_text.clear())
        log_layout.addWidget(clear_log_btn, alignment=Qt.AlignmentFlag.AlignRight)

        main_layout.addWidget(log_group, stretch=1)

        # 4. 底部信息
        footer_layout = QHBoxLayout()
        footer_label = QLabel("© 2025 自动签到助手（智能优化版）- 请勿用于非法用途")
        footer_label.setAlignment(Qt.AlignmentFlag.AlignCenter)
        footer_label.setStyleSheet("color: #888888; font-size: 12px;")
        footer_layout.addWidget(footer_label)
        main_layout.addLayout(footer_layout)

        # 初始化日志
        self.logger = setup_logging(self)
        self.logger.info("程序启动成功（智能优化版）")

        # 定时更新状态
        self.status_timer = QTimer()
        self.status_timer.timeout.connect(self.update_status)
        self.status_timer.start(1000)

    def append_log(self, msg):
        self.log_text.append(msg)
        cursor = self.log_text.textCursor()
        cursor.movePosition(cursor.MoveOperation.End)
        self.log_text.setTextCursor(cursor)

    def get_style_sheet(self):
        return """
        QWidget {
            background-color: #f8f9fa;
            color: #333333;
            font-family: "微软雅黑";
        }
        QGroupBox {
            border: 2px solid #e9ecef;
            border-radius: 10px;
            padding-top: 15px;
            margin-top: 8px;
            font-weight: bold;
        }
        QGroupBox::title {
            subcontrol-origin: margin;
            left: 10px;
            padding: 0 10px 0 10px;
            color: #495057;
        }
        QPushButton {
            background-color: #007bff;
            color: white;
            border: none;
            border-radius: 8px;
            padding: 10px 20px;
            font-size: 14px;
            font-weight: bold;
            min-height: 40px;
        }
        QPushButton:hover {
            background-color: #0056b3;
            border: 1px solid #004085;
        }
        QPushButton:pressed {
            background-color: #004085;
            padding-left: 11px;
            padding-top: 11px;
        }
        QSpinBox, QLineEdit {
            border: 2px solid #e9ecef;
            border-radius: 8px;
            padding: 8px 10px;
            font-size: 14px;
            background-color: white;
        }
        QSpinBox:focus, QLineEdit:focus {
            border-color: #007bff;
            outline: none;
        }
        QTextEdit {
            border: 2px solid #e9ecef;
            border-radius: 10px;
            padding: 10px;
            background-color: white;
            selection-background-color: #007bff;
            selection-color: white;
        }
        QLabel {
            font-size: 14px;
        }
        QMainWindow {
            border: 1px solid #dee2e6;
            border-radius: 15px;
        }
        """

    def load_config(self):
        config = configparser.ConfigParser()
        if CONFIG_PATH.exists():
            config.read(CONFIG_PATH, encoding="utf-8")
        if "SIGNIN" not in config:
            config["SIGNIN"] = {"hour": 8, "minute": 30}
        return config["SIGNIN"]

    def save_config(self):
        self.config["hour"] = str(self.hour_spin.value())
        self.config["minute"] = str(self.minute_spin.value())

        with open(CONFIG_PATH, "w", encoding="utf-8") as f:
            config = configparser.ConfigParser()
            config["SIGNIN"] = self.config
            config.write(f)

        self.logger.info(f"配置已保存：定时签到时间 {self.hour_spin.value()}:{self.minute_spin.value():02d}")
        QMessageBox.information(self, "成功", "时间设置已保存！")

    def start_manual_sign_in(self):
        if self.sign_in_thread and self.sign_in_thread.isRunning():
            QMessageBox.warning(self, "提示", "签到操作正在进行中，请稍候...")
            return

        self.sign_in_thread = SignInThread()
        self.sign_in_thread.log_signal.connect(self.append_log)
        self.sign_in_thread.finish_signal.connect(self.on_sign_in_finish)
        self.sign_in_thread.start()

    def on_sign_in_finish(self, success, msg):
        self.last_sign_in_time = datetime.now()
        self.last_sign_label.setText(self.last_sign_in_time.strftime("%Y-%m-%d %H:%M:%S"))

        if success:
            QMessageBox.information(self, "成功", msg)
        else:
            QMessageBox.critical(self, "失败", msg)

    def toggle_scheduler(self):
        if self.scheduler and self.scheduler.running:
            self.scheduler.shutdown()
            self.scheduler = None
            self.timer_btn.setText("▶️ 启动定时签到")
            self.timer_btn.setStyleSheet("")
            self.status_label.setText("已停止")
            self.status_label.setStyleSheet("color: #ff6b6b; font-weight: bold;")
            self.logger.info("定时签到任务已停止")
        else:
            hour = self.hour_spin.value()
            minute = self.minute_spin.value()

            self.scheduler = BackgroundScheduler(timezone="Asia/Shanghai")
            self.scheduler.add_job(
                func=self.start_manual_sign_in,
                trigger="cron",
                hour=hour,
                minute=minute,
                id="daily_sign_in",
                replace_existing=True,
                misfire_grace_time=300
            )

            self.scheduler.start()
            self.timer_btn.setText("⏹️ 停止定时签到")
            self.timer_btn.setStyleSheet(
                "QPushButton { background-color: #dc3545; } QPushButton:hover { background-color: #c82333; }")
            self.status_label.setText(f"运行中（每日{hour:02d}:{minute:02d}执行）")
            self.status_label.setStyleSheet("color: #28a745; font-weight: bold;")

            self.logger.info(f"定时签到任务已启动，每日{hour:02d}:{minute:02d}自动执行")
            QMessageBox.information(self, "成功", f"定时签到已启动！每日{hour:02d}:{minute:02d}自动执行")

    def update_status(self):
        if self.scheduler and self.scheduler.running:
            self.status_label.setText(f"运行中（每日{self.hour_spin.value():02d}:{self.minute_spin.value():02d}执行）")
        else:
            self.status_label.setText("已停止")

    def closeEvent(self, event):
        if self.scheduler and self.scheduler.running:
            self.scheduler.shutdown()
        if self.sign_in_thread and self.sign_in_thread.isRunning():
            self.sign_in_thread.terminate()
        self.logger.info("程序已退出")
        event.accept()


# -------------------------- 主函数 --------------------------
if __name__ == "__main__":
    pyautogui.FAILSAFE = True
    pyautogui.PAUSE = 0.5

    app = QApplication(sys.argv)
    app.setStyle("Fusion")

    window = SignInGUI()
    window.show()

    sys.exit(app.exec())
